## xSMMU vs FANDS on Intel VT-d (RX Path, Linux 6.0.3)

This document summarizes the behavior of the xSMMU and FANDS IOMMU optimizations on Intel VT-d, focusing on RX performance and IOTLB invalidation behavior in Linux 6.0.3, and explains why FANDS can show **lower IOTLB invalidations per IOVA page** yet still achieve **lower RX throughput** than xSMMU.

All code references and numbers below are taken from:
- This tree: Linux 6.0.3 with xSMMU (`/users/chath/disk/kernel/linux-6.0.3-xsmmu`)
- FANDS tree: Linux 6.0.3 + `fands.patch` (`/users/chath/disk/kernel/linux-6.0.3`)
- Experiment logs: `/users/chath/disk/iommu-experiments/xsmmu-intel/invalidation-count`

The RX workloads discussed here are the multi-queue iperf3 RX runs with 60-second measurement periods (200 Gbit NIC).

---

## 1. Measurement methodology

The Intel VT-d driver in both trees is instrumented to collect periodic IOTLB invalidation statistics:

- `drivers/iommu/intel/dmar.c`:
  - `print_iotlb_inv_count` (enabled via `iommu.print_iotlb_inv_count=1`)
  - Counters:
    - `iotlb_inv_psi_count` (page-selective invalidations)
    - `iotlb_inv_dsi_count` (domain-selective invalidations)
    - `iotlb_inv_global_count` (global invalidations)
    - `iotlb_inv_dsi_xsmmu_count` (xSMMU-specific DSIs with IH=1; only in xSMMU tree)
    - `iotlb_unmapped_iova_pages` (4KB IOVA pages unmapped)
  - Periodic 10s summary in dmesg:
    - Line 1: total invalidations and type breakdown (per second rates included)
    - Line 2: per-page metrics:
      - `IOVA_unmapped` (pages / 10s)
      - `inv/page` = TotalInvalidations / IOVA_unmapped
      - `DSI_xSMMU/page` (only when xsmmu DSI counters are non-zero)

The key metric used here is:

- **`inv/page`**: IOTLB invalidations per unmapped IOVA page over each 10-second window.

In both the FANDS and xSMMU trees, **the same accounting is used**:

- `drivers/iommu/intel/iommu.c`:
  - `intel_iommu_unmap()` calls `atomic64_add(size >> VTD_PAGE_SHIFT, &iotlb_unmapped_iova_pages);`
  - All IOMMU unmap paths (`iommu_unmap_fast()` / `iommu_unmap()`) funnel through this, so the IOVA page count is comparable across trees.

---

## 2. Representative RX results (10s windows)

Below are representative 10s windows from the dmesg logs for strict, FANDS, and xSMMU under the RX multi-queue workload on Intel VT-d.

### 2.1. Strict mode (baseline)

File: `strict.6.0.3.rx.multi.dmesg.log`

Example RX-heavy windows (times in seconds):

- Around `t ≈ 407–448 s`:
  - `Total ≈ 2,191,091` invalidations over 10s (`219,109.1/s`)
  - `IOVA_unmapped ≈ 12,078,275` pages (`1,207,827.5/s`)
  - `inv/page ≈ 0.181`

Interpretation:

- Strict mode (no batching optimizations) is close to **one invalidation per page** for many windows, but with some coalescing via normal VT-d PSI mechanisms.

### 2.2. FANDS (RX, multi-queue)

File: `fands.rx.multi.dmesg.log`

In the RX iperf3 window (HIGH IOVA_unmapped), e.g. around `t ≈ 3663–3694 s`:

- `t = 3663.754302`:
  - `Total = 1,768,801` (invalidations / 10s, all PSI)
  - `IOVA_unmapped = 37,000,506` pages / 10s
  - `inv/page ≈ 0.047`
- `t = 3673.994112`:
  - `Total = 2,032,335`
  - `IOVA_unmapped = 43,013,732`
  - `inv/page ≈ 0.047`
- `t = 3684.234089`:
  - `Total = 2,030,722`
  - `IOVA_unmapped = 43,117,756`
  - `inv/page ≈ 0.047`

So for RX:

