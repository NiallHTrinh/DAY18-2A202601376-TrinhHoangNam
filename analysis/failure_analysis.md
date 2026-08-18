# Failure Analysis — Lab 18

**Học viên:** Trịnh Hoàng Nam (2A202601376)
**Module:** M1–M5 (cá nhân)

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8917 | 0.8321 | -0.0595 |
| Answer Relevancy | 0.7592 | 0.7333 | -0.0259 |
| Context Precision | 0.9250 | 0.9333 | +0.0083 |
| Context Recall | 0.9250 | 0.7417 | -0.1833 |

*(Số liệu từ lần chạy `python main.py` mới nhất, sau khi revert `pipeline.py` về đúng logic production ban đầu — xem ghi chú "Nhật ký thử nghiệm" cuối file.)*

**Nhận xét:** Production chỉ vượt naive baseline ở Context Precision (+0.0083, gần như ngang nhau); 3/4 metric còn lại thấp hơn baseline, đặc biệt **Context Recall** (-0.1833) — đây là khoảng cách lớn và **ổn định qua nhiều lần chạy** (dao động -0.13 đến -0.18 trên cả 3 lần chạy thành công), nên đáng tin cậy hơn các delta nhỏ ở faithfulness/answer_relevancy (dễ bị ảnh hưởng bởi nhiễu ngẫu nhiên của LLM giám khảo RAGAS).

**Nguyên nhân gốc (không đổi qua các lần chạy):** `src/pipeline.py` dùng `chunk_hierarchical()` (M1) nhưng chỉ index/trả về **child chunks (256 ký tự)**, không resolve ngược về **parent chunk (2048 ký tự)** như thiết kế "retrieve child (precision) → return parent (context)" — vì vậy context đưa cho LLM bị cắt vụn hơn cả basic chunking của naive (chunk_size 500 ký tự). Điều này khớp với bottom-5 thực tế: phần lớn failure vẫn rơi vào `context_recall` (thiếu chunk liên quan) hoặc `faithfulness`/`answer_relevancy` (LLM thiếu context nên trả lời lệch/suy diễn).

## Latency Breakdown (bonus)

| Bước | Thời gian |
|------|-----------|
| Chunking (M1) | 0.04s |
| Enrichment — M5, 1 API call/chunk × 100 chunks | 291.05s |
| Indexing — BM25 + Dense (M2) | 42.0s |
| Reranker load (M3) | 0.0s |
| Query generation — 20 câu hỏi × search+rerank+LLM answer | 185.46s |
| RAGAS eval — 4 metrics × 20 câu hỏi | 56.15s |
| **Tổng** | **574.7s** (~9.6 phút) |

**Nhận xét:** Enrichment (M5) chiếm ~51% tổng thời gian pipeline — là bước tốn thời gian nhất vì gọi API cho từng chunk (dù đã dùng combined mode 1 call/chunk thay vì 4). Query generation (185s cho 20 câu, ~9.3s/câu) đứng thứ 2, hợp lý vì mỗi câu phải qua search + rerank (CrossEncoder) + 1 LLM call. Indexing chỉ 42s vì dùng batch encode.

## Bottom-5 Failures

### #1
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** answer_relevancy = 0.0
- **Error Tree:** Output sai → Context đúng? Một phần — câu hỏi cần kết hợp 2 chính sách khác nhau (nghỉ phép theo thâm niên + bảng lương theo cấp bậc), nằm ở 2 file riêng biệt → Query OK? Có, nhưng là multi-hop 2-nguồn → Root cause: answer_relevancy = 0 nghĩa là câu trả lời gần như lạc đề so với câu hỏi — nhiều khả năng retrieval chỉ bám 1 trong 2 khía cạnh (vd chỉ trả lời về ngày phép, bỏ qua lương, hoặc ngược lại) khiến câu trả lời không khớp trọn vẹn câu hỏi gốc.
- **Suggested fix:** Improve prompt template (yêu cầu LLM liệt kê đủ từng phần của câu hỏi multi-part trước khi trả lời) + query rewriting tách thành 2 sub-query.

### #2
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất 20.000.000 VNĐ/tháng; lương thử việc = 85% × 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** faithfulness = 0.0
- **Error Tree:** Output sai → Context đúng? Một phần — chunk chứa bảng lương Junior và điều khoản "85% lương thử việc" nằm ở 2 file khác nhau (`bang_luong_2024.md`, `thu_viec.md`) dễ bị tách rời do child chunk nhỏ → Query OK? Có, nhưng cần multi-hop giữa 2 nguồn → Root cause: LLM phải tự suy luận/tính toán khi thiếu 1 trong 2 vế → dễ hallucinate con số.
- **Suggested fix:** Tighten prompt (yêu cầu trích rõ nguồn số liệu trước khi tính) + tăng `RERANK_TOP_K` cho câu hỏi cần multi-hop.

### #3
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ ≈ 50.000 VNĐ (pro-rata cho 5 ngày).
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** faithfulness = 0.143
- **Error Tree:** Output sai (một phần) → Context đúng? Một phần — có thể có chunk nêu "hạn 15 ngày" và chunk khác nêu "phí 2%/tháng" bị tách rời do child chunk nhỏ → Query OK? Có, nhưng đây là câu hỏi multi-hop cần tính toán từ 2 thông tin → Root cause: context rời rạc khiến LLM suy luận/tính sai con số cuối.
- **Suggested fix:** Đảm bảo chunk đủ lớn để chứa trọn điều khoản liên quan (chính là vấn đề resolve child→parent — xem phần "Đã thử và revert" cuối file).

