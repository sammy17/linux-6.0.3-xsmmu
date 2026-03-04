# Plan: xSMMU Batch Unmap for NVMe Storage Driver

## Goal

Apply the same xSMMU batch unmap optimization used in the networking (MLX5) stack to the NVMe PCI host driver. Multiple completed I/O requests per completion-queue poll are unmapped with `_no_sync` APIs, then a single `dma_sync_device_iotlb()` is performed before any DMA buffer or metadata is freed. This reduces IOTLB invalidation cost for high-IOPS NVMe workloads.

**Gating:** The optimization is used **only when the `iommu.xsmmu=1` boot parameter is set**. When `xsmmu_batch_unmap` is false, the driver keeps the current sync-unmap-per-request behavior.

---

## Security Invariant

**Requirement:** The device must not use an IOVA after it is unmapped. Therefore we must **not free or reuse DMA buffer memory** (or related PRP/SGL metadata) until **after** `dma_sync_device_iotlb()` has completed for those unmaps.

**Ordering:**

1. For each completed request in the batch: unmap data (and integrity metadata) with `_no_sync` APIs only; do **not** free PRP/SGL lists or `iod->sgt.sgl` yet.
2. Call `dma_sync_device_iotlb(dev->dev)` **once** for the batch.
3. Then for each request: free PRP/SGL/sgl and call `nvme_complete_batch_req(req)`.

Same as networking: **unmap (all in batch) → single sync → then free/complete**.

---

## Scope

- **In scope:** NVMe PCI host driver (`drivers/nvme/host/pci.c`) for read and write I/O.
- **Out of scope (for this plan):** NVMe over Fabrics (RDMA, TCP, FC), Apple NVMe, and other transports; they can be addressed later with the same pattern if desired.

---

## Step-by-Step Implementation Plan

### Phase 1: DMA API and IOMMU layer — add scatter-gather no-sync unmap

The existing xSMMU code provides `unmap_page_no_sync` and `sync_device_iotlb`. NVMe uses `dma_unmap_sgtable()` for multi-segment I/O, so we need a `_no_sync` path for scatter-gather.

#### Step 1.1 — Add `unmap_sg_no_sync` to `struct dma_map_ops`

- **File:** `include/linux/dma-map-ops.h`
- **Change:** In `struct dma_map_ops`, add an optional callback:
  - `void (*unmap_sg_no_sync)(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction dir, unsigned long attrs);`
- **Note:** Optional (can be NULL). When NULL, drivers must use the normal sync unmap path.

#### Step 1.2 — Implement `iommu_dma_unmap_sg_no_sync` in dma-iommu

- **File:** `drivers/iommu/dma-iommu.c`
- **Change:**
  - Add `iommu_dma_unmap_sg_no_sync()`. Reuse the same logic as `iommu_dma_unmap_sg()` for computing the single contiguous IOVA range (start/end) from the scatterlist, but call `__iommu_dma_unmap_no_sync(dev, start, end - start)` instead of `__iommu_dma_unmap()`. Handle `dev_use_swiotlb(dev)` by iterating segments and calling the existing no-sync page unmap (e.g. `iommu_dma_unmap_swiotlb_no_sync` or equivalent per-segment unmap) so SWIOTLB and IOMMU paths both have a no-sync SG path.
  - In the non-SWIOTLB path, do **not** call `iommu_dma_sync_sg_for_cpu` in the no_sync variant (or document that callers must handle CPU sync separately if needed; for NVMe completion path we typically use `DMA_ATTR_SKIP_CPU_SYNC` where appropriate).
  - Register the new callback in `iommu_dma_ops`: set `.unmap_sg_no_sync = iommu_dma_unmap_sg_no_sync` (or a SWIOTLB-aware wrapper that dispatches to the right implementation).
- **Reference:** Existing `iommu_dma_unmap_sg()` (lines ~1562–1610) and `__iommu_dma_unmap_no_sync` usage elsewhere in the file.

#### Step 1.3 — Add public DMA API helpers

- **File:** `include/linux/dma-mapping.h`
  - Declare:
    - `void dma_unmap_sg_attrs_no_sync(struct device *dev, struct scatterlist *sg, int nents, enum dma_data_direction dir, unsigned long attrs);`
    - `void dma_unmap_sgtable_attrs_no_sync(struct device *dev, struct sg_table *sgt, enum dma_data_direction dir, unsigned long attrs);`
  - Add a macro wrapper for the no-attrs case if desired (e.g. `dma_unmap_sg_no_sync`).
