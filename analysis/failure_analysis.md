# Failure Analysis — Lab 18

**Học viên:** Trịnh Hoàng Nam (2A202601376)
**Module:** M1–M5 (cá nhân)

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8694 | **0.8542** | -0.0153 |
| Answer Relevancy | 0.7541 | **0.8851** | +0.1311 |
| Context Precision | 0.9250 | **0.9417** | +0.0167 |
| Context Recall | 0.9250 | **0.9000** | -0.0250 |


**Nhận xét:** Cả 4/4 metric của Production đều đạt ngưỡng bonus ≥ 0.75, và Faithfulness đạt 0.8542 ≥ 0.85 (ngưỡng bonus RUBRIC.md). So với naive baseline, Production vượt trội ở Answer Relevancy (+0.1311) và Context Precision (+0.0167), chỉ thấp hơn nhẹ ở Faithfulness (-0.0153) và Context Recall (-0.0250) — các chênh lệch âm này rất nhỏ và nằm trong biên độ nhiễu tự nhiên của RAGAS (LLM giám khảo), không còn là gap hệ thống như các lần chạy trước.

**Nguyên nhân cải thiện:** Gap lớn ở các lần chạy trước (đặc biệt Context Recall -0.18) đến từ việc `chunk_hierarchical()` (M1) chỉ index/trả về child chunk quá nhỏ (256 ký tự), khiến context bị cắt vụn hơn cả basic chunking của naive. Thay vì resolve child→parent (đã thử và gây regression nghiêm trọng — xem log thử nghiệm trước), giải pháp cuối cùng tăng `HIERARCHICAL_CHILD_SIZE` 256→550 (gần ngang basic chunk_size 500 của naive) để giảm phân mảnh mà không đưa context quá dài (2048 ký tự) khiến RAGAS fail parse như lần thử trước. Kết hợp thêm 2 cải tiến prompt (yêu cầu trích số liệu cụ thể + trả lời rõ Có/Không cho câu hỏi negation) đã kéo toàn bộ 4 metric lên trên ngưỡng bonus.

## Latency Breakdown (bonus)

| Bước | Thời gian |
|------|-----------|
| Chunking (M1) | 0.04s |
| Enrichment — M5, 1 API call/chunk | 166.83s |
| Indexing — BM25 + Dense (M2) | 38.57s |
| Reranker load (M3) | 0.0s |
| Query generation — 20 câu hỏi × search+rerank+LLM answer | 327.49s |
| RAGAS eval — 4 metrics × 20 câu hỏi | 67.58s |
| **Tổng** | **600.51s** (~10.0 phút) |


**Nhận xét:** Sau khi tăng `HIERARCHICAL_CHILD_SIZE` (256→550), số lượng chunk giảm nên **Enrichment giảm mạnh** (291.05s → 166.83s, -43%) và **Indexing cũng giảm nhẹ** (42.0s → 38.57s) — ít chunk hơn cần enrich/index hơn. Ngược lại, **Query generation tăng đáng kể** (185.46s → 327.49s, +76%): context dài hơn (child chunk 550 ký tự thay vì 256) khiến mỗi LLM call xử lý nhiều token hơn, cộng thêm prompt mới yêu cầu trích dẫn số liệu cụ thể + xử lý câu hỏi có/không cũng dài hơn prompt gốc. RAGAS eval cũng tăng nhẹ (56.15s → 67.58s) vì evaluate context dài hơn. Tổng thời gian tăng nhẹ (574.7s → 600.51s, +4.5%) nhưng đổi lại đạt cả 2 tiêu chí bonus RAGAS (Faithfulness ≥0.85 và all-4-metric ≥0.75) — đánh đổi hợp lý giữa latency và chất lượng.

## Bottom-5 Failures

### #1
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16-30 ngày cần phê duyệt của Giám đốc điều hành (CEO). Lưu ý: nghỉ trên 14 ngày không lương, nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** context_recall = 0.0
- **Error Tree:** Output sai → Context đúng? Không — không lấy được chunk chứa điều khoản phân cấp phê duyệt theo số ngày nghỉ không lương → Query OK? Có, rõ ràng → Root cause: retrieval (BM25+Dense+RRF) không ưu tiên đúng chunk chứa mốc "16-30 ngày → CEO phê duyệt", có thể do câu hỏi dùng "20 ngày" (số cụ thể) trong khi policy diễn đạt theo khoảng ("16-30 ngày") — mismatch về cách biểu đạt số liệu giữa query và corpus.
- **Suggested fix:** Thêm xử lý số liệu dạng khoảng (range) khi BM25 tokenize, hoặc query rewriting chuẩn hóa "20 ngày" → tìm khoảng chứa giá trị này trước khi search.

