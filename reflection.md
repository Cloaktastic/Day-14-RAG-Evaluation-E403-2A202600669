# Day 14 — Reflection
## Evaluation Report & Failure Analysis

---

## 1. Benchmark Results Summary

Tóm tắt kết quả từ Exercise 3.2:

**Overall pass rate:** 25.00%

**Average scores:**

| Metric | Average | Min | Max | Std Dev |
|--------|---------|-----|-----|---------|
| Faithfulness | 0.64 | 0.00 | 1.00 | 0.28 |
| Relevance | 0.42 | 0.00 | 0.90 | 0.21 |
| Completeness | 0.56 | 0.00 | 1.00 | 0.35 |
| Overall Score | 0.54 | 0.00 | 0.88 | 0.22 |

**Score interpretation (theo bài giảng):**
- Bao nhiêu metrics ở Good (0.8–1.0)? 0
- Bao nhiêu metrics ở Needs Work (0.6–0.8)? 1 (Faithfulness: 0.64)
- Bao nhiêu metrics ở Significant Issues (<0.6)? 3 (Relevance: 0.42, Completeness: 0.56, Overall Score: 0.54)

**Failure type distribution:**

| Failure Type | Count | Percentage |
|--------------|-------|------------|
| hallucination | 3 | 15.0% |
| irrelevant | 4 | 20.0% |
| incomplete | 2 | 10.0% |
| off_topic | 6 | 30.0% |
| refusal | 0 | 0.0% |

---

## 2. Top 3 Worst Failures — 5 Whys Analysis

Theo bài giảng: "Phân loại failure TRƯỚC KHI fix. Đừng fix từng failure riêng lẻ — CLUSTER rồi fix root cause."

### Failure 1 (ID: M02)

**Question:** Nếu điểm GPA tích lũy (CGPA) dưới 2.0 thì sinh viên sẽ bị xử lý như thế nào?

**Agent Answer:** Sinh viên sẽ bị cảnh cáo học vụ nếu GPA tích lũy dưới 2.0.

**Scores:** Faithfulness: 0.00 | Relevance: 0.00 | Completeness: 0.00 | Overall: 0.00

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Điểm số của tất cả các metrics đánh giá đều bằng 0 (Passed = No). |
| Why 1 | Tại sao xảy ra? | Do câu trả lời sử dụng thuật ngữ "GPA tích lũy" thay vì "CGPA" như trong context, dẫn đến không khớp từ khóa. |
| Why 2 | Tại sao Why 1 xảy ra? | Heuristic đánh giá chỉ dựa trên so sánh tập hợp từ (word-overlap) thuần túy mà không xem xét ngữ nghĩa. |
| Why 3 | Tại sao Why 2 xảy ra? | Hệ thống đánh giá được lập trình đơn giản chỉ sử dụng phương pháp tokenization cơ bản để loại bỏ stopword. |
| Why 4 | Root cause là gì? | Phương pháp đo lường lexical overlap thô sơ thất bại khi gặp từ đồng nghĩa tiếng Việt, không phản ánh đúng ngữ nghĩa. |

**Root cause (from `find_root_cause()`):**
> `Multiple issues detected — review full pipeline` (Do cả 3 điểm đều sập về 0.0 nên thuật toán xác định là tie).

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý một phần. Vì các điểm số đều bằng 0 nên bị coi là lỗi hệ thống toàn diện. Thực chất đây là lỗi của phương thức đo lường (word overlap) không xử lý được từ đồng nghĩa tiếng Việt.

**Proposed fix (cụ thể, actionable):**
> - Thay thế word overlap heuristic bằng các metric đánh giá ngữ nghĩa chuyên dụng như BERTScore hoặc Sentence Transformers Similarity.
> - Bổ sung tài liệu từ đồng nghĩa hoặc hướng dẫn LLM Generator thống nhất thuật ngữ viết tắt của trường.

---

### Failure 2 (ID: A03)

**Question:** Điểm số của kỳ thi kiểm tra đầu vào tại Đại học Fulbright Việt Nam được tính như thế nào?

**Agent Answer:** Đại học Fulbright Việt Nam tính điểm số kỳ thi kiểm tra đầu vào dựa trên bài luận và phỏng vấn cá nhân.

**Scores:** Faithfulness: 0.09 | Relevance: 0.70 | Completeness: 0.12 | Overall: 0.30

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Điểm Faithfulness và Completeness cực kỳ thấp, câu trả lời bị xếp vào nhóm bịa đặt thông tin. |
| Why 1 | Tại sao xảy ra? | Agent cố trả lời về quy chế của một đại học khác (Fulbright) không hề có trong cơ sở tri thức của RAG. |
| Why 2 | Tại sao Why 1 xảy ra? | Agent không phát hiện ra câu hỏi này là nằm ngoài phạm vi hoạt động của hệ thống (out-of-scope). |
| Why 3 | Tại sao Why 2 xảy ra? | Hệ thống chưa có bộ lọc intent hoặc cơ chế kiểm soát đầu vào ở tầng Gateway. |
| Why 4 | Root cause là gì? | Thiếu bộ phân loại ý định (Query Intent Classifier) để chặn và từ chối các câu hỏi nằm ngoài phạm vi VinUni. |

