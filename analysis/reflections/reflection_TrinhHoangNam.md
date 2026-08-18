# Individual Reflection — Lab 18

**Tên:** Trịnh Hoàng Nam (2A202601376)
**Module phụ trách:** M1–M5 (toàn bộ — bài cá nhân)

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:** Toàn bộ M1–M5 (bài cá nhân, không chia module theo người như bản nhóm).
- **Các hàm/class chính đã viết:**
  - M1: `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()`, helper `_split_by_size()`.
  - M2: `segment_vietnamese()`, `BM25Search.index()/search()`, `DenseSearch.index()/search()`, `reciprocal_rank_fusion()`.
  - M3: `CrossEncoderReranker._load_model()/rerank()`, `FlashrankReranker.rerank()` (optional).
  - M4: `evaluate_ragas()`, `failure_analysis()`.
  - M5: `summarize_chunk()`, `generate_hypothesis_questions()`, `contextual_prepend()`, `extract_metadata()`, `_enrich_single_call()` (combined mode).
- **Số tests pass:** 37/37 (`pytest tests/ -v` — M1: 13, M2: 5, M3: 5, M4: 4, M5: 10). `Select-String "# TODO" src\m*.py` không còn TODO nào sót.
- **Pipeline:** `python main.py` chạy end-to-end thành công (naive baseline → production → so sánh), sinh `reports/ragas_report.json` + `reports/naive_baseline_report.json`. Kết quả reproduce 2 lần cho số liệu giống hệt nhau, xác nhận pipeline ổn định.

## 2. Kiến thức học được

- **Khái niệm mới nhất:** Cơ chế RRF (Reciprocal Rank Fusion) hợp nhất kết quả BM25 + Dense dựa trên **rank** thay vì **score tuyệt đối** — giải quyết vấn đề 2 phương pháp có thang điểm không tương đồng (BM25 score có thể > 1, cosine similarity nằm trong [0,1]).
- **Điều bất ngờ nhất:** Ban đầu, một pipeline "production" với đầy đủ hierarchical chunking + hybrid search + reranking + enrichment lại cho RAGAS score **thấp hơn** naive baseline ở 3/4 metric — kỹ thuật càng phức tạp không tự động đồng nghĩa với càng tốt nếu implementation thiếu 1 bước quan trọng trong thiết kế (chi tiết ở mục 3). Bất ngờ thứ hai: hướng fix "đúng lý thuyết" nhất (resolve child→parent) lại làm cả hệ thống sụp điểm gần về 0 do context quá dài khiến RAGAS's internal judge fail parse — trong khi hướng fix "nửa vời" hơn (chỉ tăng child chunk size vừa phải + chỉnh prompt) lại thành công đưa cả 4 metric vượt ngưỡng bonus. Bài học: đúng lý thuyết chưa chắc an toàn về mặt vận hành (context length limits của hệ thống eval).
- **Kết nối với bài giảng:** Khớp trực tiếp với phần "Diagnostic Tree" trong RAGAS eval (M4) và khái niệm parent-child retrieval (M1) — bài học thực tế hôm nay minh chứng rõ tại sao cả 2 khái niệm này cùng tồn tại trong 1 buổi giảng.