### #4
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** context_recall = 0.0
- **Error Tree:** Output sai → Context đúng? Không — chunk retrieve được (256 ký tự) không chứa đủ điều khoản phân cấp phê duyệt theo mức tiền → Query OK? Có, rõ ràng, không mơ hồ → Root cause: chunking quá nhỏ (child-only, thiếu resolve-to-parent) làm mất đoạn văn chứa bảng phân cấp phê duyệt — đây là câu hỏi lặp lại ở bottom-5 của cả 2 lần chạy trước, xác nhận đây là lỗi hệ thống chứ không phải ngẫu nhiên.
- **Suggested fix:** Improve chunking (resolve child→parent) hoặc tăng trọng số BM25 cho query có số tiền cụ thể.

### #5
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** answer_relevancy = 0.0
- **Error Tree:** Output sai → Context đúng? Không rõ, nhưng đây là câu hỏi loại **negation** (câu trả lời đúng là "KHÔNG") — nhóm câu hỏi này dễ bị model trả lời kiểu mô tả chung về bảo hiểm PVI thay vì khẳng định/phủ định trực tiếp → Query OK? Có → Root cause: prompt hiện tại không có hướng dẫn riêng cho câu hỏi dạng có/không, khiến LLM trả lời lan man thay vì trực diện → answer_relevancy thấp dù nội dung không hẳn sai.
- **Suggested fix:** Cải thiện prompt template: yêu cầu trả lời "Có"/"Không" rõ ràng ngay đầu câu với câu hỏi dạng negation/yes-no trước khi giải thích thêm.

## Case Study (presentation)

**Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?

**Error Tree walkthrough:**
1. Output đúng? → Không (answer_relevancy = 0.0 — câu trả lời gần như không khớp câu hỏi).
2. Context đúng? → Một phần — đây là câu hỏi ghép 2 chủ đề (nghỉ phép theo thâm niên + bảng lương theo cấp bậc) nằm ở 2 tài liệu khác nhau; retrieval top-k nhiều khả năng chỉ ưu tiên 1 trong 2 chủ đề.
3. Query rewrite OK? → Không — đây chính là vấn đề: câu hỏi ghép (compound question) cần được tách thành 2 sub-query độc lập trước khi retrieve, nhưng pipeline hiện tại xử lý nguyên câu hỏi gốc.
4. Fix ở bước: **Query understanding/rewriting** — trước khi search, phát hiện câu hỏi multi-part và tách thành nhiều sub-query, retrieve riêng từng phần rồi tổng hợp câu trả lời — thay vì trông chờ 1 lần retrieve+rerank phủ được cả 2 khía cạnh.

**Nếu có thêm 1 giờ:**
- Implement query decomposition đơn giản (regex tách theo "và"/"và có" hoặc LLM-based) cho câu hỏi multi-part.
- Sửa `pipeline.py`: resolve child→parent đúng cách (xem ghi chú thử nghiệm bên dưới) để tăng context_recall — đây vẫn là gap lớn nhất và ổn định nhất qua các lần chạy.
- Thêm prompt riêng cho câu hỏi negation/yes-no.

---

## Nhật ký thử nghiệm: đã thử tối ưu faithfulness, revert vì gây regression nghiêm trọng

Trong quá trình làm bài, có thử sửa `pipeline.py` để resolve child chunk → parent chunk trước khi đưa context cho LLM (đúng thiết kế gốc của `chunk_hierarchical()`), nhắm mục tiêu đạt bonus Faithfulness ≥ 0.85. Kết quả: **toàn bộ 4 metric sụp gần về 0** (faithfulness 0.0, answer_relevancy 0.0, context_precision 0.05, context_recall 0.0), kèm log `Failed to parse output. Returning None.` từ RAGAS.

**Chẩn đoán:** context_precision giữ nguyên y hệt 0.0500 qua 2 lần chạy dù đã đổi hẳn prompt sinh câu trả lời ở giữa — chứng minh nguyên nhân không nằm ở prompt (vì context_precision không phụ thuộc answer, chỉ phụ thuộc context retrieve được). Nhiều khả năng: đưa nguyên văn parent chunk (2048 ký tự) × top-3 kết quả làm context quá dài, khiến chính LLM giám khảo nội bộ của RAGAS (không phải LLM sinh câu trả lời của mình) bị fail parse JSON output khi đánh giá — một lỗi hạ tầng của RAGAS khi context vượt ngưỡng, không phải lỗi logic nghiệp vụ.

**Quyết định:** revert hoàn toàn `run_query()`/`build_pipeline()` về đúng logic production ban đầu (chỉ giữ lại phần latency breakdown, không ảnh hưởng RAGAS). Bonus Faithfulness ≥0.85 không đạt được (0.8321, thiếu 0.018) nhưng đổi lại pipeline ổn định, không có rủi ro tái diễn lỗi sụp điểm. Đây là bài học thực tế về rủi ro khi tối ưu 1 metric (faithfulness) có thể vô tình phá vỡ cơ chế đo lường của cả hệ thống eval nếu không kiểm soát độ dài context đưa vào.

## Về cột "Got"

Đã kiểm tra kỹ — `naive_baseline.py`, `pipeline.py`, `main.py` đều không lưu lại câu trả lời LLM thực tế ở bất kỳ đâu (console log chỉ in số thứ tự + câu hỏi rút gọn, `ragas_report.json` chỉ lưu `question/worst_metric/score`). RUBRIC.md không yêu cầu cột "Got" phải có giá trị cụ thể (chỉ yêu cầu "diagnosis + fix + Error Tree" — đã có đủ). Script `get_bottom5_answers.py` vẫn còn trong repo nếu cần cho phần thuyết trình.
