# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 20.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.945 | 0.609 | 1.000 | Retrievable evidence rất tốt, coverage cao. |
| Context Precision | 0.948 | 0.589 | 1.000 | Ranking nhìn chung ổn, noise không phải vấn đề chính. |
| Faithfulness | 0.507 | 0.000 | 1.000 | Generation đôi lúc bịa hoặc trả lời dở dang. |
| Relevance | 0.355 | 0.000 | 0.818 | Nhiều câu lệch intent hoặc chỉ trả lời một phần. |
| Completeness | 0.303 | 0.000 | 1.000 | Metric yếu nhất, nhiều answer thiếu ý cốt lõi. |
| Overall Score | 0.441 | 0.000 | 0.939 | Điểm tổng thấp chủ yếu do answer-side yếu. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): E01, E02, E03, E04, H02
- Metrics/cases ở mức Needs Work (0.6–0.8): A03, A02, M05, H05
- Metrics/cases ở mức Significant Issues (<0.6): E05, M01, M02, M03, M04, M06, M07, H01, H03, H04, A01

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 7 | 35.0% |
| irrelevant | 4 | 20.0% |
| incomplete | 4 | 20.0% |
| off_topic | 1 | 5.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở generation hơn là retrieval. Context Recall và
Context Precision đều rất cao, nên retriever đã lấy được evidence. Nhưng
Faithfulness, Relevance và Completeness thấp hơn rõ rệt, cho thấy model trả lời
thiếu ý, lệch intent hoặc sinh ra đoạn output rỗng/cụt.

> *Câu trả lời:*

Retrieval là điểm mạnh của pipeline, còn generation là nút thắt chính. Hai chỉ
số retrieval đều quanh 0.95, nhưng answer-side metrics thấp hơn nhiều, đặc biệt
là Completeness và Relevance. Điều này khớp với các case như M01, M02 và M06:
evidence đã có, nhưng câu trả lời sinh ra bị cụt, rỗng hoặc lệch câu hỏi.

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:**

> `M01` - If a student drops a course before the census date, how do tuition and scholarship consequences work?

**Expected answer:**

> Dropping a course before or on the census date may change the student's billed credits and scholarship status. For a course dropped by the end of standard add/drop, 100% of that course's tuition is reversed; from the day after standard add/drop through the census date, 50% is reversed.

**Actual answer:**

> Based on the provided contexts, the

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Retriever lấy đúng evidence chính từ `01_academic_calendar.md`,
`03_tuition_payment_refund.md`, và `04_scholarships.md`. Không thiếu context; vấn đề là
answer bị cắt cụt ngay đầu câu.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Answer bị dở dang và không có nội dung hữu ích để trả lời câu hỏi. |
| Why 1 | Tại sao symptom xảy ra? | Generator trả về một câu mở đầu dở dang thay vì một câu trả lời hoàn chỉnh. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt/decoding chưa buộc model hoàn tất câu trả lời theo cấu trúc rõ ràng. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có post-generation validation kiểm tra độ đầy đủ trước khi chấm pass/fail. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Benchmark chỉ chấm bằng overlap heuristic nên không có guardrail riêng cho output truncated. |
| Why 5 | Root cause có thể hành động được là gì? | Cần tăng kiểm tra hoàn chỉnh câu trả lời và bắt buộc template nêu đủ các điều kiện. |

**Root cause từ `find_root_cause()`:**

> Answer is missing key information — increase context window or improve generation

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Đồng ý một phần. Trace cho thấy context đầy đủ, nên root cause chính là incomplete
generation. Tuy nhiên fix không chỉ là tăng context window, mà còn phải thêm
format guardrail và validation output để tránh answer bị cụt.

**Proposed fix cụ thể:**

> Thêm output validation cho câu trả lời quá ngắn/cụt, retry khi answer không chứa
ít nhất các thành phần bắt buộc, và dùng prompt template ép model nêu rule + fee +
deadline + exception nếu có.

### Failure 2

**ID và question:**

> `M02` - What conditions must be met for a late add in registration version 2.0?

**Expected answer:**

> A late add requires instructor approval, programme-director approval, and payment of a USD 40 late-add fee per course within two business days of approval.

**Actual answer:**

> concise/accurate). 3. **Synthesize the

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 |
Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:**

Retriever đã mang đúng policy version 2.0 và late-add rules từ `02_course_registration.md`
và `03_tuition_payment_refund.md`. Output lại là mảnh text giống prompt hướng dẫn, không
phải câu trả lời.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Output là đoạn rời rạc giống text từ prompt, không trả lời câu hỏi late add. |
| Why 1 | Tại sao symptom xảy ra? | Model sinh nhầm phần hướng dẫn thay vì đáp án. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Prompt generation chưa đủ cứng để tách instruction khỏi answer content. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có kiểm tra format/cấu trúc đầu ra trước khi lưu artifact. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống chỉ xem text output là hợp lệ miễn là không lỗi runtime. |
| Why 5 | Root cause có thể hành động được là gì? | Cần làm sạch prompt, thêm output guardrails, và retry khi text đầu ra không phải câu trả lời hoàn chỉnh. |

**Root cause và proposed fix:**

> Root cause là generation/prompt contamination. Cần ràng buộc format response,
thêm kiểm tra "answer phải nêu approvals + fee + deadline", và retry khi output
không đạt chuẩn.

### Failure 3

**ID và question:**

> `M06` - How does an approved medical leave affect scholarship and how long can it pause the award?

**Expected answer:**

> An approved medical leave pauses the scholarship for up to two consecutive regular terms and does not consume the one-term probation opportunity.

**Actual answer:**

> *Duration:* It can pause the

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.000 |
Relevance: 0.182 | Completeness: 0.000 | Overall: 0.061

**Evidence inspection:**