- **FANDS achieves `inv/page ≈ 0.047`**, i.e. about **1 invalidation per ~21 pages** on average in these windows.
- However, note the **huge `IOVA_unmapped` counts**: tens of millions of pages per 10 seconds, indicating a lot of mapping churn.

### 2.3. xSMMU (RX, multi-queue, 6.0.3)

File: `xsmmu.6.0.3.rx.multi.dmesg.log`

In the RX iperf3 window (HIGH IOVA_unmapped), looking at DSI/DSI_xSMMU-heavy windows, e.g. around `t ≈ 223–264 s`:

- `t = 223.221359`:
  - `Total = 636,998` (≈63,699.8/s)
  - `PSI = 869`, `DSI = 636,129`, `DSI_xSMMU = 621,385`
  - `IOVA_unmapped = 8,355,443` pages / 10s
  - `inv/page ≈ 0.076`
  - `DSI_xSMMU/page ≈ 0.074`
- `t = 233.461214`:
  - `Total = 1,517,187`
  - `PSI = 15,600`, `DSI = 1,501,587`, `DSI_xSMMU = 1,465,182`
  - `IOVA_unmapped = 20,616,292` pages / 10s
  - `inv/page ≈ 0.073`
  - `DSI_xSMMU/page ≈ 0.071`
- `t = 243.701082`:
  - `Total = 1,524,613`
  - `IOVA_unmapped = 20,407,123`
  - `inv/page ≈ 0.075`

So for RX:

- **xSMMU sees `inv/page ≈ 0.073–0.076`**, i.e. about **1 invalidation per 13–14 pages** in these windows.
- Absolute counts:
  - `Total` invalidations and `IOVA_unmapped` are significantly lower than in the FANDS RX runs, even though `inv/page` is numerically worse.

Empirically:

- Strict: high invalidations, high unmaps, `inv/page ≈ 0.18–1.0`, lowest throughput.
- FANDS: **very low `inv/page` (~0.047)** but very high `IOVA_unmapped`, medium throughput.
- xSMMU: slightly worse `inv/page` (~0.073) but **lower total invalidations and fewer unmapped pages**, and **highest RX throughput**.

---

## 3. Design comparison (RX path)

### 3.1. FANDS RX batching model

FANDS’ goal is to keep the IOMMU running in **strict invalidation mode** while reducing the number of invalidations by batching at the IOVA level, especially for RX MPWQE:

1. **Contiguous IOVA block per MPWQE**
   - In `mlx5e_alloc_rx_mpwqe()` (FANDS tree, 6.0.3):
     - Allocate a single contiguous IOVA block for all pages in the MPWQE:
       - `iova_allocation_size = 4096 * MLX5_MPWRQ_PAGES_PER_WQE;`
       - `iova_base = iommu_dma_alloc_iova(...)`
     - For each page (index `i`):
       - `dma_info->batch_iova = true;`
       - `dma_info->iova = iova_base + i * 4096;`
       - `dma_info->iova_size = iova_allocation_size;`
       - `dma_info->first_iova = (i == 0);`

2. **Mapping into the block**
   - `mlx5e_page_alloc_pool()`:
     - If `batch_iova` is set, use:
       - `dma_map_page_attrs_iova()` → `iommu_dma_map_page_iova()` → `__iommu_dma_map(..., iova_addr, first_iova, ...)`
     - The **first page** of the block (`first_iova = true`) triggers `ops->iotlb_sync_map(domain, iova, size * 64)`, i.e. **one map-side invalidation covering 64 pages**.

3. **Unmapping/freeing the block**
   - In `mlx5e_free_rx_mpwqe()` (FANDS tree) for IOMMU-mapped devices:

   ```c
   if (device_iommu_mapped(rq->pdev)) {
       dma_info[i].iova_size = 4096 * 64;
       dma_info[i].batch_iova = true;
       dma_info[i].free_iova = i == (MLX5_MPWRQ_PAGES_PER_WQE - 1);
       mlx5e_page_release(rq, &dma_info[i], false);
   }
   ```

   - `recycle = false` forces:
     - `mlx5e_page_release_dynamic(..., recycle=false, ...)` to:
       - Always call `mlx5e_page_dma_unmap()` and then `page_pool_release_page() + put_page()`.
     - The last page in the block (`free_iova = true`) unmaps the entire 64-page range and frees the IOVA in one go via:
       - `dma_unmap_page_attrs_iova()` → `iommu_dma_unmap_page_iova()` → `__iommu_dma_unmap_iova()`.
   - `__iommu_dma_unmap_iova()`:
     - Calls `iommu_unmap_fast(domain, dma_addr, size, &iotlb_gather)` with `size = 64 * 4KB`.
     - Calls `iommu_iotlb_sync(domain, &iotlb_gather)` if the unmap is not queued.
     - Adds `size >> 12` to `iotlb_unmapped_iova_pages`.

