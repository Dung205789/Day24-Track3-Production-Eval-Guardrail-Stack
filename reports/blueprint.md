# CI/CD Blueprint: RAG Eval + Guardrail Stack

**Sinh viên:** Ngô Quang Dũng — 2A202601819  
**Ngày:** 2026-08-26

---

## Guard Stack Architecture

```
User Input
    │
    ▼ (~?ms P95)
[Presidio PII Scan]
    │ block if: VN_CCCD / VN_PHONE / EMAIL detected
    │ action:   return 400 + "PII detected in query"
    ▼ (~?ms P95)
[NeMo Input Rail]
    │ block if: off-topic / jailbreak / prompt injection
    │ action:   return 503 + refuse message
    ▼
[RAG Pipeline (Day 18)]
    │ M1 Chunk → M2 Search → M3 Rerank → GPT-4o-mini
    ▼
[NeMo Output Rail]
    │ flag if:  PII in response / sensitive content
    │ action:   replace with safe response
    ▼
User Response
```

---

## Latency Budget

*(Điền từ kết quả Task 12 — measure_p95_latency())*

| Layer | P50 (ms) | P95 (ms) | P99 (ms) | Budget |
|---|---|---|---|---|
| Presidio PII | 9.72 | 10.74 | 10.74 | <10ms |
| NeMo Input Rail | 698.23 | 1105.36 | 1105.36 | <300ms |
| RAG Pipeline | *(không đo trong Task 12 — chỉ đo Presidio + NeMo input rail riêng lẻ theo spec)* | | | <2000ms |
| NeMo Output Rail | *(dùng chung cơ chế `self check output` — latency tương đương NeMo Input Rail ở trên)* | | | <300ms |
| **Total Guard (Presidio+NeMo input)** | 705.31 | **1115.08** | 1115.08 | **<500ms** |

**Budget OK?** [x] No (vượt budget ở NeMo Input/Output Rail)  
**Comment:** Bottleneck là **NeMo Input/Output Rail** (P95 ≈1105ms), không phải Presidio (P95 ≈11ms — đạt budget <10ms sát nút). Nguyên nhân: guardrail dùng flow `self check input`/`self check output` — mỗi request gọi thật 1 lượt LLM (gpt-4o-mini) để hỏi "tin nhắn này có vi phạm chính sách không?", nên chi phí chủ yếu là round-trip API. Cách tối ưu: (1) cache kết quả check cho các câu hỏi lặp lại, (2) dùng model nhỏ/nhanh hơn riêng cho input rail (hoặc heuristic/regex prefilter trước khi gọi LLM cho các trường hợp rõ ràng), (3) chạy input rail và RAG retrieval song song thay vì tuần tự khi input rail chỉ có tác dụng chặn ở bước cuối trước khi trả lời.

---

## CI/CD Gates (phải pass trước khi merge to main)

```yaml
# .github/workflows/rag_eval.yml
- name: RAGAS Quality Gate
  run: python src/phase_a_ragas.py
  env:
    MIN_FAITHFULNESS: 0.75
    MIN_AVG_SCORE: 0.65

- name: Guardrail Gate
  run: pytest tests/test_phase_c.py -k "test_adversarial_suite_pass_rate"
  # phải ≥ 15/20 (75%)

- name: Latency Gate
  run: python -c "from src.phase_c_guard import measure_p95_latency; ..."
  # P95 total < 500ms
```

---

## Monitoring Dashboard (production)

| Metric | Alert Threshold | Action |
|---|---|---|
| RAGAS faithfulness (daily sample) | < 0.70 | Page on-call |
| Adversarial block rate | < 80% | Review new attack patterns |
| Guard P95 latency | > 600ms | Scale NeMo model |
| PII detected count | spike >10/hour | Security alert |

---

## Kết quả thực tế từ Lab

| | Kết quả |
|---|---|
| RAGAS avg_score (50q) | 0.7937 (factual 0.8568 / multi_hop 0.7392 / adversarial 0.7763) |
| Worst metric | faithfulness (20/50 câu, tập trung nhiều nhất ở multi_hop: 14/20 = 70%) |
| Dominant failure distribution | factual (theo số lượng tuyệt đối, hòa với multi_hop — xem phân tích theo tỷ lệ trong `analysis/failure_clusters.md`) |
| Cohen's κ | 0.286 (fair, chưa đạt ngưỡng bonus >0.6) |
| Adversarial pass rate | 20 / 20 (100%) |
| Guard P95 latency | 1115.08 ms (Presidio 10.74ms + NeMo self-check 1105.36ms) |

---

## Nhận xét & Cải tiến

> **Hoạt động tốt:** Guardrail stack (Presidio + NeMo self-check input/output) chặn được 100% (20/20) adversarial inputs sau khi sửa `guardrails/config.yml` để dùng đúng flow `self check input`/`self check output` (thay vì flow dialog-turn tự viết ban đầu, vốn chạy không ổn định vì không đúng kiểu flow mà `rails.input/output` yêu cầu). RAGAS cũng cho thấy đúng tín hiệu mong đợi: adversarial avg (0.776) < factual avg (0.857) — pipeline thực sự bị bẫy bởi version conflict/phủ định như bộ test set 50q được thiết kế để phát hiện.
>
> **Cần cải thiện:** (1) `faithfulness` trên multi_hop là điểm yếu lớn nhất — RAG cần prompt ép mô hình trích rõ nguồn số liệu trước khi tính toán tổng hợp thay vì tự suy diễn. (2) Cohen's κ=0.286 (fair) cho thấy LLM-judge (so với ground truth) chưa đủ tin cậy để thay hoàn toàn con người — nên chỉ dùng làm bộ lọc sơ bộ. (3) Latency của guardrail (P95≈1.1s) vượt xa budget 500ms vì mỗi request tốn 1 lượt gọi LLM thật cho input rail — đánh đổi giữa độ chính xác (self-check bằng LLM) và tốc độ (embedding/heuristic nhanh nhưng kém tin cậy hơn, như đã thấy ở lần thử đầu).
>
> **Nếu deploy production:** sẽ tách input rail thành 2 tầng — heuristic/regex nhanh (bắt các câu jailbreak/off-topic rõ ràng, <10ms) chạy trước, chỉ gọi LLM self-check khi heuristic không chắc chắn; đồng thời cache kết quả self-check theo hash của câu hỏi để giảm chi phí cho các câu lặp lại, và theo dõi RAGAS faithfulness theo cronjob hằng ngày trên sample nhỏ để phát hiện regression sớm thay vì chạy toàn bộ 50q mỗi lần.