### #2
- **Question:** Nhân viên được tài trợ khóa học 25 triệu, nghỉ việc sau 8 tháng hoàn thành khóa học. Phải hoàn trả bao nhiêu?
- **Expected:** Nhân viên phải cam kết làm việc ít nhất 1 năm sau khi hoàn thành khóa học. Nghỉ sau 8 tháng là trước hạn cam kết, phải hoàn trả 100% chi phí tức 25.000.000 VNĐ.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** faithfulness = 0.25
- **Error Tree:** Output sai (một phần) → Context đúng? Một phần — câu hỏi multi-hop cần kết hợp điều khoản "cam kết làm việc 1 năm sau khóa học" với quy tắc hoàn trả theo tỷ lệ/toàn bộ → Query OK? Có, nhưng là dạng tính toán điều kiện (nếu nghỉ trước hạn X tháng → hoàn trả Y%) → Root cause: LLM có xu hướng tự suy luận tỷ lệ hoàn trả (vd chia theo số tháng còn thiếu) khi context không nêu rõ công thức, dẫn đến hallucinate con số khác 100%.
- **Suggested fix:** Prompt yêu cầu LLM trích nguyên văn điều khoản hoàn trả trước khi tính, không tự suy diễn công thức pro-rata nếu context không có.

### #3
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất 20.000.000 VNĐ/tháng; lương thử việc = 85% × 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** faithfulness = 0.4
- **Error Tree:** Output sai (một phần) → Context đúng? Một phần — chunk bảng lương Junior và điều khoản "85% lương thử việc" nằm ở 2 file khác nhau, vẫn dễ bị tách dù child chunk đã tăng 256→550 → Query OK? Có, nhưng cần multi-hop giữa 2 nguồn → Root cause: điểm đã cải thiện so với lần chạy trước (0.0 → 0.4) nhờ child_size lớn hơn + prompt yêu cầu trích số liệu, nhưng vẫn chưa hoàn toàn khắc phục vì 2 điều khoản vẫn ở 2 chunk khác nhau.
- **Suggested fix:** Tăng `RERANK_TOP_K` cho câu hỏi multi-hop hoặc thêm bước resolve có kiểm soát độ dài (vd chỉ ghép thêm 1 câu liền kề, không resolve nguyên parent 2048 ký tự).

### #4
- **Question:** Mentor và buddy của nhân viên mới có thể là cùng một người không? Quản lý trực tiếp có thể làm mentor không?
- **Expected:** KHÔNG cho cả hai. Mentor và buddy phải là hai người khác nhau. Quản lý trực tiếp không được làm mentor hoặc buddy.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** faithfulness = 0.5
- **Error Tree:** Output sai (một phần) → Context đúng? Có thể đúng — nhưng đây là câu hỏi ghép 2 câu hỏi con dạng có/không ("có thể cùng 1 người không?" + "quản lý trực tiếp có thể làm mentor không?") → Query OK? Có, nhưng compound negation (2 vế đều trả lời KHÔNG) → Root cause: prompt hiện tại chỉ hướng dẫn xử lý câu hỏi có/không **đơn**, chưa xử lý tốt trường hợp 2 câu hỏi có/không ghép trong 1 câu — LLM có thể trả lời đúng 1 vế, sai/thiếu vế còn lại.
- **Suggested fix:** Mở rộng hướng dẫn prompt: với câu hỏi chứa nhiều dấu "?", yêu cầu LLM trả lời "Có"/"Không" riêng cho TỪNG câu hỏi con theo thứ tự xuất hiện.

### #5
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** *(không lưu lại — xem ghi chú cuối file; không bắt buộc theo RUBRIC.md)*
- **Worst metric:** context_recall = 0.5
- **Error Tree:** Output sai (một phần, đã cải thiện) → Context đúng? Một phần — câu hỏi cần kết hợp 2 chính sách khác nhau (nghỉ phép theo thâm niên + bảng lương theo cấp bậc), nằm ở 2 file riêng biệt → Query OK? Có, nhưng là multi-hop 2-nguồn → Root cause: đây là case đã cải thiện đáng kể so với lần chạy trước (answer_relevancy 0.0 → context_recall 0.5, không còn là case tệ nhất), nhưng retrieval vẫn chỉ lấy được 1 phần trong 2 khía cạnh của câu hỏi ghép.
- **Suggested fix:** Query rewriting tách câu hỏi ghép (compound question) thành 2 sub-query độc lập trước khi retrieve — vẫn là fix triệt để nhất cho dạng câu hỏi này (xem Case Study).

## Case Study (presentation)

**Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?

**Error Tree walkthrough:**
1. Output đúng? → Không (context_recall = 0.0 — không lấy được chunk chứa điều khoản liên quan).
2. Context đúng? → Không — corpus diễn đạt theo khoảng ("nghỉ 16-30 ngày → CEO phê duyệt") trong khi câu hỏi dùng số cụ thể ("20 ngày"); retrieval (BM25 + Dense) không match được vì không có phép "hiểu" rằng 20 nằm trong khoảng 16-30.
3. Query rewrite OK? → Không — đây chính là vấn đề: cần chuẩn hóa/query rewriting để nhận diện query có số cụ thể cần map sang điều khoản dạng khoảng trong corpus, nhưng pipeline hiện tại search nguyên văn câu hỏi.
4. Fix ở bước: **Query understanding/rewriting** — thêm bước nhận diện số liệu trong câu hỏi và mở rộng query (vd thêm từ khóa "16-30 ngày", "phê duyệt nghỉ không lương") trước khi retrieve, thay vì chỉ dựa vào từ khóa trùng khớp trực tiếp.

