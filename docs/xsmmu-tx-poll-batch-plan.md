# Plan: Expand xSMMU TX batch to CQ poll

## Goal

Today the TX path does **one `dma_sync_device_iotlb()` per WQE** when batching (num_dma > threshold). Expand batching so we do **one sync at the end of each TX CQ poll**, i.e. all WQEs completed in that poll share a single domain-wide IOTLB invalidation. This improves amortization (e.g. tens to hundreds of segments per sync instead of 1–3 per WQE).

---

## Invariant: IOTLB invalidation before freeing DMA buffer memory

**Requirement:** The device must not be able to use an IOVA after we have unmapped it. So we must **not free or reuse the DMA buffer memory** (SKB and its data) until **after** `dma_sync_device_iotlb()` has completed for the unmaps that covered that buffer.

**Current (per-WQE sync):** For each WQE we do: unmap_no_sync (all segments) → sync → consume_skb. So sync completes before any free. **Correct.**

**Proposed (per-poll sync):** We will do:
1. **In the poll loop:** For each completed WQE that uses batch: unmap_no_sync (all segments); **do not** call `dma_sync_device_iotlb()`; **do not** call `mlx5e_consume_skb` / `mlx5e_tx_wi_consume_fifo_skbs` yet — record these for deferred consume.
2. **After the poll loop:** If any batch unmaps were done this poll, call `dma_sync_device_iotlb(sq->pdev)` **once**.
3. **Then** drain the deferred list: for each entry, call `mlx5e_consume_skb` (or pop and consume for fifo).

Ordering is therefore: **unmap (all WQEs in poll) → single sync → then consume/free**. So IOTLB invalidation is complete before any DMA buffer memory is freed. **Correct.**

---

## Files and functions to edit

### 1. `drivers/net/ethernet/mellanox/mlx5/core/en_tx.c`

| Function | Change |
|----------|--------|
| **`mlx5e_tx_wi_dma_unmap()`** | Add parameter `bool defer_sync`. When `batch && defer_sync`: do unmap_no_sync only, **do not** call `dma_sync_device_iotlb()`, and return `true`. When `batch && !defer_sync`: keep current behavior (unmap_no_sync then sync, return `false`). When !batch: unchanged, return `false`. Callers that pass `defer_sync == true` must defer consume until after a single sync at poll end. |
| **`mlx5e_poll_tx_cq()`** | (a) Add a stack array for deferred consumes: e.g. `struct { struct sk_buff *skb; u8 num_fifo_pkts; u64 cqe_ts; } deferred[MLX5E_TX_CQ_POLL_BUDGET * 2]` and `deferred_count`. (b) In the inner do-while, when `wi->skb`: call `mlx5e_tx_wi_dma_unmap(sq, wi, &dma_fifo_cc, true)`. If it returns true, push `(wi->skb, 0, get_cqe_ts(cqe))` to `deferred` and do **not** call `mlx5e_consume_skb`; else call `mlx5e_consume_skb` as today. (c) When `wi->num_fifo_pkts`: same — if unmap returns true, push `(NULL, wi->num_fifo_pkts, get_cqe_ts(cqe))` and do not call `mlx5e_tx_wi_consume_fifo_skbs`; else call it as today. (d) After the outer do-while, if `deferred_count > 0`: call `dma_sync_device_iotlb(sq->pdev)` once, then drain: for each deferred entry, if `skb` then build a minimal CQE (e.g. stack `struct mlx5_cqe64` with timestamp set from `cqe_ts`) and call `mlx5e_consume_skb(sq, skb, &fake_cqe, napi_budget)`; else pop `num_fifo_pkts` SKBs from `sq->db.skb_fifo` and consume each with the same fake_cqe. |
| **`mlx5e_free_txqsq_descs()`** | Call `mlx5e_tx_wi_dma_unmap(sq, wi, &dma_fifo_cc, false)` so teardown always syncs before freeing. If we later change the return to “did batch” we can add: when true, call `dma_sync_device_iotlb(sq->pdev)` before `dev_kfree_skb_any` / `mlx5e_tx_wi_kfree_fifo_skbs`. With `defer_sync == false`, `mlx5e_tx_wi_dma_unmap` will sync inside itself when batch, so no extra call needed. |

**Unchanged:**

- **`mlx5e_dma_unmap_wqe_err()`** — Called from xmit path on mapping failure; no consume. Must sync immediately after unmaps. **No change** (no defer_sync; keep current per-WQE sync when batch).

---

### 2. `drivers/net/ethernet/mellanox/mlx5/core/en.h` (optional)

- No strict need to add fields to `struct mlx5e_txqsq` if we use a **stack** array in `mlx5e_poll_tx_cq` for deferred entries. That keeps the change local to the poll function and avoids per-SQ memory.

---

### 3. CQE timestamp for deferred consume

- `mlx5e_consume_skb()` only uses `get_cqe_ts(cqe)` for hwtstamp. When draining deferred entries we have stored `cqe_ts`. Use a stack `struct mlx5_cqe64` in the drain loop, set the timestamp field to the stored `cqe_ts` (layout must match what `get_cqe_ts()` reads), and pass that as `cqe` to `mlx5e_consume_skb()`. Need to confirm the CQE64 layout in the mlx5 core headers for the timestamp location.

---

## Summary: ensuring IOTLB complete before free

| Step | Action |
|------|--------|
| 1 | In the poll loop, for batch WQEs: only unmap_no_sync; record (skb or num_fifo_pkts, cqe_ts) for later consume; **do not** free SKBs yet. |
| 2 | After the loop, if any batch unmaps: call `dma_sync_device_iotlb(sq->pdev)` **once**. This waits until the IOMMU has completed IOTLB invalidation for all unmaps done in this poll. |
| 3 | Then drain the deferred list: for each entry, consume_skb (or pop + consume for fifo). Only at this point is DMA buffer memory freed. |

So **IOTLB invalidation is guaranteed to complete before any DMA buffer memory is freed**, same as today, just with one sync per poll instead of per WQE.

---

## Edge cases

- **REQ_ERR CQE:** We still unmap and can defer consume; after the single sync we drain deferred including those from the same poll. No special case.
- **ktls resync / num_fifo_pkts:** Same rule: if unmap returns “defer”, we defer that WQE’s consume (single skb or fifo) and drain after sync.
- **mlx5e_free_txqsq_descs:** Always pass `defer_sync = false` so we never defer on teardown; sync happens inside `mlx5e_tx_wi_dma_unmap` before we free.

---

## Deferred array size

- Outer loop runs at most `MLX5E_TX_CQ_POLL_BUDGET` (128) CQEs. Each CQE can complete multiple WQEs (inner do-while). In the worst case each CQE completes one WQE, so 128 deferred entries. Using `MLX5E_TX_CQ_POLL_BUDGET * 2` (256) on the stack is safe and avoids overflow.