4. **Resulting behavior**
   - Each MPWQE gets:
     - One map-side invalidation (for 64 pages).
     - One unmap-side invalidation (for the 64-page region).
   - Hence **low `inv/page` (~0.047)**: many pages per invalidation.
   - But RX pages tied to that contiguous IOVA block are **not recycled** when `device_iommu_mapped()` is true:
     - They are always unmapped and freed at MPWQE teardown.
     - On the next RX cycle, new pages and mappings must be allocated.

### 3.2. xSMMU RX batching model

The xSMMU design:

1. **No change to RX reuse logic**
   - Upstream RX reuse logic is preserved:
     - Pages can be reused via:
       - RX cache (`mlx5e_rx_cache_get()` / `mlx5e_rx_cache_put()`).
       - `page_pool_recycle_direct()` when they leave the cache.
   - `mlx5e_page_release_dynamic()` in xSMMU:

   ```c
   void mlx5e_page_release_dynamic(struct mlx5e_rq *rq, struct page *page, bool recycle)
   {
       if (likely(recycle)) {
           if (mlx5e_rx_cache_put(rq, page))
               return;

           if (xsmmu_batch_unmap) {
               if (rq->pending_release_count >= iommu_napi_poll_threshold)
                   mlx5e_process_rx_release_batch(rq);

               mlx5e_page_dma_unmap_no_sync(rq, page);
               rq->pending_release[rq->pending_release_count].page = page;
               rq->pending_release[rq->pending_release_count].recycle = true;
               rq->pending_release_count++;
               ...
               return;
           }

           mlx5e_page_dma_unmap(rq, page);
           page_pool_recycle_direct(rq->page_pool, page);
       } else {
           if (xsmmu_batch_unmap) {
               ...
           }

           mlx5e_page_dma_unmap(rq, page);
           page_pool_release_page(rq->page_pool, page);
           put_page(page);
       }
   }
   ```

   - Datapath calls (e.g. `mlx5e_free_rx_mpwqe(..., true)`) keep `recycle = true`, so:
     - RX cache + page_pool reuse is active.
     - Only when a page actually leaves the page_pool path (recycle fails or teardown) do we unmap/finally free it.

2. **Deferred unmap + domain-wide sync**
   - When `xsmmu_batch_unmap` is enabled (`iommu.xsmmu=1`):
     - Per-page unmaps use:
       - `dma_unmap_page_attrs_no_sync()` → `iommu_dma_unmap_swiotlb_no_sync()` → `__iommu_dma_unmap_no_sync()`.
       - These calls unmap and free IOVA but **do not** trigger IOTLB invalidation.
     - At the end of the NAPI poll (or when `pending_release_count` hits `iommu_napi_poll_threshold`), the driver calls:

       ```c
       void mlx5e_process_rx_release_batch(struct mlx5e_rq *rq)
       {
           if (rq->pending_release_count == 0)
               return;

           dma_sync_device_iotlb(rq->pdev);
           // then recycle/release all queued pages
       }
       ```

     - `dma_sync_device_iotlb()` in xSMMU:
       - Uses `domain->ops->sync_domain_iotlb`, implemented in VT-d as:
         - `intel_sync_domain_iotlb()` → `qi_flush_domain_iotlb_hint()` with IH=1 (domain-wide DSI).

3. **Resulting behavior**
   - RX pages:
     - Are **reused normally** via RX cache and page_pool.
     - Only a subset of pages are actually unmapped in a given time window (when they truly leave the pool).
   - Invalidations:
     - Are **deferred and batched per NAPI poll** via domain-wide DSIs.
     - Each DSI covers all outstanding unmapped IOVAs queued in the flush queue for that domain.