- **File:** `kernel/dma/mapping.c`
  - Implement `dma_unmap_sg_attrs_no_sync`: if `ops->unmap_sg_no_sync` is non-NULL, call it; otherwise call `dma_unmap_sg_attrs()` so non-IOMMU or non-xSMMU setups are unchanged.
  - Implement `dma_unmap_sgtable_attrs_no_sync` by calling `dma_unmap_sg_attrs_no_sync(dev, sgt->sgl, sgt->orig_nents, dir, attrs)`.

---

### Phase 2: NVMe PCI driver — batch unmap in completion path

All changes are gated by `xsmmu_batch_unmap` (i.e. only when `iommu.xsmmu=1`).

#### Step 2.1 — Add NVMe PCI unmap-no-sync helper

- **File:** `drivers/nvme/host/pci.c`
- **Change:**
  - Add a new function `nvme_pci_unmap_rq_no_sync(struct request *req)` that:
    - Uses `iod = blk_mq_rq_to_pdu(req)`, `dev = iod->nvmeq->dev`.
    - If `blk_integrity_rq(req)`: call `dma_unmap_page_attrs_no_sync(dev->dev, iod->meta_dma, rq_integrity_vec(req)->bv_len, rq_data_dir(req), 0)` (or appropriate attrs).
    - If `blk_rq_nr_phys_segments(req)`: call a new helper (e.g. `nvme_unmap_data_no_sync(dev, req)`) that:
      - If `iod->dma_len`: `dma_unmap_page_attrs_no_sync(dev->dev, iod->first_dma, iod->dma_len, rq_dma_dir(req), 0)`.
      - Else: `dma_unmap_sgtable_attrs_no_sync(dev->dev, &iod->sgt, rq_dma_dir(req), 0)`.
    - **Do not** free PRP/SGL or `iod->sgt.sgl` in this function; that happens after sync in the batch path.
  - Add `nvme_unmap_data_no_sync(struct nvme_dev *dev, struct request *req)` that contains the above data-unmap-only logic (no free of prp/sgl/sgl).
- **Dependency:** Step 1.3 must be done so `dma_unmap_sgtable_attrs_no_sync` exists.

#### Step 2.2 — Add NVMe PCI “free after unmap” helper

- **File:** `drivers/nvme/host/pci.c`
- **Change:**
  - Add `nvme_pci_free_rq_resources(struct request *req)` that performs only the resource free that today is done inside `nvme_unmap_data()` after the unmap: for the sgt case, free PRP/SGL (e.g. `dma_pool_free` / `nvme_free_prps` / `nvme_free_sgls`) and `mempool_free(iod->sgt.sgl, dev->iod_mempool)`. This must be called only **after** `dma_sync_device_iotlb()` when using the batch path.
  - Ensure `nvme_unmap_data()` remains the single place that does unmap + free for the non-xSMMU path so we don’t duplicate logic.

#### Step 2.3 — Use xSMMU batch only when `iommu.xsmmu=1`

- **File:** `drivers/nvme/host/pci.c`
- **Change:**
  - Include or reference the xSMMU gate (e.g. `extern bool xsmmu_batch_unmap` from dma-iommu or a header that exports it when `CONFIG_IOMMU_DMA` is set).
  - In the **batch completion** path:
    - When `xsmmu_batch_unmap` is true:
      1. For each request in the batch: call `nvme_pci_unmap_rq_no_sync(req)`.
      2. Call `dma_sync_device_iotlb(dev->dev)` once (use the same `dev` as the requests, e.g. from the first request in the batch: `iod->nvmeq->dev`).
      3. For each request: call `nvme_pci_free_rq_resources(req)`, then `nvme_complete_batch_req(req)`.
    - When `xsmmu_batch_unmap` is false: keep current behavior — for each request call `nvme_pci_unmap_rq(req)` then `nvme_complete_batch_req(req)` (current `nvme_complete_batch(iob, nvme_pci_unmap_rq)`).
  - Single-request completion path (`nvme_pci_complete_rq`) remains unchanged: always call `nvme_pci_unmap_rq(req)` then `nvme_complete_rq(req)`. No batching, so no _no_sync usage there.

#### Step 2.4 — Wire batch completion to the new flow

