Reply to review of [PATCH v10 07/10] drm/xe/pm: Introduce xe_device_suspend/resume()

=== 1. "Can this cause a deadlock during the PCIe FLR prepare sequence?" ===

Yes, but not for the reason given. The TDR *is* queued, and the fence still
may never signal.

The premise "TDR wasn't queued" is not correct. xe_guc_flr_prepare() calls
two things:

	xe_guc_submit_stop(guc);
	xe_guc_submit_pause_abort(guc);

xe_guc_submit_stop() -> guc_exec_queue_stop() is the one that skips banning
kernel queues. But xe_guc_submit_pause_abort() then walks every queue in
guc->submission_state.exec_queue_lookup with no kernel exemption and calls
guc_exec_queue_kill(), which does set_exec_queue_killed() plus
xe_guc_exec_queue_trigger_cleanup() -> xe_sched_tdr_queue_imm(). The migrate
queue's TDR is queued.

Ordering is also guaranteed. The TDR's timeout_wq is guc_to_gt(guc)->ordered_wq
(xe_guc_submit.c, xe_sched_init() call in guc_exec_queue_init()), which is a
drmm_alloc_ordered_workqueue(). xe_uc_flr_prepare() queues uc_flr_sanitize onto
that same ordered wq after uc_flr_prepare() has returned and flushes it, so
every TDR queued during step 1 has run by the time xe_uc_flr_prepare() returns.
That is what the comment in that function is describing.

Interrupts being disabled is also not the issue. guc_exec_queue_stop() does

	atomic_and(EXEC_QUEUE_STATE_WEDGED | EXEC_QUEUE_STATE_BANNED |
		   EXEC_QUEUE_STATE_KILLED | EXEC_QUEUE_STATE_DESTROYED |
		   EXEC_QUEUE_STATE_SUSPENDED, &q->guc->state);

which clears ENABLED and PENDING_DISABLE, so the "Kick job / queue off
hardware" block in guc_exec_queue_timedout_job() is skipped entirely. No G2H
round trip, no dependency on interrupts.

The actual problem is further down the same TDR. For a kernel queue with an
in-flight job:

  - wedged stays false, because exec_queue_killed(q) is true so
    guc_submit_hint_wedged() is skipped
  - timeout_needs_gt_reset() returns true unconditionally for
    EXEC_QUEUE_FLAG_KERNEL
  - xe_sched_invalidate_job(job, 2) is atomic_inc_return(&karma) > 2, so it is
    false on the first pass
  - we therefore take xe_gt_reset_async() and "goto rearm", which returns
    DRM_GPU_SCHED_STAT_NO_HANG *without* reaching the
    xe_sched_job_set_error() / drm_sched_for_each_pending_job() block

and xe_gt_reset_async() is itself a no-op here:

	if (!xe_fault_gt_reset() && xe_uc_reset_prepare(&gt->uc))
		return;

because uc_flr_prepare() already called xe_uc_reset_prepare(), which sets
guc->submission_state.stopped via atomic_fetch_or(). The second call returns
the previous value 1 and the reset is never queued.

So nothing signals the job, the TDR just re-arms on a scheduler that
uc_flr_sanitize() is about to stop, and the untimed, uninterruptible
dma_fence_wait() in xe_migrate_wait() never returns.

This only reproduces when the migrate queue is non-idle at the moment FLR is
triggered. In the idle case m->fence is already signalled and the TDR returns
early on the DMA_FENCE_FLAG_SIGNALED_BIT check, which is why testing has not
hit it.

The invariant that must hold before GuC scheduling and interrupts are
disabled is that outstanding migrate jobs are either drained or cancelled with
their fences signalled. Neither holds today, which is what makes this a real
deadlock rather than a theoretical one.

Note that simply moving xe_tile_migrate_wait() earlier would not be a complete
fix on its own -- it would still leave a drain-versus-submit race unless new
migrations are blocked first. Two patches instead:

  1/2 makes the cancellation branch of the invariant actually hold, by not
      forcing the GT-reset path for a kernel queue that has been deliberately
      killed. The TDR then falls through to xe_sched_job_set_error() and
      signals the timed-out job plus every other pending job with -ECANCELED.

  2/2 drops the xe_bo_pci_dev_remove_pinned() call from FLR prepare, which
      removes the wait entirely (and fixes 3 below).

With 1/2 in place there is no unsignalled kernel-queue fence left across the
teardown, so the stale job is also no longer sitting on the scheduler's pending
list when guc_exec_queue_reinit_kernel() re-initializes the queue after FLR.


=== 2. "Does this error path leave the system in an inconsistent state?" ===

