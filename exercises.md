# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Khi câu hỏi là Out-of-Scope / Adversarial và hệ thống chủ động trả lời từ chối ("Tôi không tìm thấy thông tin trong tài liệu") thay vì cố gắng trích dẫn context; hoặc khi thông tin trong context mâu thuẫn/thiếu sót. | Sinh viên hỏi về quy chế học tập, học phí, điều kiện tốt nghiệp mà LLM bịa đặt (hallucinate) ra quy định, con số hoặc mốc thời gian không có trong văn bản được cung cấp. | Siết chặt system prompt (ép quy tắc grounding "chỉ trả lời dựa trên context"), thêm hallucination detection guardrail, giảm `temperature=0`, bổ sung câu từ chối mẫu khi thiếu thông tin. |
| Answer Relevance | Khi câu hỏi của sinh viên quá mơ hồ/đa nghĩa (cần hỏi lại để làm rõ ý) hoặc khi câu hỏi vi phạm quy định/out-of-scope mà hệ thống phải trả lời từ chối lịch sự (dẫn đến token overlap với câu hỏi thấp). | Câu trả lời hoàn toàn lạc đề (ví dụ: sinh viên hỏi thời gian đăng ký môn học nhưng AI lại trả lời về quy định ký túc xá) hoặc đưa ra thông tin chung chung không giải quyết thắc mắc. | Cải thiện prompt (bổ sung Query Understanding / Intent Classification), fine-tune instruction-following, áp dụng kỹ thuật Query Rewriting / HyDE trước khi sinh câu trả lời. |
| Context Recall | Các câu hỏi tra cứu đơn giản (Factual Lookup) chỉ cần 1 thông tin duy nhất nằm trong 1 chunk là đủ trả lời chính xác, không nhất thiết phải lấy hết tất cả các văn bản liên quan rộng. | Câu hỏi phức tạp đòi hỏi tổng hợp từ nhiều văn bản (ví dụ: điều kiện xin hoãn thi + quy trình nộp đơn + lệ phí) nhưng Retriever bỏ sót các chunk chứa điều kiện quan trọng. | Nâng cấp Retriever (chuyển sang Hybrid Search: BM25 + Dense Vector Search), điều chỉnh chunk size/overlap, tăng `top_k` chunks lấy về, sử dụng Multi-query expansion. |
| Context Precision | Khi `top_k` lấy về nhiều chunks (ví dụ $k=10$), các chunk hữu ích nằm ở vị trí 2–3 thay vì vị trí 1, nhưng Generator vẫn lọc và tổng hợp đúng thông tin mà không bị xao nhãng. | Chunk chứa câu trả lời đúng bị đẩy xuống cuối (vị trí 4–5) trong khi các chunk đầu tiên toàn thông tin nhiễu (noise), gây ra lỗi "Lost in the Middle" cho LLM Generator. | Thêm bước Reranking (sử dụng Cross-Encoder Reranker hoặc Overlap Reranking) sau bước Retrieval để đẩy các chunk có độ tương quan cao nhất lên vị trí 1–2. |
| Completeness | Sinh viên chỉ xin câu trả lời xác nhận ngắn gọn (ví dụ: "Trường có ký túc xá không?" $\rightarrow$ "Có"), bỏ qua các chi tiết phụ khi không được yêu cầu giải thích chi tiết. | Câu hỏi yêu cầu quy trình nhiều bước (ví dụ: 5 bước xin miễn giảm học phí hoặc danh mục hồ sơ tốt nghiệp) nhưng LLM chỉ liệt kê 1–2 bước, bỏ sót các hạn chót hoặc hồ sơ quan trọng. | Thiết kế prompt yêu cầu liệt kê đầy đủ (structured output / bullet points), bổ sung checklist kiểm tra thông tin trong rubric hoặc áp dụng chain-of-thought prompting. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> 
> - **Thiết kế Thí nghiệm (Pairwise Evaluation):** Chuẩn bị tập $N$ câu hỏi cùng cặp câu trả lời từ hai hệ thống: $A$ (Response A) và $B$ (Response B).
>   - **Condition 1 (Order A-B):** Trình bày cho LLM Judge theo thứ tự: Candidate 1 = Response A, Candidate 2 = Response B. Yêu cầu Judge chọn câu trả lời tốt hơn hoặc chấm điểm.
>   - **Condition 2 (Order B-A):** Đổi thứ tự trình bày: Candidate 1 = Response B, Candidate 2 = Response A cho cùng câu hỏi và giữ nguyên rubric/prompt chấm điểm.
> - **Chỉ số đo lường & Phát hiện Bias:**
>   - Tính Tỷ lệ Thắng (Win Rate) của vị trí Candidate 1 ở cả hai lượt.
>   - Tính Tỷ lệ Mất nhất quán (Inconsistency Rate): Phân trăm trường hợp Judge chọn Candidate 1 ở Condition 1 nhưng lại chọn Candidate 1 ở Condition 2 (tức đảo ngược lựa chọn từ Response A sang Response B chỉ vì vị trí đứng trước).
>   - **Kết luận:** Nếu Win Rate của Candidate 1 lệch đáng kể ($> 55\%$) hoặc Inconsistency Rate cao, bằng chứng khẳng định LLM Judge mắc Position Bias. Giải pháp là áp dụng **Position Swapping & Averaging** (chấm cả 2 lượt lấy trung bình) hoặc chuyển sang **Single-Response Evaluation**.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 
> 1. **Quy định Tiêu chí Ngắn gọn / Trọng tâm (Conciseness Criterion):** Đưa định nghĩa rõ ràng vào Rubric (đặc biệt mức 5 điểm): *"Câu trả lời xuất sắc phải trực diện, đủ ý chính và không chứa từ ngữ thừa/dài dòng. Trừ điểm nếu chứa thông tin lan man hoặc lặp lại."*
> 2. **Đánh giá dựa trên Mật độ thông tin (Claim-based Evaluation):** Yêu cầu LLM Judge trích xuất danh sách ý chính (discrete claims) thay vì chấm cảm quan độ dài bài viết. Điểm số = số ý chính đúng chia cho tổng ý cần trả lời.
> 3. **Thêm Cảnh báo trực tiếp trong System Prompt của Judge:** Thêm chỉ dẫn nghiêm ngặt: *"DO NOT favor longer responses. A short 50-word accurate answer must receive a higher score than a 300-word response padded with repetitive or irrelevant information."*
> 4. **Cung cấp Reference Answer làm chuẩn độ dài:** Đưa expected answer chuẩn vào prompt để Judge so sánh độ phủ thông tin thay vì tự do thưởng điểm cho các phản hồi dài.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 
> 1. **Đảm bảo Alignment với Ground Truth của Domain:** LLM Judge dễ mắc các bias hệ thống (position, verbosity, self-preference) hoặc hiểu sai quy định học vụ đặc thù. Human labels từ chuyên gia/cán bộ nhà trường cung cấp chuẩn mực thực tế (Ground Truth).
> 2. **Đo lường độ tin cậy qua chỉ số đồng thuận (Inter-Annotator Agreement):** Cần tính toán chỉ số tương quan (như Cohen's Kappa $\kappa$, Krippendorff's Alpha, hoặc Pearson correlation $r$) giữa điểm của LLM Judge và Human Experts. Chỉ khi độ tương quan đạt mức cao ($\kappa \ge 0.75$ hoặc $r \ge 0.8$), LLM Judge mới đủ độ tin cậy để chạy tự động hóa trên quy mô lớn.
> 3. **Tối ưu hóa Prompt & Rubric cho Judge:** Quá trình calibration giúp phát hiện các sai sót thường gặp của Judge (ví dụ: quá dễ dắt đối với câu trả lời bịa đặt nhưng trôi chảy). Từ đó, người phát triển tinh chỉnh (iterate) prompt và rubric của Judge cho đến khi tỷ lệ sai lệch so với con người nằm trong ngưỡng an toàn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | `0.85` | Là metric sinh tử (critical) chống bịa đặt (hallucination). Trong trợ lý tư vấn sinh viên, trả lời sai quy chế/học phí/điểm số có thể gây hậu quả nghiêm trọng về học vụ. Cần threshold rất cao để block bất kỳ release nào gây ra hallucination. |
| Answer Relevance | `0.80` | Đảm bảo hệ thống trả lời đúng trọng tâm thắc mắc của sinh viên, không trả lời lan man hoặc lạc đề. Mức 0.80 cho phép dung sai nhỏ đối với các câu hỏi phức tạp hoặc câu trả lời từ chối lịch sự khi gặp câu hỏi out-of-scope. |
| Completeness | `0.75` | Đảm bảo cung cấp đủ các bước/điều kiện trong quy trình. Threshold này cho phép linh hoạt hơn Faithfulness vì một câu trả lời ngắn gọn nhưng đúng và đủ ý chính vẫn chấp nhận được mà không nhất thiết phải chép lại toàn bộ văn bản. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> 
> - **Offline Evaluation:**
>   - *Khi nào:* Chạy tự động trong CI/CD pipeline hoặc môi trường Staging mỗi khi có Pull Request, thay đổi code, cập nhật prompt, nâng cấp mô hình LLM hoặc điều chỉnh thuật toán retrieval. Chạy trên Golden Dataset cố định (ví dụ: 20–100+ QA pairs).
>   - *Mục đích:* Phát hiện lỗi thụt lùi (regression bugs), kiểm thử diện rộng nhanh chóng với chi phí cố định mà không ảnh hưởng tới sinh viên thật.
> - **Online Evaluation:**
>   - *Khi nào:* Chạy liên tục (continuous monitoring) trên môi trường Production với dữ liệu chat thực tế của sinh viên.
>   - *Mục đích:* Theo dõi biến động dữ liệu thực tế (data drift, query drift), phát hiện chủ đề câu hỏi mới chưa có trong golden dataset, giám sát latency, sentiment và phản hồi người dùng (thumbs up/down, user correction).
> - **Human Review:**
>   - *Khi nào:* 
>     1. Giai đoạn đầu xây dựng/calibrate LLM Judge và Golden Dataset.
>     2. Audit định kỳ theo mẫu ngẫu nhiên (ví dụ 1–5% log production).
>     3. Khi gặp các trường hợp rủi ro cao (high-stakes queries như khiếu nại điểm, kỷ luật, hoãn thi) hoặc khi mô hình trả về score có độ tự tin thấp (low-confidence).
>   - *Mục đích:* Cung cấp Ground Truth chính xác tuyệt đối, loại bỏ bias của LLM và phát hiện các lỗi tinh vi mà automated metrics bỏ sót.

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
