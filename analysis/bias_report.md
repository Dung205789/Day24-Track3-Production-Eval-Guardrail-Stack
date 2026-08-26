# LLM Judge Bias Report — Phase B

**Sinh viên:** Ngô Quang Dũng — 2A202601819  
**Ngày:** 2026-08-26  
**Judge model:** gpt-4o-mini

Phương pháp: vì pipeline chỉ có 1 hệ thống RAG (không có 2 hệ thống để so sánh chéo), pairwise judge được chạy giữa **Answer A = câu trả lời của RAG pipeline** và **Answer B = ground_truth** trong `test_set_50q.json` / `human_labels_10q.json`. `final_winner = "A"` hoặc `"tie"` → coi model answer là đúng/đủ (judge_label=1); `final_winner = "B"` → ground truth rõ ràng tốt hơn → model answer sai/thiếu (judge_label=0). Toàn bộ dữ liệu thô nằm trong `reports/judge_results.json`.

---

## 1. Pairwise Judge Results

*(5 cặp mẫu đầu tiên trong `reports/judge_results.json`, A = RAG answer, B = ground_truth)*

| # | Question (tóm tắt) | Winner | Reasoning tóm tắt |
|---|---|---|---|
| 1 | Nghỉ bao nhiêu ngày khi kết hôn? | B | B (ground truth) nêu rõ "nghỉ có lương, không trừ phép năm" mà model answer thiếu |
| 2 | Mua thiết bị 55 triệu cần ai phê duyệt? | B | B nêu đúng ngưỡng phê duyệt >50 triệu; A (model) không nêu ngưỡng cụ thể |
| 3 | Thưởng Tết tối thiểu cho NV ≥6 tháng? | B | B đầy đủ hơn, có cả trường hợp NV <6 tháng |
| 4 | Senior 9 năm: phép năm + khoảng lương? | tie | Pass 1 chọn B, Pass 2 hòa → 2 pass không nhất quán → final=tie |
| 5 | Tài trợ khóa học 25tr, nghỉ sau 8 tháng, hoàn trả? | B | B chi tiết hơn về điều kiện cam kết và cách tính hoàn trả |

---

## 2. Swap-and-Average Results

| # | Pass 1 Winner | Pass 2 Winner | Final | Position Consistent? |
|---|---|---|---|---|
| 1 | B | B | B | True |
| 2 | B | B | B | True |
| 3 | B | B | B | True |
| 4 | B | tie | tie | **False** |
| 5 | B | B | B | True |
| 6 | B | B | B | True |
| 7 | B | B | B | True |
| 8 | B | B | B | True |
| 9 | B | A | **tie** | **False** |
| 10 | B | B | B | True |

*(10/15 case đầu tiên từ `reports/judge_results.json`; đủ 15 case đo trong bias_report bên dưới.)*

**Position bias rate:** 20% (3/15 case KHÔNG nhất quán giữa 2 pass — dưới ngưỡng 30% được coi là đáng lo ngại)

---

## 3. Cohen's κ Analysis

**Human labels:** `human_labels_10q.json` (10 câu, 5 label=1, 5 label=0)  
**Judge labels:** chạy `swap_and_average(question, model_answer, ground_truth)` trên đúng 10 câu này, quy đổi `final_winner` → 0/1 theo quy ước ở trên.

| Question ID | Human Label | Judge Label | Agree? |
|---|---|---|---|
| 1 | 1 | 0 | ✗ |
| 5 | 0 | 0 | ✓ |
| 12 | 1 | 0 | ✗ |
| 21 | 1 | 1 | ✓ |
| 23 | 1 | 0 | ✗ |
| 29 | 0 | 0 | ✓ |
| 33 | 1 | 0 | ✗ |
| 41 | 0 | 0 | ✓ |
| 46 | 1 | 1 | ✓ |
| 50 | 0 | 0 | ✓ |

**Cohen's κ:** 0.286  
**Interpretation:** fair (0.2–0.4 theo thang Landis-Koch) — **chưa đạt** ngưỡng bonus (>0.6, substantial).

---

## 4. Verbosity Bias

Trong 11 case có winner rõ ràng (không phải tie), trên tổng 15 case đo:
- A (RAG answer) thắng + A dài hơn B: 0 / 11 cases
- B (ground_truth) thắng + B dài hơn A: 11 / 11 cases
- **Verbosity bias rate:** 100%

**Kết luận:** LLM judge gần như luôn chọn `ground_truth` (B) — và ground_truth cũng luôn dài/chi tiết hơn model answer trong các case này, nên số liệu "verbosity bias" ở đây bị **nhiễu bởi confound**: judge có thể đang thắng vì B *đúng và đầy đủ hơn* (ground truth vốn được viết kỹ), chứ không hẳn vì B *dài hơn*. Đây là hạn chế của thiết kế "model answer vs ground truth" — không tách biệt được yếu tố độ dài khỏi yếu tố chính xác. Muốn đo verbosity bias "sạch" cần so sánh 2 answer có độ chính xác tương đương nhưng độ dài khác nhau (VD: 2 câu trả lời từ cùng 1 hệ RAG ở 2 lần chạy khác nhau, hoặc RAG vs naive baseline).

---

## 5. Nhận xét chung

> κ=0.286 (fair) — LLM judge (so với ground_truth) không đủ tin cậy để thay thế hoàn toàn đánh giá con người ở đây; phần lớn mismatch (Q1, Q12, Q23, Q33) là các case human_label=1 (đúng) nhưng judge lại chê thiếu chi tiết so với ground_truth — cho thấy judge có xu hướng khắt khe hơn con người khi so sánh trực tiếp với đáp án mẫu đầy đủ. Position bias 20% ở mức chấp nhận được (<30%), và swap-and-average có tác dụng thật: 2/3 case bất nhất quán (Q4, Q9 trong bảng mục 2, và câu hỏi mentor/buddy) được chuyển đúng thành "tie" thay vì chốt nhầm theo thứ tự xuất hiện — nếu chỉ chạy 1 pass sẽ có nguy cơ kết luận sai ở các case này. Trong production, nên dùng LLM-judge dạng này như một tín hiệu sàng lọc nhanh (flag câu trả lời khả nghi để review), không nên dùng làm ground-truth thay thế con người khi κ chưa vượt ngưỡng substantial (>0.6); nếu muốn cải thiện κ, nên đổi sang so sánh pairwise giữa 2 hệ thống RAG thực sự (không dùng ground truth làm 1 vế) hoặc cho judge chấm điểm theo rubric chi tiết thay vì chọn A/B/tie.