Yes, and you are right that it is pre-existing. The err_display label is
identical to what xe_pm_suspend() already had before this patch, and patch 7
is pure code motion for that part -- it neither introduces nor worsens the
problem. Note also that err_display is only reachable from the else (non-FLR)
branch; the FLR branch has no failure path, since xe_gt_flr_prepare() is void.

Changing the unwind here would be a functional change to the system suspend
path smuggled into a refactor, so it does not belong in this patch. It should
be a separate patch that resumes the already-suspended GTs and restores the
evicted BOs before returning the error.


=== 3. "Does skipping xe_bo_restore_late() during FLR resume create an IOMMU
       bypass or Use-After-Free risk?" ===

The asymmetry is real and is a bug. The security framing is overstated for the
configurations FLR is currently enabled for, but the fix is needed either way.

On the factual question -- does xe_bo_pci_dev_remove_pinned() really unmap
dma-buf DMA, or is it only releasing BAR / TTM I/O / aperture state? It really
unmaps DMA. The full chain, no inference from the name required:

	static void xe_bo_pci_dev_remove_pinned(struct xe_device *xe)
	{
		(void)xe_bo_apply_to_pinned(xe, &xe->pinned.late.external,
					    &xe->pinned.late.external,
					    xe_bo_dma_unmap_pinned);
		...
	}

	int xe_bo_dma_unmap_pinned(struct xe_bo *bo)
	{
		...
		if (ttm_bo->type == ttm_bo_type_sg && ttm_bo->sg) {
			dma_buf_unmap_attachment(ttm_bo->base.import_attach,
						 ttm_bo->sg,
						 DMA_BIDIRECTIONAL);
			ttm_bo->sg = NULL;
			xe_tt->sg = NULL;
		} else if (xe_tt->sg) {
			dma_unmap_sgtable(..., xe_tt->sg, DMA_BIDIRECTIONAL, 0);
			sg_free_table(xe_tt->sg);
			xe_tt->sg = NULL;
		}
		...
	}

That is dma_buf_unmap_attachment() for imported dma-bufs and dma_unmap_sgtable()
for everything else on pinned.late.external. The IOVAs are released.

And FLR resume only calls xe_bo_restore_map() -- early/late kernel_bo_present,
GGTT only. Nothing on the resume side touches pinned.late.external.

Worth adding: calling xe_bo_restore_late() in the FLR branch would not fix it
either. The only thing that re-creates the sg is xe_bo_restore_pinned(), and
it starts with

	struct xe_bo *backup = bo->backup_obj;
	if (!backup)
		return 0;

The backup object is created by xe_bo_evict_pinned(), which FLR deliberately
skips along with xe_bo_evict_all(). So there is no backup object and the
restore is a no-op. There is no "remap dma" helper to pair with the unmap.

On the exfiltration question specifically: for a stale IOVA to be reachable you
need a surviving GPU-visible mapping that points at it. FLR is gated to DGFX
(xe_pci_reset_skip()), where both GGTT and user PPGTT page tables are wiped by
the reset, and xe_bo_restore_map() only re-installs kernel bos. So there is no
surviving mapping pointing at the released IOVAs today. What the asymmetry does
leave behind is pinned external bos with ttm_bo->sg == NULL and xe_tt->sg ==
NULL, still on pinned.late.external, on a device that has just been advertised
as usable again. It would become the IOMMU problem you describe as soon as FLR
is extended to integrated, which is already on the series TODO list.

The right fix is not to add a restore but to not unmap in the first place.
xe_bo_pci_dev_remove_pinned() is a pci_device *removal* helper -- its only
other caller is xe_bo_pci_dev_remove_all(). FLR resets the state of the PCI
function, not the IOMMU domain it is attached to; those mappings stay valid
across the reset and do not need tearing down. unmap_mapping_range() still
handles the CPU side, which is what userspace needs in order to re-fault
against post-FLR state.

This also removes the dma_fence_wait() discussed in 1. The export of
xe_bo_pci_dev_remove_pinned() added in patch 4 is reverted along with its only
new caller.

One fair criticism of the original wording: "IOMMU bypass" is the wrong term,
since an unmapped IOVA faults rather than bypassing anything, and the
corruption/disclosure chain needs more links than were established. The
narrower question -- does xe_bo_pci_dev_remove_pinned() invalidate DMA
addresses that surviving GPU page tables still reference, and if so where are
they restored before submission is re-enabled -- is the better one to ask. The
answer to it is "yes, and nowhere", which is why the patch is still needed.

Patches attached:

  xe-flr-v11-fixup-1-guc-submit-tdr.patch
  xe-flr-v11-fixup-2-pm-suspend.patch

1/2 applies to current upstream and was compile-tested there (W=1, clean). The
revert hunks in 2/2 were round-trip verified against a reconstructed v10
post-image: applying them restores xe_bo_evict.c byte-identically to its
current upstream state.
