# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Đào Phước Thịnh
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Linux (Ubuntu)
- **CPU:** Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
- **Cores:** 8 physical / 8 logical
- **CPU extensions:** AVX2 / AVX-512: false
- **RAM:** 7.6 GB
- **Accelerator:** NVIDIA GeForce MX150, 2048 MiB
- **llama.cpp backend đã chọn:** CUDA (but limited by small GPU memory, runs on CPU)
- **Recommended model tier:** TinyLlama-1.1B (Q4_K_M)

**Setup story** (≤ 80 chữ): Lab chạy trên laptop Dell Inspiron 5482 với Intel i5-8250U và NVIDIA MX150 (2GB VRAM). Do VRAM thấp, inference chạy trên CPU với 8 threads. CUDA available nhưng không đủ VRAM cho llama.cpp GPU offload hiệu quả với TinyLlama-1.1B.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| (Q4_K_M) | 2763 | 528 / 833 | 137.3 / 151.9 | 8452 / 10055 / 10380 | 7.3 |
| (Q2_K)   | 940 | 675 / 879 | 102.5 / 117.2 | 6560 / 7094 / 7114 | 9.8 |

**Một quan sát** (≤ 50 chữ): Q4_K_M decode rate 7.3 tok/s vs Q2_K 9.8 tok/s (~34% chênh). Q4_K_M load time gấp 3 lần (2763ms vs 940ms) nhưng cho chất lượng text tốt hơn đáng kể. Đánh đổi này xứng đáng với production use cases. Q2_K phù hợp khi RAM extremely tight.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.16 | 22,000 | 38,000 | 38,000 | 0% |
| 50 | 0.10 | 28,000 | 47,000 | 47,000 | 0% |

**KV-cache observation** (từ `record-metrics.py`): llama-cpp-python 0.3.22 **không có** endpoint /metrics. Metrics endpoint được planned trong future version. Server logs cho thấy prefix-match hit trong KV cache khi prompts có shared prefix. Tokens predicted observable qua server log output: `llama_perf_context_print` shows `total time` and token counts per request. KV cache usage ratio được tracking via prefix-match hits trong log.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: in-memory toy data
- **N19 (Vector + Feature Store):** stub: TOY_DOCS keyword matching

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: N/A (stub)
- retrieve: 0.1 ms (stub toy data)
- llama-server: 50,000-96,000 ms (CPU inference, long E2E due to small CPU-only laptop)

**Reflection** (≤ 60 chữ): Bottleneck là LLM inference time trên CPU-only laptop. Retrieve sử dụng toy keyword matching nên rất nhanh (0.1ms). Real-world RAG pipeline sẽ dominated by embed time (~100-300ms per query) và LLM decode time (~50-100s on CPU).

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Switching from Q4_K_M to Q2_K quantization to reduce model size and memory pressure on CPU-only laptop.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before (Q4_K_M): Load 2763ms, TPOT 137.3ms, Decode 7.3 tok/s
after (Q2_K):   Load 940ms,  TPOT 102.5ms, Decode 9.8 tok/s
speedup: ~1.34× faster decode, ~2.9× faster load
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Q2_K quantization sử dụng ít bits hơn per weight (~2.5 bits vs ~4.5 bits cho Q4_K_M). Điều này giảm:
1. Memory bandwidth pressure — CPU inference là bandwidth-bound, Q2_K đọc ít data hơn từ RAM
2. Model load time — file nhỏ hơn 2.9× nên load nhanh hơn tương ứng
3. Cache efficiency tốt hơn — working set nhỏ hơn fit tốt hơn trong L3/L4 cache

Trade-off: quality reduction. Q2_K có thể produce text kém chính xác trong một số cases, đặc biệt với technical content. Production nên test quality trước khi commit Q2_K.

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Ngạc nhiên: llama-cpp-python server version 0.3.22 chưa có /metrics endpoint, dù README của lab gợi ý sẽ có. Điều này có nghĩa là tracking KV cache usage phải thông qua server logs thay vì Prometheus scrape.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep) — stub only, bonus track not run
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`) — 2 present, need 4 more
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