Evidence đầy đủ trong `04_scholarships.md` và `06_leave_and_withdrawal.md`, nhưng answer
bị cắt cụt ở giữa câu. Đây là failure generation rất rõ.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Answer bị dở dang và không nêu đủ nội dung về thời hạn pause scholarship. |
| Why 1 | Tại sao symptom xảy ra? | Generation bị ngắt giữa chừng hoặc không hoàn tất câu. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Token budget/format prompt chưa đủ mạnh để ép câu trả lời hoàn chỉnh. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có kiểm tra độ đầy đủ trước khi ghi artifact. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống chỉ xem text output là hợp lệ miễn là không lỗi runtime. |
| Why 5 | Root cause có thể hành động được là gì? | Tăng output validation, retry khi answer quá ngắn, và dùng prompt template bắt buộc trả lời đủ điều kiện. |

**Root cause và proposed fix:**

> Root cause là incomplete generation do output bị cắt cụt. Fix ưu tiên là thêm
validation cho câu trả lời quá ngắn hoặc đứt đoạn, rồi retry với prompt chặt hơn.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Output truncation / incomplete generation | M01, M06, M04, M05, H02, H03 | High |
| 2 | Prompt contamination / bad formatting | M02, H01, H04 | High |
| 3 | Intent mismatch / weak answer targeting | M07, H05, A01, A03 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Tôi chọn cluster 1 vì nó giải thích nhiều failure nhất và có tác động trực
tên đến Completeness, Faithfulness và một phần Relevance. Chỉ cần cải thiện một
điểm ở output validation/prompt completion là có thể giảm đồng thời nhiều case fail.

---

## 4. Improvement Log

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Answer is missing key information — increase context window or improve generation | Implement a stronger grounding check to block unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Tighten intent routing and improve prompt clarity so answers stay on task | Open |
| F003 | hallucination | Context is missing or irrelevant — improve retrieval | Increase context coverage and add examples that require complete answers | Open |
| F004 | incomplete | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
| F005 | incomplete | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
| F006 | incomplete | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Review pipeline and retriever | Open |
| F008 | off_topic | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
| F009 | irrelevant | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
| F010 | incomplete | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
| F011 | hallucination | Context is missing or irrelevant — improve retrieval | Review pipeline and retriever | Open |
| F012 | hallucination | Context is missing or irrelevant — improve retrieval | Review pipeline and retriever | Open |
| F013 | irrelevant | Answer does not address the question — improve prompt clarity | Review pipeline and retriever | Open |
| F014 | irrelevant | Answer does not address the question — improve prompt clarity | Review pipeline and retriever | Open |
| F015 | hallucination | Context is missing or irrelevant — improve retrieval | Review pipeline and retriever | Open |
| F016 | irrelevant | Answer is missing key information — increase context window or improve generation | Review pipeline and retriever | Open |
```

**Ba improvement suggestions ưu tiên**

1. Implement a stronger grounding check to block unsupported claims
2. Tighten intent routing and improve prompt clarity so answers stay on task
3. Increase context coverage and add examples that require complete answers

| Suggestion | Target metric | Verification method |
|---|---|---|
| Implement a stronger grounding check to block unsupported claims | Faithfulness | So sánh số hallucination cases trước/sau. |
| Tighten intent routing and improve prompt clarity so answers stay on task | Relevance | Đo avg Relevance và số case irrelevant/off_topic. |
| Increase context coverage and add examples that require complete answers | Completeness | Đo avg Completeness trên các case M01, M02, M06, H03. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy trong CI sau mỗi thay đổi prompt/retriever/model, và trước khi merge hoặc
deploy để phát hiện sớm metric tụt so với baseline.

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> Có, vì bài toán policy/student-services cần độ ổn định khá cao. Ngưỡng 0.05 đủ
bắt regression nhỏ nhưng đáng kể, nhưng vẫn nên kết hợp ngưỡng an toàn tuyệt đối
cho các metric quan trọng.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Block khi Faithfulness, Relevance hoặc Completeness giảm xuống dưới ngưỡng an
toàn hoặc khi hallucination/irrelevant tăng rõ rệt. Alert khi retrieval metrics
giảm nhẹ nhưng answer-side vẫn còn trong vùng chấp nhận.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Local tests] → [Benchmark regression] → [Human review / sign-off] → Deploy
```

> *Giải thích:*

> Đầu tiên chạy local tests để chắc code không vỡ. Sau đó chạy benchmark/regression
so với baseline. Cuối cùng review thủ công các failure lớn hoặc các case biên trước
khi deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Add output validation and retry for truncated answers | Completeness | Giảm các case answer cụt như M01, M02, M06. |
| 2 | Harden generation prompt to force complete, evidence-backed answers | Faithfulness, Relevance | Tăng answer-side averages và giảm output fragment. |
| 3 | Add regression guardrails for prompt injection and refusal correctness | Safety/privacy, Relevance | Cải thiện A01/A02 và hành vi từ chối đúng. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> Nên thêm các case output bị cắt cụt giống M01/M02/M06, một case prompt
injection rõ hơn như A02, và một case refusal/out-of-scope để kiểm tra chặn đúng.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Mình dự đoán retrieval tốt sẽ kéo overall lên đáng kể, nhưng thực tế Context
Recall và Context Precision rất cao trong khi answer-side metrics vẫn thấp.
Điều này cho thấy nút thắt nằm ở generation chứ không phải retrieval.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> Heuristic overlap chỉ đo trùng token, không hiểu nghĩa, không đánh giá được
paraphrase tốt, và dễ thưởng/phạt sai khi câu trả lời ngắn, dài hoặc có stopword.
Trong production, mình sẽ bổ sung semantic judge metrics, output structure checks,
và human review cho các case rủi ro cao.
