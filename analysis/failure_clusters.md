# Failure Cluster Analysis — Phase A

**Sinh viên:** Ngô Quang Dũng — 2A202601819  
**Ngày:** 2026-08-26

---

## 1. Aggregate RAGAS Scores theo Distribution

| Metric | factual | multi_hop | adversarial |
|---|---|---|---|
| faithfulness | 0.8125 | 0.4783 | 0.8500 |
| answer_relevancy | 0.7649 | 0.6786 | 0.7133 |
| context_precision | 0.9750 | 0.9292 | 0.8917 |
| context_recall | 0.8750 | 0.8708 | 0.6500 |
| **avg_score** | **0.8568** | **0.7392** | **0.7763** |

(Nguồn: `reports/ragas_50q.json`, chạy trên 50 câu qua RAGAS: faithfulness, answer_relevancy, context_precision, context_recall.)

---

## 2. Bottom 10 Questions

| Rank | Distribution | Question | avg_score | worst_metric |
|---|---|---|---|---|
| 1 | factual | Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt? | 0.2500 | faithfulness |
| 2 | multi_hop | So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA. | 0.2500 | faithfulness |
| 3 | multi_hop | Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào? | 0.3750 | faithfulness |
| 4 | multi_hop | Nhân viên Manager có thâm niên 12 năm: tổng phụ cấp hàng tháng và số ngày phép năm theo v2024 là bao nhiêu? | 0.3750 | faithfulness |
| 5 | adversarial | Nhân viên Manager có thể dùng VPN cá nhân (như NordVPN) khi WFH để tăng bảo mật thêm không? | 0.4167 | faithfulness |
| 6 | factual | Nam nhân viên được nghỉ bao nhiêu ngày khi vợ sinh con? | 0.5000 | faithfulness |
| 7 | multi_hop | Nhân viên tạm ứng 4 triệu và một nhân viên khác tạm ứng 7 triệu: quy trình phê duyệt khác nhau thế nào? | 0.5000 | answer_relevancy |
| 8 | factual | Thông tin lương thuộc cấp độ phân loại dữ liệu nào? | 0.5929 | faithfulness |
| 9 | multi_hop | Nhân viên đi công tác trong nước 2 ngày, ở khách sạn giá 1.500.000 VNĐ/đêm. Công ty thanh toán tối đa bao nhiêu cho tiền khách sạn? | 0.6198 | faithfulness |
| 10 | adversarial | Theo chính sách nghỉ phép cũ (v2023), nhân viên được nghỉ bao nhiêu ngày? Còn chính sách nào đang có hiệu lực hiện tại? | 0.6307 | context_precision |

---

## 3. Failure Cluster Matrix

*(Mỗi ô = số câu có worst_metric = row, thuộc distribution = col)*

| worst_metric | factual | multi_hop | adversarial | Total |
|---|---|---|---|---|
| faithfulness | 5 | 14 | 1 | 20 |
| answer_relevancy | 11 | 4 | 1 | 16 |
| context_precision | 3 | 1 | 2 | 6 |
| context_recall | 1 | 1 | 6 | 8 |

---

## 4. Dominant Failure Analysis

**Dominant distribution (theo số lượng tuyệt đối):** factual (20 câu — hoà với multi_hop, cũng 20 câu; hàm `cluster_analysis()` chọn phần tử đầu tiên trong danh sách khi có hòa)
**Dominant metric:** faithfulness (20/50 câu, chiếm 40% tổng số failure)

**Lý do phân tích:**

> Vì `factual` và `multi_hop` đều có 20 câu (so với 10 của `adversarial`), so sánh theo **số lượng tuyệt đối** không phản ánh đúng độ nặng — cần nhìn theo **tỷ lệ trong từng nhóm**. Theo tỷ lệ, `multi_hop` mới là distribution có vấn đề nặng nhất: 14/20 câu (70%) có `faithfulness` là điểm yếu nhất, so với chỉ 5/20 (25%) ở `factual`. Điều này khớp với bản chất của multi_hop: câu hỏi đòi hỏi kết hợp số liệu từ ≥2 tài liệu (VD: tính lương + phép năm cùng lúc), nên LLM dễ tự suy diễn/nội suy con số khi ngữ cảnh retrieve không đủ đầy đủ cả hai vế — đây chính là hallucination mà `faithfulness` đo được. Ngược lại, `adversarial` lại yếu nhất ở `context_recall` (6/10 câu, 60%) chứ không phải faithfulness — hợp lý vì các câu adversarial cố tình hỏi về policy cũ (v2023) hoặc dùng phủ định, khiến pipeline khó retrieve đúng chunk chứa thông tin "còn hiệu lực hay không", dẫn đến thiếu context liên quan (recall thấp) hơn là bịa thông tin.

---

## 5. Suggested Fixes

| Metric yếu | Root cause | Suggested fix |
|---|---|---|
| faithfulness | LLM hallucinating khi phải tự tính/suy diễn từ nhiều số liệu rời rạc (đặc biệt ở multi_hop) | Tighten system prompt (yêu cầu liệt kê rõ nguồn số liệu trước khi tính), giảm temperature, thêm few-shot ví dụ tính toán multi-hop |
| context_recall | Thiếu chunk liên quan, đặc biệt khi câu hỏi ngầm định "văn bản đang có hiệu lực" (adversarial) | Thêm metadata `effective_status`/`version` khi chunk, kết hợp filter theo metadata hoặc tăng BM25_TOP_K/HYBRID_TOP_K cho câu có từ khóa version |
| context_precision | Nhiều chunk không liên quan lọt vào top-k (đặc biệt khi có nhiều version cùng chủ đề) | Thêm reranking mạnh hơn (đã có) + metadata filter loại bỏ version cũ khi câu hỏi không hỏi rõ về policy cũ |
| answer_relevancy | Answer không khớp trọng tâm câu hỏi (factual chiếm 11/20 case) | Cải thiện prompt template để bám sát đúng phần được hỏi (VD: câu hỏi hỏi "ai phê duyệt" nhưng answer chỉ liệt kê quy trình chung) |

---

## 6. Nhận xét về Adversarial Distribution

> `avg_score`: factual (0.8568) > adversarial (0.7763) > multi_hop (0.7392). Đúng như kỳ vọng của bộ test set (bonus criterion: adversarial < factual — **đạt**), pipeline bị "nhầm" đáng kể bởi các bẫy version conflict và phủ định, dù không tệ bằng multi_hop. Trong bottom 10 có 2 câu adversarial: #5 (VPN cá nhân — pipeline không nhận ra chính sách VPN v1.3 cấm dùng VPN cá nhân, faithfulness=điểm yếu) và #10 (hỏi về policy nghỉ phép v2023 cũ — context_precision là điểm yếu, tức là pipeline có retrieve nhưng lẫn cả version cũ/mới không phân biệt rõ). Điều này cho thấy vấn đề adversarial nằm nhiều ở tầng retrieval/metadata (chọn nhầm hoặc trộn version) hơn là tầng generation, khác với multi_hop (vấn đề chủ yếu ở generation/tính toán).