Compared to FANDS, xSMMU:

- Does **fewer total unmaps and IOVA frees** (because pages live longer).
- Issues **fewer total invalidations** (despite slightly worse `inv/page`).
- Achieves **higher RX throughput** because the overall IOMMU and memory allocator work per byte is lower.

---

## 4. Why FANDS disables recycling for MPWQE under IOMMU

In FANDS’ design, the contiguous IOVA block used for an MPWQE is treated as an **ephemeral object** tied to the lifetime of that WQE:

- All pages in the block are mapped once for that WQE.
- On WQE teardown:
  - All pages are unmapped and freed.
  - The IOVA block is unconditionally freed on the last page.

This creates a strong invariant:

> Each batched IOVA block is owned by exactly one MPWQE and is destroyed (unmapped + IOVA freed) when that MPWQE is freed. No page from that block is reused in another WQE while the block still exists.

If you allowed normal RX recycling (`recycle = true`) in `mlx5e_free_rx_mpwqe()` for the IOMMU case:

- Some subset of pages from the contiguous IOVA block could:
  - Be returned to RX cache/page_pool.
  - Later be reattached to **new** MPWQEs while still tied to the original IOVA block assumptions (`iova`, `iova_size`, `free_iova`).
- The current batching logic assumes:
  - `free_iova` is set **only on the last page** and that all pages of the block are going away together.
  - When `free_iova` is true, `__iommu_dma_unmap_iova()` can safely unmap the entire range and free the IOVA once.

With recycling, that assumption breaks:

- You risk:
  - Freeing an IOVA block that still backs reused pages.
  - Double-freeing IOVAs when a reused page later hits another `free_iova` path.
  - Retaining mappings (and stale IOTLB entries) for pages that were logically freed, violating strict semantics.

To fix this properly you would need:

- Per-IOVA-block lifetime tracking and refcounting.
- A way to detach recycled pages from their original block and possibly remap them to new blocks.
- More complex rules around when `free_iova` can be set and how large a range can be unmapped.

The FANDS patch does not implement this additional complexity. Instead, it takes the **simple and safe** approach:

- When `device_iommu_mapped(rq->pdev)`:
  - **Do not recycle MPWQE pages** at teardown; always free them.
  - This ensures the “one MPWQE ↔ one IOVA block” model remains correct.

This design choice explains the empirical behavior:

- **FANDS**
  - Excellent per-invalidations amortization (`inv/page ≈ 0.047`).
  - But **much higher total IOVA unmaps and invalidations** due to heavy page churn (no RX reuse for those MPWQE pages under IOMMU).
  - RX throughput is limited by this extra IOMMU and allocator work.

- **xSMMU**
  - Slightly worse `inv/page` (~0.073).
  - But **lower total mappings/unmaps** and **lower total invalidations** because page reuse is preserved and invalidations are deferred and batched domain-wide.
  - As observed experimentally, this yields **higher RX throughput** on the 200 Gbit setup.

---

## 5. Conceptual perspective: F&S vs xSMMU

### 5.1. Is recycling fundamentally incompatible with F&S?

At the paper level, Fast & Safe (F&S) does **not** state that page recycling is forbidden. The core ideas are:

- Allocate **contiguous IOVA regions per NIC queue / ring**.
- Exploit **IOMMU page-walk caches** so IOTLB misses are cheap (even in strict mode).
- Batch invalidations over those regions while still unmapping and invalidating immediately after DMA (strict semantics).

In principle, you could:

- Keep recycled pages mapped within the same contiguous IOVA regions, and
- Maintain per-region reference counting so a region is only unmapped/freed when no page in that region is in use.

What the FANDS implementation does, however, is adopt a **much simpler invariant**:

- One MPWQE owns one contiguous IOVA block.
- All pages in that block are freed at MPWQE teardown.
- The IOVA block is freed once on the last page.

That invariant is **practically incompatible with RX recycling in the current Linux `mlx5` + page_pool design** unless you introduce substantial extra bookkeeping (per-block refcounts, ownership tracking, etc.). Thus:

- It is not that F&S fundamentally cannot support recycling.
- Rather, **the specific FANDS implementation chooses to disable recycling for MPWQE pages under IOMMU** so that its block-based batching logic stays simple and obviously correct.