**Root cause:**
> `Context is missing or irrelevant — improve retrieval` (Do Faithfulness 0.09 là điểm thấp nhất).

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý. Do context nạp vào không chứa dữ liệu về Đại học Fulbright nên Retriever không lấy được thông tin đúng, dẫn đến việc Generator bịa đặt câu trả lời.

**Proposed fix:**
> - Triển khai một Gateway Intent Classifier sử dụng Few-shot learning để phát hiện câu hỏi ngoài lề (out-of-scope) và trả về câu trả lời từ chối mẫu.
> - Thêm ràng buộc trong system prompt để LLM từ chối trả lời nếu độ tương thích tài liệu lấy ra thấp.

---

### Failure 3 (ID: M07)

**Question:** Sinh viên nghỉ ốm làm thế nào để được chấp nhận nghỉ học có phép cho buổi kiểm tra giữa kỳ?

**Agent Answer:** Sinh viên nghỉ ốm cần nộp đơn xin nghỉ và thi bù lên văn phòng học vụ.

**Scores:** Faithfulness: 0.38 | Relevance: 0.25 | Completeness: 0.29 | Overall: 0.31

**5 Whys Analysis:**
| Level | Question | Answer |
|-------|----------|--------|
| Symptom | Vấn đề là gì? | Điểm Relevance và Completeness rất thấp, lỗi phân loại là `irrelevant`. |
| Why 1 | Tại sao xảy ra? | Agent bỏ sót các chi tiết điều kiện hành chính cốt lõi (như nộp trong 3 ngày làm việc, giấy khám của bệnh viện). |
| Why 2 | Tại sao Why 1 xảy ra? | LLM Generator bị xu hướng khái quát hóa quá mức, trả lời chung chung cho nhanh. |
| Why 3 | Tại sao Why 2 xảy ra? | Prompt của Generator không có chỉ thị bắt buộc trích xuất đầy đủ các chi tiết định lượng (thời gian, mốc ngày). |
| Why 4 | Root cause là gì? | Thiếu các ràng buộc về tính đầy đủ và chính xác số liệu hành chính trong Generator system prompt. |

**Root cause:**
> `Answer does not address the question — improve prompt clarity` (Do Relevance 0.25 là thấp nhất).

**Bạn có đồng ý với root cause suggestion không? Tại sao?**
> Đồng ý một phần. Gợi ý chỉ ra cần cải thiện prompt clarity của generator để bám sát câu hỏi hơn, giúp đưa ra câu trả lời chứa đầy đủ các khía cạnh hành chính bắt buộc.

**Proposed fix:**
> - Cập nhật system prompt của Generator với định dạng đầu ra bắt buộc phải liệt kê rõ: (1) Hồ sơ yêu cầu, (2) Thời hạn thực hiện, (3) Địa điểm/Email tiếp nhận.

---

## 3. Failure Clustering

Theo bài giảng: "Fix 1 root cause giải quyết nhiều failures cùng lúc."

**Cluster Analysis:**

| Cluster | Root Cause | Failures in cluster | Priority |
|---------|-----------|--------------------:|----------|
| 1 | Thiếu bộ lọc câu hỏi ngoài phạm vi (Out-of-scope) | A01, A02, A03 | High |
| 2 | Generator tóm tắt quá mức, thiếu mốc thời gian/giấy tờ cụ thể | M01, M02, M03, M04, M05, M06, M07 | High |
| 3 | Khác biệt từ ngữ đồng nghĩa tiếng Việt (GPA tích lũy vs CGPA) | E02, H01, H02, H03, H04, H05 | Medium |

**Nếu chỉ fix 1 cluster, bạn chọn cluster nào? Tại sao?**
> Tôi chọn **Cluster 2 (Generator tóm tắt quá mức, thiếu chi tiết định lượng)**. Đây là nhóm lỗi chiếm tỷ trọng lớn nhất (7/15 failures). Bằng việc tinh chỉnh prompt yêu cầu trích xuất toàn bộ điều kiện và mốc thời gian hành chính, ta có thể cải thiện đáng kể điểm Completeness và Relevance của đa phần các câu hỏi nghiệp vụ thực tế.

---

## 4. Improvement Log (from `generate_improvement_log`)

Đầu ra của `generate_improvement_log()`:

```
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Answer is missing key information — increase context window or improve generation | Enforce strict grounding in prompt (e.g., 'only use the context provided') | Open |
| F003 | hallucination | Multiple issues detected — review full pipeline | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F004 | off_topic | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers to improve completeness | Open |
| F005 | irrelevant | Answer does not address the question — improve prompt clarity | Refine system prompt and routing mechanism to improve question relevancy | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Use query expansion or HyDE to retrieve more relevant documents | Open |
| F007 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F008 | incomplete | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F009 | off_topic | Answer does not address the question — improve prompt clarity | N/A | Open |
| F010 | incomplete | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F011 | irrelevant | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F012 | off_topic | Answer is missing key information — increase context window or improve generation | N/A | Open |
| F013 | irrelevant | Answer does not address the question — improve prompt clarity | N/A | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | N/A | Open |
```

