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
| Faithfulness | Hơi dưới mục tiêu khi câu trả lời vẫn bám theo ngữ cảnh nhưng có một cụm nhỏ chưa được hỗ trợ, và cụm đó không làm đổi khuyến nghị chính. | Dưới 0.6 khi câu trả lời bịa ra chính sách, ngày tháng, số tiền hoặc ngoại lệ không có trong nguồn. | Siết chặt grounding, kiểm tra prompt/ngữ cảnh, và chặn deploy nếu có claim không được hỗ trợ. |
| Answer Relevance | Điểm thấp nhưng vẫn chấp nhận được trong các câu hỏi thăm dò hoặc trả lời chưa trọn vẹn, miễn là câu trả lời vẫn đúng chủ đề chính. | Dưới 0.6 khi câu trả lời bỏ qua ý định người dùng hoặc trả lời sang một chủ đề student-services khác. | Sửa routing/prompt và kiểm tra model có trả lời trực tiếp đúng câu hỏi hay không. |
| Context Recall | Hơi thấp nhưng vẫn chấp nhận được khi tập retrieved vẫn có evidence chính, chỉ thiếu một vài chi tiết phụ. | Dưới 0.6 khi retriever bỏ sót hoàn toàn evidence chính hoặc chỉ trả về các chunk nhiễu. | Cải thiện retrieval, chunking hoặc cách viết query trước khi chỉnh generation. |
| Context Precision | Điểm trung bình-thấp có thể tạm chấp nhận với câu hỏi multi-hop rộng nếu chunk liên quan vẫn có mặt và nhiễu không quá nhiều. | Dưới 0.6 khi phần lớn chunk truy xuất là không liên quan và evidence hữu ích bị chôn xuống dưới. | Re-rank, giảm nhiễu, hoặc tinh chỉnh retrieval để đưa evidence đúng lên sớm hơn. |
| Completeness | Hơi thấp nhưng vẫn chấp nhận được khi câu trả lời nêu đúng quy tắc chính nhưng thiếu một điều kiện phụ không làm đổi kết luận cốt lõi. | Dưới 0.6 khi thiếu ngày tháng, số tiền, ngoại lệ hoặc các bước tiếp theo quan trọng. | Mở rộng độ bao phủ câu trả lời và đối chiếu lại expected-answer với corpus. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*

Chạy cùng một câu hỏi với hai thứ tự hiển thị khác nhau: một lần cho response A đứng trước, một lần cho response B đứng trước, trong khi nội dung hai câu trả lời tương đương. Nếu điểm số thường xuyên ưu tiên câu trả lời ở vị trí đầu tiên qua nhiều lần thử, đó là dấu hiệu của position bias. Có thể làm phiên bản mạnh hơn bằng cách random thứ tự trên nhiều cặp so sánh rồi kiểm tra xem lựa chọn được hiển thị trước có thắng nhiều hơn ngẫu nhiên hay không.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*

Dùng rubric nhấn mạnh chất lượng nội dung thay vì độ dài: quy định rõ rằng câu dài hơn không tự động được điểm cao hơn nếu không thêm evidence đúng, điều kiện bắt buộc hoặc thông tin giúp câu trả lời đầy đủ hơn. Có thể phạt phần lặp lại, lan man, hoặc mở rộng không có căn cứ, và giới hạn điểm với câu trả lời dài nhưng vẫn thiếu ý. Câu ngắn gọn nhưng đủ thông tin quan trọng nên được chấm cao hơn câu dài có cùng nội dung.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*

Nhãn do con người gán giúp phát hiện lỗi trong rubric và các edge case đặc thù của domain mà LLM judge có thể bỏ sót hoặc đánh giá quá cao. Việc calibration giúp judge bám sát chất lượng mục tiêu thực tế, đo được bias hệ thống và đặt threshold có cơ sở khi dùng trong CI/CD. Nó cũng giúp tránh tin vào một judge lúc thì quá dễ dãi, lúc thì quá khắt khe, hoặc bị ảnh hưởng bởi cách diễn đạt thay vì tính đúng đắn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Các claim không được hỗ trợ rất rủi ro trong câu trả lời policy, nên điểm thấp hơn mức này thường báo hiệu hallucination cần chặn release. |
| Answer Relevance | 0.75 | Assistant phải trả lời đúng câu hỏi thực sự của sinh viên; điểm thấp thường cho thấy lỗi routing hoặc prompt. |
| Completeness | 0.75 | Thiếu ngày tháng, mức phí hoặc ngoại lệ có thể làm câu trả lời dù đúng vẫn không an toàn hoặc không dùng được. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*

Dùng offline evaluation cho mọi thay đổi về prompt, retriever hoặc model trước khi merge vì nó lặp lại được, rẻ và tốt cho việc phát hiện regression. Dùng online evaluation sau khi deploy để theo dõi traffic thật, drift và các lỗi bất ngờ. Dùng human review cho các quyết định rủi ro cao, các case mơ hồ hoặc để calibration chất lượng judge/rubric.

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