### 5.2. Why FANDS can be slower despite better `inv/page`

A concise way to frame the difference:

- **FANDS optimizes the *cost per IOTLB event* in isolation.**
  - It reduces the number of invalidations per unmapped page (`inv/page ≈ 0.047`) by aligning RX MPWQE layout with contiguous IOVA blocks and batching invalidations over those blocks.
  - But in this Linux implementation it pays for that by:
    - Turning off RX recycling for IOMMU-mapped MPWQE pages.
    - Causing **many more map/unmap operations** and higher `IOVA_unmapped` per unit traffic.
  - This is an optimization “near” the IOMMU, but somewhat at odds with how the RX buffer pipeline (page_pool + cache) wants to behave.

- **xSMMU optimizes the *end-to-end DMA/IOMMU pipeline*.**
  - It keeps the upstream RX reuse model intact (RX cache + page_pool).
  - It changes the invalidation contract: explicit `_no_sync` unmaps + deferred **domain-wide IH=1 DSIs** via `sync_domain_iotlb`.
  - Thus it:
    - Unmaps **fewer IOVA pages overall** for the same workload.
    - Issues **fewer total invalidations**, even if each invalidation covers fewer pages on average (`inv/page ≈ 0.073`).
  - xSMMU effectively co-designs batching with the NAPI/pool lifecycle rather than with a single MPWQE.

This explains the experimental result:

- FANDS looks better on the **`inv/page` micro-metric**, but
- xSMMU wins on **throughput**, because throughput is dominated by the absolute amount of map/unmap/flush work and allocator churn, not just by `inv/page`.

### 5.3. Why the FANDS implementation is RX-only (and not applied to TX)

The F&S ideas could, in principle, apply to both RX and TX, but the Linux FANDS patch applies them only to RX MPWQE in `mlx5`. The reasons are practical:

- **RX MPWQE + page_pool is a good fit:**
  - Buffers are driver-owned pages with a relatively simple lifetime.
  - MPWQE already groups a fixed number of pages per WQE.
  - Contiguous IOVA blocks per MPWQE and “free once at WQE teardown” semantics are straightforward.

- **TX is much more complex in Linux/`mlx5`:**
  - TX buffers come from many sources (socket send buffers, page cache pages, user-space zero-copy), often via scatter-gather.
  - Pages may be shared across WQEs, queues, and cloned SKBs; their lifetime is not tied to a single WQE.
  - To impose a FANDS-style contiguous IOVA block model on TX, you would need either:
    - A driver-owned TX slab + copy path (software bounce buffers), or
    - Much more invasive tracking (per-block refcounts, tracking all TX references to a region).

Given those constraints, the FANDS patch:

- Chooses to implement the F&S idea where the required invariants are easiest to satisfy (RX MPWQE).
- Leaves TX unchanged, which is why:
  - RX sees the characteristic FANDS behavior (low `inv/page`, high `IOVA_unmapped`, no recycling).
  - TX does not benefit from FANDS-style batching, whereas xSMMU explicitly optimizes both TX and RX.

## 6. Summary

- The surprising observation that **FANDS has fewer IOTLB invalidations per IOVA page on RX** yet lower throughput than xSMMU is explained by:
  - FANDS’ MPWQE/IOMMU batching model sacrificing RX page reuse (for IOMMU-mapped traffic) to maintain a simple, strict, per-block batching invariant.
  - xSMMU preserving normal RX reuse and instead optimizing **how and when** invalidations are issued (deferred domain-wide DSIs with IH=1).

- Both trees share the same `iotlb_unmapped_iova_pages` accounting in `intel_iommu_unmap()`, so the differences are due to:
  - **How often mappings are created/destroyed**, and
  - **How invalidations are batched**, not due to mismatched counters.

- Conceptually:
  - FANDS, as implemented here, is a **strict, per-MPWQE IOVA-block batching scheme** that trades reuse and simplicity for more overall churn.
  - xSMMU is a **deferred, domain-wide batching scheme** that keeps the standard Linux RX reuse model and also extends batching to TX, reducing total IOMMU and allocator work—which is why it achieves higher throughput despite a slightly worse `inv/page` metric on RX.