**Thêm 3 improvement suggestions từ `generate_improvement_suggestions()`:**
1. Implement hallucination checker to filter unsupported claims
2. Enforce strict grounding in prompt (e.g., 'only use the context provided')
3. Increase chunk size in RAG pipeline to reduce context fragmentation

---

## 5. Regression Testing Strategy

### CI/CD Integration

**Câu 1: Khi nào chạy `run_regression()` trong production system?**
> Tự động trigger trong CI/CD pipeline khi:
> 1. Thay đổi code của hệ thống (Retriever hoặc Generator logic).
> 2. Có sự chỉnh sửa hoặc cập nhật System Prompt.
> 3. Cập nhật dữ liệu tài liệu mới vào cơ sở tri thức (Knowledge Base).
> 4. Nâng cấp phiên bản mô hình LLM nền tảng.

**Câu 2: Threshold regression 0.05 có phù hợp domain của bạn không?**
> Không phù hợp. Đối với quy chế học vụ và tuyển sinh đại học (liên quan trực tiếp đến tài chính, tốt nghiệp, kỷ luật), sai số 0.05 là quá lỏng lẻo. Cần đặt threshold khắt khe hơn: `0.02` cho Relevance/Completeness, và bắt buộc phải là `0.00` (không chấp nhận bất cứ sự sụt giảm nào) đối với Faithfulness để tránh hoàn toàn hallucination.

**Câu 3: Khi phát hiện regression — block deployment hay chỉ alert?**
> **Bắt buộc block deployment**. Đưa thông tin sai lệch về quy chế học viện lên production sẽ gây hậu quả pháp lý và nhầm lẫn nghiêm trọng cho sinh viên. Trade-off là có thể làm chậm tốc độ release khi có thay đổi nhỏ, nhưng đảm bảo an toàn tuyệt đối.

**Câu 4: Eval pipeline nên chạy ở đâu trong CI/CD flow?**

```
Code change → [Chạy unit test logic code] → [Chạy benchmark eval 20 QA] → [Kiểm tra regression & threshold] → Deploy
                 (bước 1)                   (bước 2)                       (bước 3)
```

---

## 6. Continuous Improvement Loop

Theo bài giảng: Evaluate → Analyze → Improve → Augment (add to benchmark) → lặp lại

**Sau lab hôm nay, 3 actions tiếp theo bạn sẽ làm để improve agent:**

| Priority | Action | Metric sẽ improve | Expected impact |
|----------|--------|-------------------|-----------------|
| 1 | Triển khai Gateway Intent Classifier để lọc câu hỏi ngoài phạm vi (out-of-scope). | Relevance, Safety | Loại bỏ triệt để các lỗi lạc đề và tấn công prompt injection. |
| 2 | Chỉnh sửa Generator system prompt, ép định dạng đầu ra phải liệt kê đủ các mốc thời gian và hồ sơ. | Completeness | Cải thiện mạnh mẽ chất lượng thông tin học vụ được cung cấp. |
| 3 | Tích hợp Reranker (BGE-Reranker) để chọn lọc và sắp xếp lại tài liệu. | Context Precision | Đưa các tài liệu chính xác nhất lên đầu, giảm nhiễu thông tin. |

**Bạn sẽ thêm failure cases nào vào benchmark cho sprint tiếp theo?**
> - Câu hỏi so sánh trực tiếp quy chế qua các năm học khác nhau (ví dụ: Quy định tốt nghiệp năm 2024 có gì khác năm 2025?).
> - Các câu hỏi sử dụng từ lóng/viết tắt của sinh viên (như "rút môn", "học bổng học kỳ", "nợ môn") để kiểm tra tính linh hoạt của retriever.

---

## 7. Framework Reflection

**Framework bạn đã dùng trong lab:** Custom Heuristic Evaluator (RAGAS-inspired)

**Nếu dùng trong production, bạn sẽ chọn framework nào? Tại sao?**
> Tôi chọn **DeepEval**.

| Tiêu chí | Lý do chọn |
|----------|------------|
| Focus phù hợp vì... | Hỗ trợ Unit testing rất tốt thông qua cơ chế test case assertions tựa pytest, giúp phát hiện lỗi cực nhanh ngay trong quá trình coding. |
| CI/CD integration vì... | CLI hỗ trợ tích hợp sâu với GitHub Actions, tự động xuất báo cáo HTML trực quan cho mỗi lần pull request. |
| Team workflow vì... | Có cổng dashboard trực quan (Confident AI) giúp lưu trữ lịch sử chấm điểm, thống kê failure logs giúp team phân tích lỗi dễ dàng. |