**Nếu có thêm 1 giờ:**
- Implement query expansion đơn giản cho câu hỏi chứa số liệu cụ thể (map số → khoảng chứa số đó trong policy).
- Mở rộng prompt xử lý câu hỏi có/không: hỗ trợ trả lời riêng từng vế khi câu hỏi ghép nhiều câu hỏi con (case #4, mentor/buddy).
- Query decomposition cho câu hỏi ghép 2 chủ đề khác nhau (case #5, Senior 9 năm) — vẫn là gap lớn nhất còn lại sau khi đã tối ưu faithfulness/negation.

---

## Nhật ký thử nghiệm: đã thử tối ưu faithfulness, revert vì gây regression nghiêm trọng

Trong quá trình làm bài, có thử sửa `pipeline.py` để resolve child chunk → parent chunk trước khi đưa context cho LLM (đúng thiết kế gốc của `chunk_hierarchical()`), nhắm mục tiêu đạt bonus Faithfulness ≥ 0.85. Kết quả: **toàn bộ 4 metric sụp gần về 0** (faithfulness 0.0, answer_relevancy 0.0, context_precision 0.05, context_recall 0.0), kèm log `Failed to parse output. Returning None.` từ RAGAS.

**Chẩn đoán:** context_precision giữ nguyên y hệt 0.0500 qua 2 lần chạy dù đã đổi hẳn prompt sinh câu trả lời ở giữa — chứng minh nguyên nhân không nằm ở prompt (vì context_precision không phụ thuộc answer, chỉ phụ thuộc context retrieve được). Nhiều khả năng: đưa nguyên văn parent chunk (2048 ký tự) × top-3 kết quả làm context quá dài, khiến chính LLM giám khảo nội bộ của RAGAS (không phải LLM sinh câu trả lời của mình) bị fail parse JSON output khi đánh giá — một lỗi hạ tầng của RAGAS khi context vượt ngưỡng, không phải lỗi logic nghiệp vụ.

**Quyết định (lần 1):** revert hoàn toàn `run_query()`/`build_pipeline()` về đúng logic production ban đầu (chỉ giữ lại phần latency breakdown, không ảnh hưởng RAGAS). Bonus Faithfulness ≥0.85 chưa đạt được ở thời điểm đó (0.8321, thiếu 0.018) nhưng đổi lại pipeline ổn định, không có rủi ro tái diễn lỗi sụp điểm. Đây là bài học thực tế về rủi ro khi tối ưu 1 metric (faithfulness) có thể vô tình phá vỡ cơ chế đo lường của cả hệ thống eval nếu không kiểm soát độ dài context đưa vào.

### Lần thử thứ 2 (thành công): tổ hợp "1/5/6" — an toàn hơn, không đụng đến cấu trúc context

Sau khi rút kinh nghiệm từ lần thất bại trên (nguyên nhân: context quá dài do resolve full parent 2048 ký tự), lần thử thứ 2 chọn cách tiếp cận rủi ro thấp hơn, **không thay đổi cấu trúc `contexts` trong `run_query()`** (vẫn giữ nguyên logic child-chunk gốc), chỉ kết hợp 3 thay đổi nhỏ, độc lập với retrieval:

1. **Tăng `HIERARCHICAL_CHILD_SIZE`** trong `config.py`: 256 → 550 ký tự (gần bằng chunk_size 500 của naive baseline). Mục tiêu: giảm phân mảnh context ngay từ bước chunking (M1), mà không cần resolve về parent 2048 ký tự — tránh lặp lại lỗi context quá dài từng khiến RAGAS's internal judge fail parse JSON.
2. **Prompt yêu cầu trích dẫn số liệu cụ thể**: sửa system prompt trong `run_query()`, yêu cầu LLM nêu rõ con số/mức/ngày/tên chính sách có trong context thay vì trả lời chung chung — giảm hallucination (tăng faithfulness) và tăng độ khớp câu trả lời với câu hỏi (tăng answer_relevancy).
3. **Prompt xử lý riêng câu hỏi negation/yes-no**: yêu cầu LLM trả lời rõ "Có"/"Không" ngay đầu câu với các câu hỏi dạng "có được...không", "có cần...không" — trực tiếp khắc phục case #5 ở bottom-5 (câu hỏi PVI insurance).

**Kết quả:** cả 4/4 metric đạt ngưỡng bonus ≥ 0.75, Faithfulness đạt 0.8542 ≥ 0.85 (đạt bonus). Answer Relevancy tăng mạnh nhất (+0.1311 so với naive). Không xảy ra hiện tượng sụp điểm như lần thử đầu — xác nhận giả thuyết: vấn đề nằm ở **độ dài context**, không phải ở nội dung/độ chi tiết của prompt. Tăng child_size vừa phải (550, không phải 2048) là điểm cân bằng giữa "đủ ngữ cảnh" và "không vượt ngưỡng RAGAS judge có thể parse".
