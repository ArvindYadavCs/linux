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

Fix: drop the xe_bo_pci_dev_remove_pinned() call from the FLR prepare path
(see 3 below), which removes the wait entirely. The underlying issue -- that a
kernel-queue TDR during FLR teardown leaves pending fences unsignalled -- has
no waiter left after that, but it is still latent in patch 3's teardown and is
worth a separate look.


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

Confirmed: FLR prepare calls xe_bo_pci_dev_remove_pinned(), which runs
xe_bo_dma_unmap_pinned() over xe->pinned.late.external, and FLR resume only
calls xe_bo_restore_map() -- early/late kernel_bo_present, GGTT only. Nothing
on the resume side touches pinned.late.external.

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

This also removes the dma_fence_wait() discussed in 1, so both issues go away
with one change. The export of xe_bo_pci_dev_remove_pinned() added in patch 4
can be reverted along with it.

Patch attached as xe-flr-v11-fixup.patch.