- **File:** `drivers/nvme/host/pci.c`
- **Change:**
  - Replace the current `nvme_pci_complete_batch()` implementation that calls `nvme_complete_batch(iob, nvme_pci_unmap_rq)` with logic that:
    - If `!xsmmu_batch_unmap`: call `nvme_complete_batch(iob, nvme_pci_unmap_rq)` (unchanged).
    - If `xsmmu_batch_unmap`: run the three-step sequence (unmap_no_sync all → sync_device_iotlb once → free + complete_batch_req for each).
  - Ensure the same `struct nvme_dev *dev` is used for `dma_sync_device_iotlb(dev->dev)` for all requests in the batch (all requests in a batch are from the same queue, so same `dev`).

---

### Phase 3: Cleanup and fallbacks

#### Step 3.1 — No-sync API fallback when ops lack `unmap_sg_no_sync`

- **File:** `kernel/dma/mapping.c`
- **Change:** In `dma_unmap_sg_attrs_no_sync`, if `ops->unmap_sg_no_sync` is NULL, fall back to `dma_unmap_sg_attrs()` so that when IOMMU or platform does not provide the callback, behavior is the same as today (sync unmap).

#### Step 3.2 — NVMe: avoid xSMMU path when no-sync is not available

- **File:** `drivers/nvme/host/pci.c`
- **Change:** Optionally, if the DMA device does not support `unmap_sg_no_sync` (e.g. ops is NULL or callback is NULL), do not use the batch no-sync path even when `xsmmu_batch_unmap` is true; fall back to per-request sync unmap. This can be done by checking `get_dma_ops(dev->dev)->unmap_sg_no_sync` before choosing the batch path. (Optional step for robustness.)

---

## Files and functions summary

| File | Function / area | Change |
|------|------------------|--------|
| `include/linux/dma-map-ops.h` | `struct dma_map_ops` | Add optional `unmap_sg_no_sync`. |
| `drivers/iommu/dma-iommu.c` | New + `iommu_dma_ops` | Add `iommu_dma_unmap_sg_no_sync`, handle SWIOTLB; register in ops. |
| `include/linux/dma-mapping.h` | Declarations | Add `dma_unmap_sg_attrs_no_sync`, `dma_unmap_sgtable_attrs_no_sync`. |
| `kernel/dma/mapping.c` | New | Implement both; fallback to sync unmap if no_sync is NULL. |
| `drivers/nvme/host/pci.c` | New helpers | `nvme_unmap_data_no_sync`, `nvme_pci_unmap_rq_no_sync`, `nvme_pci_free_rq_resources`. |
| `drivers/nvme/host/pci.c` | `nvme_unmap_data` | Keep as-is for non-batch path; ensure free logic can be called from `nvme_pci_free_rq_resources` (refactor if needed). |
| `drivers/nvme/host/pci.c` | `nvme_pci_complete_batch` | When `xsmmu_batch_unmap`: unmap_no_sync all → sync_device_iotlb once → free + complete each. Else: current behavior. |
| `drivers/nvme/host/pci.c` | `nvme_pci_complete_rq` | No change; always sync unmap. |

---

## Testing and verification

1. **Boot:** With `iommu.xsmmu=0` (default) or without the parameter, no behavior change; all unmaps remain sync.
2. **Boot:** With `iommu.xsmmu=1`, run read/write workloads (e.g. fio on NVMe); check that I/O completes correctly and no use-after-free or IOMMU faults.
3. **Error paths:** Requests that do not enter the batch (e.g. non-zero status, or `blk_mq_add_to_batch` returns false) still go through `nvme_pci_complete_rq` with sync unmap.
4. **Teardown / reset:** Ensure queue teardown and request cancellation paths still use sync unmap (no batching there).

---

## Dependency order

1. Phase 1 (Steps 1.1 → 1.2 → 1.3): DMA/IOMMU layer first.
2. Phase 2 (Steps 2.1 → 2.2 → 2.3 → 2.4): NVMe changes, gated by `iommu.xsmmu=1`.
3. Phase 3: Fallbacks and optional checks.

---

## Summary

- **Gating:** xSMMU batch unmap in NVMe is used **only when `iommu.xsmmu=1`**.
- **Batch boundary:** Existing CQ poll batch (same as current `nvme_pci_complete_batch`).
- **Ordering:** Unmap (no_sync) for all requests in batch → one `dma_sync_device_iotlb(dev->dev)` → then free resources and complete each request.
- **Scope:** NVMe PCI host driver only; other transports unchanged.