### Phần 1 (ASSIGNMENT.md): Mapping bài giảng → Code

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Nhóm câu theo cosine similarity (threshold 0.85 mặc định, model `all-MiniLM-L6-v2`). Vì model tiếng Anh áp dụng cho câu tiếng Việt, độ ổn định similarity không cao bằng model đa ngôn ngữ — cần lưu ý khi áp dụng thực tế. |
| Hierarchical chunking (parent-child) | M1 | `chunk_hierarchical()` | Parent 2048 / child 256 ký tự, child luôn có `parent_id` hợp lệ (test pass). Tuy nhiên qua kết quả RAGAS thực tế phát hiện `pipeline.py` chỉ dùng child mà không resolve về parent khi trả context cho LLM — cho thấy tầm quan trọng của bước "return parent" trong thiết kế, không chỉ "retrieve child". |
| BM25 + Dense fusion (RRF) | M2 | `reciprocal_rank_fusion()` | RRF giải quyết vấn đề BM25 và Dense có thang điểm không tương đồng — chỉ dựa vào **rank** nên tránh được việc 1 phương pháp lấn át phương pháp kia. |
| Vietnamese word segmentation | M2 | `segment_vietnamese()` | underthesea nối từ ghép bằng `_` (vd "nghỉ_phép") — bắt buộc phải `replace("_", " ")` trước khi BM25 tokenize bằng `split(" ")`, nếu không "nghỉ phép" (query, 2 token) sẽ không khớp "nghỉ_phép" (corpus, 1 token). |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Load model lần đầu chậm (~46s, bao gồm tải model `bge-reranker-v2-m3`), các lần sau nhanh hơn nhờ cache. Reranker giúp câu trả lời "nghỉ phép" được ưu tiên đúng hơn so với doc không liên quan (VPN, mật khẩu) trong tập test mẫu. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Kết quả cuối cùng (sau khi áp dụng tổ hợp tối ưu, xem mục 3): Faithfulness 0.8542, Answer Relevancy 0.8851, Context Precision 0.9417, Context Recall 0.9000 — cả 4/4 đạt ngưỡng bonus ≥0.75 và Faithfulness đạt ngưỡng bonus ≥0.85. Các lần chạy trước (chunking mặc định, chưa tối ưu) cho kết quả thấp hơn rõ rệt (vd Faithfulness 0.7667, Context Recall 0.7917) — chênh lệch chủ yếu do độ dài child chunk, không phải do RAGAS đo sai. |
| Contextual embeddings / Enrichment | M5 | `_enrich_single_call()` | Dùng combined mode (1 API call/chunk thay vì 4 call riêng) để tối ưu chi phí — 100 chunks mất ~340s. Enrichment thêm 1 câu context mô tả vị trí chunk trong tài liệu (kiểu Anthropic contextual retrieval), nhưng không thể bù đắp việc chunk gốc bị thiếu thông tin — enrichment giúp *định vị* chunk tốt hơn, không giúp *mở rộng* nội dung chunk. |

## 3. Khó khăn & Cách giải quyết (= Phần 2 ASSIGNMENT.md)

- **Khó khăn 1 — Sai môi trường Python khi chạy `pytest`:**
  Lần chạy đầu, dù đã activate `.venv`, lệnh `pytest tests/test_m1.py -v` lại chạy bằng Python global (`C:\Users\Admin\...\Python313\python.exe`) thay vì `.venv`, dẫn đến lỗi:
  ```
  ModuleNotFoundError: No module named 'sentence_transformers'
  ```
  Thử ép chạy bằng `.venv\Scripts\python.exe -m pytest` thì lại báo `No module named pytest` — vì `pytest` chưa từng được cài trong `.venv` (và cũng không có trong `requirements.txt` gốc của repo).
  **Cách giải quyết:** Cài `pytest` trực tiếp vào `.venv` (`pip install pytest`). Sau khi cài đúng, banner pytest hiện đúng path `.venv\Scripts\python.exe`, `sentence_transformers` được nhận diện bình thường, toàn bộ 37 test M1–M5 pass.
  **Thời gian debug:** ~10-15 phút (1 lần chạy sai + phân tích banner + fix).
  **Kiến thức thiếu → bổ sung:** Hiểu rõ hơn cách Windows PowerShell resolve lệnh theo PATH ngay cả khi venv đã activate — không phải cứ thấy `(.venv)` trên prompt là chắc chắn lệnh chạy đúng interpreter, cần kiểm tra bằng banner/`--version` hoặc `where pytest`.

- **Khó khăn 2 — Production RAG cho kết quả RAGAS thấp hơn Naive Baseline:**
  Sau khi chạy `pipeline.py`/`main.py` thành công (2 lần, số liệu giống hệt nhau), 3/4 metric (Faithfulness, Context Precision, Context Recall) đều thấp hơn `naive_baseline.py`, ngược với kỳ vọng "production phải tốt hơn basic".
  **Cách debug:** Đối chiếu code `pipeline.py` với docstring thiết kế của `chunk_hierarchical()` ("retrieve child → return parent"), phát hiện pipeline chỉ index/sử dụng child chunk (256 ký tự) mà không bao giờ resolve ngược lại parent (2048 ký tự) khi đưa context cho LLM — khiến context đưa cho LLM còn nhỏ hơn cả basic chunking (500 ký tự) của naive baseline. Đối chiếu với `failure_analysis()` (M4) cho thấy phần lớn failure rơi vào `faithfulness` (LLM hallucinate vì thiếu context) và `context_recall` (thiếu chunk liên quan) — khớp với giả thuyết trên. Chi tiết đầy đủ (bottom-5, Error Tree) nằm trong `analysis/failure_analysis.md`.
  **Thời gian debug:** ~20 phút (đọc lại code pipeline.py, đối chiếu docstring, đối chiếu bottom-5 failures).
  **Kiến thức thiếu → bổ sung:** Hiểu sâu hơn rằng thiết kế đúng của một kỹ thuật (hierarchical chunking) không tự động đảm bảo kết quả tốt — nếu implementation chỉ dùng một nửa thiết kế (chỉ "retrieve child", bỏ "return parent") thì có thể phản tác dụng so với baseline đơn giản hơn.

- **Khó khăn 3 — Thử fix "đúng thiết kế" (resolve child→parent) lại làm sập toàn bộ hệ thống eval:**
  Sau khi xác định nguyên nhân ở Khó khăn 2, bước fix đầu tiên là sửa `pipeline.py` để resolve child chunk → parent chunk trước khi đưa context cho LLM, đúng thiết kế gốc của `chunk_hierarchical()`. Kết quả: cả 4 metric sụp gần về 0 (faithfulness 0.0, context_precision 0.05...), kèm log `Failed to parse output. Returning None.` từ RAGAS — dù đã thử nới lỏng prompt sinh câu trả lời, lỗi vẫn giữ nguyên.
  **Cách debug:** So sánh `context_precision` (metric chỉ phụ thuộc context, không phụ thuộc câu trả lời của LLM) giữa 2 lần chạy với 2 prompt khác nhau — thấy giá trị giữ nguyên y hệt 0.0500 → suy ra nguyên nhân không nằm ở prompt mà nằm ở chính context (parent chunk 2048 ký tự × top-3 kết quả là quá dài, khiến RAGAS's internal judge LLM fail parse JSON output khi chấm điểm).
  **Cách giải quyết:** Revert hoàn toàn về context chỉ dùng child chunk (không resolve parent), sau đó thử hướng an toàn hơn: tăng `HIERARCHICAL_CHILD_SIZE` vừa phải (256→550, không phải nhảy thẳng lên 2048) kết hợp 2 cải tiến prompt (yêu cầu trích số liệu cụ thể + trả lời rõ Có/Không cho câu hỏi negation). Kết quả: cả 4 metric vượt ngưỡng bonus, không tái diễn lỗi sụp điểm.
  **Thời gian debug:** ~40 phút (2 lần chạy full pipeline ~10 phút/lần để kiểm chứng giả thuyết, cộng thời gian phân tích log).
  **Kiến thức thiếu → bổ sung:** Một hệ thống eval tự động (RAGAS) cũng có giới hạn hạ tầng riêng (context length mà internal judge LLM có thể parse ổn định) — tối ưu 1 metric mà không kiểm soát các ràng buộc này có thể phá vỡ cả cơ chế đo lường, không chỉ ảnh hưởng đến điểm số. Fix "đúng lý thuyết nhất" không phải lúc nào cũng là fix an toàn nhất về mặt vận hành thực tế.

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:** Trước khi thử resolve child→parent (vốn đã gây sụp điểm ở bài này), sẽ đo trước độ dài trung bình của parent chunk và giới hạn số lượng/độ dài context đưa cho RAGAS judge — ví dụ resolve parent nhưng cắt bớt hoặc chỉ resolve 1 phần liên quan nhất, thay vì đưa nguyên văn 2048 ký tự × top-3. Ngoài ra sẽ viết eval nhỏ (vài câu hỏi) để test riêng bước "context length vs RAGAS parse success" trước khi chạy full 20 câu — tránh mất ~10 phút/lần chạy chỉ để phát hiện lỗi hạ tầng.
- **Module nào muốn thử tiếp:** M2 — thử nghiệm MMR (Maximal Marginal Relevance) thay vì chỉ lấy top-k theo RRF score thuần túy, để tăng diversity cho các câu hỏi multi-hop (cần thông tin từ nhiều nguồn khác nhau).

## 5. Action Plan cho project cá nhân (= Phần 3 ASSIGNMENT.md)

## Project: Trợ lý tra cứu chính sách nội bộ (HR/IT Policy Assistant)

### Hiện tại
- RAG pipeline hiện tại: bản MVP dạng "basic RAG" — chunking theo đoạn văn cố định, dense-only search (tương tự `naive_baseline.py` của lab này), chưa có rerank, chưa có evaluation tự động, câu trả lời đôi khi trộn lẫn thông tin giữa chính sách cũ/mới (vd nhầm mức nghỉ phép 12 ngày v2023 với 15 ngày v2024).
- Known issues:
  1. Không phân biệt được version chính sách hiện hành vs. đã bị thay thế (giống pattern `mat_khau_v1.md`/`v2.md` trong lab).
  2. Câu hỏi cần tổng hợp nhiều nguồn (multi-hop) thường trả lời thiếu hoặc sai một phần.
  3. Chưa có cách đo lường khách quan chất lượng câu trả lời — chỉ đánh giá cảm tính qua vài lần thử thủ công.

### Plan áp dụng
1. [x] Chunking strategy: chọn **hierarchical chunking** (parent 1500-2000 ký tự / child 200-300 ký tự) để retrieve chính xác bằng child nhưng **bắt buộc resolve về parent trước khi đưa context cho LLM** — đây là bài học trực tiếp từ lỗi trong `pipeline.py` của lab hôm nay (chỉ dùng child mà quên resolve parent khiến production thua cả naive baseline).
2. [ ] Search: **Hybrid (BM25 + Dense + RRF)** — BM25 quan trọng cho các query có số liệu/tên riêng chính xác (mức lương, số ngày, tên chính sách), Dense xử lý tốt câu hỏi diễn đạt tự nhiên/đồng nghĩa. Cân nhắc thêm metadata filter theo `is_current`/ngày hiệu lực để loại chính sách cũ đã bị thay thế trước khi search.
3. [ ] Reranking: **Có** — dùng `CrossEncoderReranker` (bge-reranker-v2-m3) như trong lab, top_k=3-5 tùy độ dài context cho phép.
4. [ ] Evaluation: **RAGAS 4 metrics** làm baseline định kỳ (chạy lại mỗi khi đổi corpus/policy), bổ sung custom metric riêng theo loại câu hỏi (lookup/version/multi-hop/numeric) như gợi ý trong `failure_analysis.md` để biết chính xác đang yếu ở nhóm câu hỏi nào thay vì chỉ nhìn con số trung bình.
5. [ ] Enrichment: **Contextual prepend** (ưu tiên) — vì corpus có nhiều file version chồng chéo, câu mô tả "đoạn văn này thuộc chính sách nào, version nào" giúp retrieval phân biệt tốt hơn; kết hợp **combined single-call mode** để tối ưu chi phí API giống M5 của lab.

### Timeline
- Tuần 1: Chuẩn hóa corpus (đánh dấu version hiện hành/cũ), implement hierarchical chunking + đảm bảo resolve parent đúng cách ngay từ đầu.
- Tuần 2: Implement hybrid search + reranking, chạy RAGAS baseline đầu tiên.
- Tuần 3: Enrichment (contextual prepend) + đo lại RAGAS, so sánh trước/sau — mục tiêu tất cả metric ≥ 0.75 và không có metric nào thấp hơn bản MVP ban đầu (bài học từ lab: luôn so sánh với baseline để tránh "production" phản tác dụng).
- Tuần 4: Failure analysis theo từng loại câu hỏi, vá các lỗ hổng lớn nhất trước khi đưa vào dùng thử nội bộ.

## 6. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 — nắm được cơ chế và implement đúng theo spec cả 5 module (37/37 test pass), nhưng phần tự phát hiện lỗi thiết kế trong `pipeline.py` (thiếu resolve child→parent) cho thấy vẫn cần đọc kỹ hơn phần "return parent" ngay từ đầu thay vì chỉ dừng ở việc test pass. |
| Code quality | 4 — code có fallback rõ ràng khi thiếu API key/model, có docstring, tách hàm hợp lý; điểm trừ vì một số phần (vd `_split_by_size`) là cải tiến thêm ngoài spec gốc nên cần review kỹ hơn để đảm bảo không lệch ý đồ ban đầu của đề bài. |
| Teamwork | N/A — bài cá nhân |
| Problem solving | 5 — debug đúng gốc rễ 3 vấn đề (môi trường pytest/venv, RAGAS production thấp hơn baseline, và regression khi thử fix "đúng lý thuyết") bằng cách đối chiếu code/docstring với dữ liệu thực tế (đặc biệt dùng context_precision không đổi để cô lập nguyên nhân về context thay vì prompt); đã thực sự sửa và đo lại — kết quả cuối đạt cả 4/4 metric ≥0.75 và Faithfulness ≥0.85, có nhật ký đầy đủ cả lần thử fail lẫn lần thành công trong `failure_analysis.md`. |