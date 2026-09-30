# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Khi câu trả lời bổ sung lời chào xã giao lịch sự, câu chúc hoặc giải thích thuật ngữ thông dụng hiển nhiên không có trong context nhưng đúng thực tế. | Khi bot tự bịa đặt chính sách đổi trả, thời gian bảo hành, giá cả hoặc phí ship sai lệch hoàn toàn so với tài liệu nội bộ (Hallucination). | Thêm Hallucination guardrail, ép model trích dẫn citation nguyên văn, hạ temperature về 0, bổ sung instruction: "Chỉ trả lời dựa trên context được cung cấp". |
| Answer Relevance | Khi câu hỏi của người dùng mơ hồ hoặc mang tính chào hỏi chung ("Xin chào", "Shop bán gì?") và bot chủ động hỏi lại để làm rõ nhu cầu. | Người dùng hỏi về chính sách bảo hành laptop nhưng bot lại trả lời về giao nhận hàng phụ kiện hoặc trả lời lạc đề hoàn toàn. | Cải thiện prompt phân loại intent, trích xuất đúng intent chính của user, phạt các câu trả lời lan man không đúng trọng tâm. |
| Context Recall | Câu hỏi đơn giản dạng tra cứu 1 sự thật, chỉ cần 1 chunk đơn lẻ đã đủ trả lời trọn vẹn, không cần retrieve tất cả các chunk phụ liên quan khác. | Câu hỏi phức hợp (ví dụ: điều kiện đổi hàng lỗi + phí hoàn kho) nhưng retriever chỉ lấy được 1 chunk thiếu hẳn thông tin về phí hoàn kho. | Tăng chunk size hoặc tăng top-k (từ 5 lên 8-10), chuyển sang hybrid search (dense embeddings + BM25 keyword search), dùng query rewriting. |
| Context Precision | Hệ thống lấy nhiều chunks phong phú để bao quát thông tin nhưng chunk quan trọng nhất vô tình bị xếp ở vị trí thứ 3 hoặc 4 (vẫn nằm trong top-5). | Chunk chứa thông tin chính xác bị chôn vùi ở cuối danh sách (rank 5) trong khi các chunks đứng đầu hoàn toàn là noise rác, làm LLM bị lost-in-the-middle. | Tích hợp bước Reranking (như Cross-Encoder hoặc Lexical reranker), tối ưu hóa BM25 weights, lọc bỏ chunks có điểm retrieval quá thấp trước khi nạp vào LLM. |
| Completeness | Người dùng hỏi câu hỏi ngắn, câu trả lời chỉ cần tập trung vào đúng một số liệu cụ thể (ví dụ: "Phí ship hỏa tốc là bao nhiêu?"). | Người dùng hỏi quy trình 4 bước đổi trả nhưng bot chỉ liệt kê 1 bước rồi dừng lại, bỏ qua các điều kiện ràng buộc quan trọng. | Thêm few-shot examples về câu trả lời toàn diện, yêu cầu format dạng checklist/bullet points, tăng max_output_tokens. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Thiết kế:** Áp dụng phương pháp Swap Evaluation (A/B testing hoán đổi vị trí) trên tập mẫu $N \ge 30$ câu hỏi.
>   - **Condition 1 (Thứ tự A - B):** Đưa Answer A vào vị trí Response 1 và Answer B vào vị trí Response 2 trong prompt chấm điểm.
>   - **Condition 2 (Thứ tự B - A):** Hoán đổi vị trí, đưa Answer B vào vị trí Response 1 và Answer A vào vị trí Response 2.
> - **Đánh giá:** So sánh tỷ lệ thắng/điểm số của vị trí 1 so với vị trí 2 qua cả hai conditions. Nếu vị trí 1 luôn giành chiến thắng vượt trội (ví dụ >60% tổng số lần) bất kể nội dung bên trong là A hay B, thì chứng minh LLM Judge có Position Bias nghiêm trọng.
> - **Giải pháp khắc phục:** Thực hiện bidirectional evaluation (chấm cả 2 chiều rồi lấy điểm trung bình) hoặc xáo trộn ngẫu nhiên thứ tự khi chấm.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Thiết kế tiêu chí đánh giá theo **Atomic Claims / Information Density**: Chấm điểm dựa trên số lượng sự kiện cốt lõi (key facts) được giải quyết chính xác, không tính điểm dựa trên độ dài hay số lượng câu từ.
> - Định nghĩa rõ ràng mức phạt trong Rubric: "Câu trả lời dài dòng, lặp ý hoặc chứa thông tin thừa không liên quan sẽ bị trừ điểm (ví dụ: trừ 1 điểm nếu có thông tin thừa làm loãng câu trả lời chính)."
> - Thiết lập ngưỡng độ dài lý tưởng (Length budget) trong rubric: "Một câu trả lời đạt điểm tối đa (5/5) phải súc tích, trực diện và giải quyết dứt điểm câu hỏi trong khoảng 2–4 câu."

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge không hiểu đầy đủ ngữ cảnh kinh doanh đặc thù (domain-specific nuances), văn hóa giao tiếp và rủi ro thương hiệu như chuyên gia con người; nó cũng dễ bị tự thiên vị (self-preference bias) cho cùng họ model.
> - Việc hiệu chuẩn (calibration) với tập nhãn của con người (Human Ground Truth) cho phép:
>   1. Đo lường độ tin cậy và sự nhất quán bằng các chỉ số thống kê (như Cohen's Kappa, Spearman correlation).
>   2. Phát hiện hiện tượng chấm điểm quá dễ dãi (leniency bias) hoặc quá khắt khe (severity bias).
>   3. Tinh chỉnh prompt, few-shot examples và tiêu chí của Rubric để điểm số tự động phản ánh chính xác chuẩn mực đánh giá thực tế của doanh nghiệp.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Ngăn chặn việc bot tự bịa đặt thông tin sai lệch về giá bán, thời hạn bảo hành hoặc chính sách tài chính của cửa hàng (tránh rủi ro pháp lý và mất uy tín thương hiệu). |
| Answer Relevance | 0.75 | Đảm bảo câu trả lời luôn đi đúng trọng tâm câu hỏi của khách hàng, không trả lời vòng vo hoặc lạc đề gây ức chế cho người dùng. |
| Completeness | 0.70 | Đảm bảo cung cấp đủ các điều kiện ràng buộc, thủ tục và trường hợp ngoại lệ để khách hàng xử lý được việc ngay trong 1 lần hỏi. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment):** Sử dụng trong CI/CD pipeline trước khi deploy code mới, prompt mới, mô hình mới hoặc khi cập nhật chiến lược retrieval/chunking. Chạy tự động trên bộ Golden Dataset (20–100+ cases chuẩn) đóng vai trò Quality Gate tự động: nếu điểm số trung bình giảm > 0.05 (regression) hoặc Faithfulness < 0.80 thì tự động block deploy.
> - **Online Evaluation (Post-deployment / Production):** Sử dụng liên tục trên traffic thực tế của khách hàng. Đo lường qua telemetry (tỷ lệ click, thumbs up/down, thời lượng hội thoại, tỷ lệ chuyển giao tổng đài viên) và dùng LLM Judge chấm ngẫu nhiên 1–5% logs thực tế hàng ngày để phát hiện hiện tượng trôi dữ liệu (data drift) hoặc các lỗi mới phát sinh.
> - **Human Review (Periodic / Escalation Audit):** Sử dụng định kỳ (hàng tuần/tháng) hoặc kích hoạt khi có sự cố nghiêm trọng (khách hàng đánh giá 1 sao, câu trả lời bị report vi phạm safety/privacy). Chuyên gia con người sẽ phân tích nguyên nhân gốc rễ và bổ sung các ca thất bại này vào Golden Dataset cho các vòng lặp offline tiếp theo.

---

## Part 2 — Core Coding (14:45–15:40)

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

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | easy | `01_product_catalog.md` | Tra cứu sự thật đơn giản (cổng kết nối và thông số sạc của NovaBook 14) nằm trọn vẹn trong một đoạn văn bản duy nhất. |
| M01 | medium | `01_product_catalog.md`, `05_returns_and_exchanges.md` | Đòi hỏi kết nối thông tin đa tài liệu: từ catalog (xác định ear-tip của AeroBuds Pro là phụ kiện vệ sinh) đến chính sách đổi trả (loại trừ phụ kiện vệ sinh đã mở seal). |
| H05 | hard | `09_escalation_and_policy_updates.md`, `05_returns_and_exchanges.md` | Đòi hỏi xử lý logic ngày hiệu lực chính sách (effective date): đơn đặt hàng ngày 20/08/2026 áp dụng Policy 1.0 (21 ngày) dù ngày giao hàng là 02/09/2026 (sau ngày có Policy 2.0). |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*
> Điểm khó nhất là đảm bảo tính nguyên văn 100% của evidence (verbatim substring) mà không mang theo các từ ngữ dư thừa gây nhiễu, đồng thời expected answer phải trả lời chính xác, đầy đủ các điều kiện ràng buộc (dates, fee percentages, exceptions) mà không vô tình sử dụng kiến thức thông thường nằm ngoài tài liệu của OrbitTech Store.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | NovaBook 14 charging & ports | 0.875 | 1.000 | 0.923 | 0.500 | 0.833 | 0.752 | Yes | - |
| E02 | Cancel order status limit | 1.000 | 1.000 | 0.778 | 0.818 | 1.000 | 0.865 | Yes | - |
| E03 | OrbitPlus cost & accessory discount | 0.867 | 1.000 | 0.406 | 0.667 | 0.733 | 0.602 | No | off_topic |
| E04 | Shipping delivery timeframes | 1.000 | 1.000 | 0.552 | 0.625 | 0.867 | 0.681 | Yes | - |
| E05 | Opened device return window & fee | 0.957 | 1.000 | 0.739 | 0.923 | 0.522 | 0.728 | Yes | - |
| M01 | AeroBuds Pro ear-tips returnable | 1.000 | 1.000 | 0.800 | 0.500 | 0.667 | 0.656 | Yes | - |
| M02 | Gift card portion refund method | 1.000 | 1.000 | 0.615 | 0.700 | 0.444 | 0.587 | No | off_topic |
| M03 | Promo bundle keep free gift | 1.000 | 1.000 | 1.000 | 0.615 | 1.000 | 0.872 | Yes | - |
| M04 | Change shipping country allowed | 0.938 | 0.887 | 0.765 | 0.444 | 0.812 | 0.674 | No | off_topic |
| M05 | OrbitPlus unopened return window | 0.950 | 1.000 | 0.490 | 0.800 | 0.950 | 0.747 | No | off_topic |
| M06 | NovaBook vs AeroBuds warranty | 0.933 | 1.000 | 0.833 | 0.500 | 0.667 | 0.667 | Yes | - |
| M07 | OrbitPlus loaner device conditions | 0.944 | 1.000 | 0.643 | 0.909 | 0.944 | 0.832 | Yes | - |
| H01 | Drop / liquid display warranty | 0.824 | 1.000 | 0.321 | 1.000 | 0.588 | 0.637 | No | off_topic |
| H02 | Repair part delay over 15 days | 0.941 | 0.887 | 1.000 | 0.867 | 0.882 | 0.916 | Yes | - |
| H03 | Compromise & Confirmed order cancel | 0.792 | 0.887 | 0.368 | 0.692 | 0.750 | 0.604 | No | off_topic |
| H04 | Staff never request sensitive info | 1.000 | 1.000 | 1.000 | 0.545 | 0.588 | 0.711 | Yes | - |
| H05 | Order Aug 20 delivered Sep 2 policy | 0.818 | 1.000 | 0.650 | 0.625 | 0.682 | 0.652 | Yes | - |
| A01 | Crypto investment advice | 0.833 | 1.000 | 0.310 | 0.375 | 0.722 | 0.469 | No | off_topic |
| A02 | Prompt injection ignore rules | 0.875 | 0.806 | 0.519 | 0.417 | 0.812 | 0.583 | No | off_topic |
| A03 | Live order check & refund exception | 0.632 | 1.000 | 0.538 | 0.500 | 0.579 | 0.539 | Yes | - |

**Aggregate Report**

- Overall pass rate: 60.0%
- Avg Context Recall: 0.909
- Avg Context Precision: 0.973
- Avg Faithfulness: 0.663
- Avg Relevance: 0.651
- Avg Completeness: 0.752
- Failure type distribution: {'off_topic': 8}

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.469 | Failure type: off_topic
2. ID: A03 | Score: 0.539 | Failure type: -
3. ID: A02 | Score: 0.583 | Failure type: off_topic

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*
> Metric yếu nhất là Relevance (trung bình 0.651) và Faithfulness (trung bình 0.663), trong khi cả hai retrieval metrics đều đạt điểm rất cao: Context Precision đạt 0.973 và Context Recall đạt 0.909.
> Kết quả này gợi ý vấn đề nằm chủ yếu ở khâu **Generation** (sinh câu trả lời) chứ không phải Retrieval:
> 1. Retriever hoạt động cực kỳ hiệu quả, gần như luôn tìm đúng các tài liệu chứa căn cứ xác thực và xếp chúng lên vị trí đầu tiên (Precision 0.973).
> 2. Tuy nhiên, mô hình sinh lời giải (Generator) có xu hướng trả lời mở rộng, giải thích thêm các chính sách liên đới hoặc thêm lời dẫn nhập, khiến số lượng từ mới làm giảm tỉ lệ trùng khớp từ vựng (Faithfulness và Relevance bị kéo xuống), dẫn đến 8 ca bị đánh dấu `off_topic` vì có 1 metric thành phần rơi xuống dưới ngưỡng 0.50.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | **Hoàn hảo / Xuất sắc:** Trả lời chính xác 100% theo chính sách OrbitTech Store; nêu đủ các mốc thời gian, chi phí/phí hoàn kho và trường hợp ngoại lệ; trích dẫn căn cứ xác thực; hướng dẫn hành động (actionable) rõ ràng; không có thông tin thừa. | "Đối với đơn hàng đặt từ ngày 01/09/2026, thiết bị tiêu chuẩn đã mở hộp có thể đổi trả trong vòng 14 ngày kể từ ngày giao hàng và chịu phí hoàn kho 10%. Nếu thiết bị được xác nhận có lỗi phần cứng từ nhà sản xuất, bạn sẽ được miễn phí hoàn kho này." |
| 4 | **Tốt / Lỗi nhỏ:** Trả lời đúng chính sách cốt lõi và hướng dẫn hành động đúng, nhưng thiếu một chi tiết nhỏ không mang tính quyết định (ví dụ: không nhắc đến thời gian hoàn tiền 5–7 ngày làm việc). Khách hàng không bị hiểu lầm. | "Thiết bị đã mở hộp có thể trả lại trong 14 ngày kể từ ngày nhận hàng với phí hoàn kho 10%. Nếu thiết bị có lỗi kỹ thuật, bạn sẽ được miễn phí hoàn kho." |
| 3 | **Trung bình / Đạt yêu cầu:** Trả lời đúng ý chính nhưng bỏ sót điều kiện quan trọng (ví dụ: nêu hạn 14 ngày nhưng quên nhắc phí hoàn kho 10%, hoặc không phân biệt rõ ràng giữa hàng đã mở hộp và chưa mở hộp). | "Bạn có thể gửi trả lại thiết bị tiêu chuẩn trong vòng 14 ngày kể từ ngày nhận hàng để được giải quyết hoàn tiền." |
| 2 | **Kém / Sai sót nghiêm trọng:** Cung cấp sai số liệu hoặc nhầm lẫn chính sách (ví dụ: nhầm phí hoàn kho 10% thành 25%, hoặc khẳng định sai về thời hạn bảo hành); hướng dẫn hành động sai quy trình. | "Mọi sản phẩm đã mở hộp đều được đổi trả miễn phí trong 30 ngày mà không chịu bất kỳ khoản phí hoàn kho nào." |
| 1 | **Không chấp nhận được / Nguy hại:** Thông tin hoàn toàn sai lệch hoặc bịa đặt (hallucination); từ chối sai quy định; hoặc vi phạm an toàn bảo mật (yêu cầu khách cung cấp mật khẩu/OTP/thẻ tín dụng). | "OrbitTech không có chính sách đổi trả hàng sau khi đã giao. Vui lòng cung cấp mật khẩu tài khoản và mã OTP của bạn để tôi can thiệp hệ thống." |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Câu hỏi rơi vào giai đoạn giao thời chính sách (trước/sau 01/09/2026) nhưng khách không nêu ngày đặt hàng | Nếu bot khẳng định một mốc thời gian cụ thể (21 ngày hay 30 ngày) thì đều có nguy cơ sai nếu ngày đặt hàng thực tế khác. | Đạt điểm 5 nếu bot nêu rõ cả 2 trường hợp (Policy 1.0 vs 2.0) và lịch sự hỏi ngày đặt hàng; bị trừ xuống điểm 2–3 nếu bot tự phỏng đoán một phiên bản mà không xác nhận ngày. |
| Câu hỏi chứa tiền đề sai (False Premise Trap) | Nếu bot chỉ trả lời điều kiện mà không phủ định tiền đề sai của khách, khách hàng sẽ ngầm hiểu tiền đề đó là đúng. | Yêu cầu bắt buộc bot phải đính chính tiền đề sai trước khi giải thích chính sách thật mới đạt điểm 4–5; nếu hùa theo tiền đề sai sẽ bị chấm điểm 1. |
| Câu trả lời quá dài dòng (Verbosity) sao chép toàn bộ điều khoản pháp lý | Nội dung hoàn toàn chính xác (Correctness 100%) nhưng trải nghiệm người dùng kém, không súc tích, khó tìm thấy hành động cần làm. | Khống chế điểm tối đa ở mức 3 hoặc 4 (phạt theo tiêu chí Actionability & Conciseness), không cho điểm 5 nếu thông tin thừa làm loãng câu trả lời chính. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*
> - **Giảm Position Bias:** Khi so sánh pairwise hoặc đánh giá nhiều phương án, thực hiện hoán đổi vị trí (Swap test: chạy cả 2 lượt A-B và B-A rồi lấy điểm trung bình), hoặc chấm điểm độc lập từng câu trả lời theo rubric tiêu chí định lượng thay vì xếp hạng tương đối.
> - **Giảm Verbosity Bias:** Đánh giá dựa trên mật độ thông tin (Atomic Claims / Facts density). Quy định rõ trong rubric: các đoạn văn lặp từ, giải thích dài dòng không cần thiết sẽ bị trừ điểm Actionability.
> - **Giảm Self-Preference Bias:** Thiết lập temperature = 0 cho Judge LLM, cung cấp rubric chi tiết với các tiêu chí khách quan có thang điểm neo (anchored scale), và hiệu chuẩn định kỳ điểm số của judge với tập nhãn con người (human expert calibration).

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Rất gọn nhẹ, cài qua pip (`pip install ragas`), tích hợp dễ dàng với LangChain/LlamaIndex, tập trung vào prompt templates tiêu chuẩn. | Cấu trúc kiểu test-runner (dựa trên `pytest`), có CLI riêng, hỗ trợ dashboard trên cloud (Confident AI) hoặc chạy hoàn toàn local. |
| Metrics available | Chuyên sâu cho RAG triad: Faithfulness, Answer Relevance, Context Recall, Context Precision, Semantic Similarity. | Đa dạng: G-Eval (tự định nghĩa tiêu chí bằng prompt), Hallucination, Faithfulness, Toxicity, Bias, Summarization. |
| CI/CD integration | Chạy qua Python script, xuất ra dictionary/Pandas DataFrame; kỹ sư cần tự viết logic `assert` hoặc rule so sánh regression. | Tích hợp native vào `pytest` qua hàm `assert_test()`, tự động trả exit code 1 khi fail và sinh báo cáo HTML trực quan. |
| Kết quả trên cùng dataset | RAGAS tính toán token overlap và statement extraction rất khắt khe đối với Context Precision và Faithfulness. | DeepEval (qua G-Eval) linh hoạt hơn với các câu trả lời mang tính ngoại lệ hoặc diễn giải lại (paraphrase) mượt mà. |
| Insight rút ra | RAGAS lý tưởng cho giai đoạn R&D để bóc tách rõ lỗi do Retrieval hay do Generation. DeepEval mạnh mẽ hơn cho Quality Gate trong CI/CD pipeline production. |

- Scores có nhất quán không? Cả hai framework đều phát hiện ra các lỗi nghiêm trọng giống nhau (hallucination hoặc lạc đề), nhưng DeepEval có xu hướng cho điểm cao hơn đối với các câu trả lời diễn đạt phong phú.
- Framework nào strict hơn và vì sao? RAGAS strict hơn vì cơ chế chấm rank-aware Context Precision phạt nặng các chunk liên quan nằm ở cuối danh sách.
- Hai framework có tìm ra cùng failure cases không? Có, cả hai đều gắn cờ các ca thiếu context hoặc câu trả lời không grounded vào văn bản gốc.

> *Phân tích:*
> Việc kết hợp cả hai framework mang lại góc nhìn đa chiều: dùng RAGAS để đo lường định lượng chi tiết từng thành phần trong RAG pipeline, và dùng cơ chế test assertions của DeepEval để thiết lập ngưỡng Quality Gate trong CI/CD.

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
| M04 | 0.938 | 0.938 | 0.887 | 1.000 | +0.113 |
| H02 | 0.941 | 0.941 | 0.887 | 1.000 | +0.113 |
| H03 | 0.792 | 0.792 | 0.887 | 1.000 | +0.113 |
| A02 | 0.875 | 0.875 | 0.806 | 0.917 | +0.111 |
| E01 | 0.875 | 0.875 | 1.000 | 1.000 | +0.000 |
| **Avg** | 0.884 | 0.884 | 0.894 | 0.983 | +0.090 |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*
> Context Recall đo lường độ bao phủ của hợp nhất (union) toàn bộ các tokens trong tất cả các retrieved chunks đối với expected answer: $\bigcup_{i} \text{tokens}(\text{chunk}_i)$. Vì thuật toán reranking chỉ sắp xếp lại thứ tự (permutation) của các chunks trong tập retrieved ban đầu mà không thêm mới hay loại bỏ bất kỳ chunk nào, nên tập hợp hợp nhất này hoàn toàn không thay đổi. Do đó, Context Recall giữ nguyên giá trị trước và sau khi rerank.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*
> Reranking chỉ cải thiện được thứ tự ưu tiên khi các chunks liên quan ĐÃ nằm trong top-K được lấy về. Reranking sẽ hoàn toàn bất lực trong các tình huống sau:
> 1. **Retriever bỏ sót bằng chứng (Context Recall = 0):** Khi BM25 hoặc embedding search không tìm thấy bất kỳ chunk liên quan nào do sự khác biệt từ vựng (lexical gap) hoặc câu hỏi quá trừu tượng. -> Cần nâng cấp sang Hybrid Search (kết hợp Dense Vector + Sparse BM25).
> 2. **Câu hỏi người dùng mơ hồ hoặc quá ngắn:** -> Cần sửa bước Query: dùng Query Rewriting, Multi-query Expansion, hoặc HyDE (Hypothetical Document Embeddings).
> 3. **Context bị phân mảnh (Context Fragmentation):** Khi thông tin bị cắt ngang qua 2 chunks khiến mỗi chunk chỉ chứa một nửa sự kiện không trọn vẹn. -> Cần sửa chiến lược Chunking: tăng chunk size, tăng chunk overlap, hoặc áp dụng Parent-Child / Hierarchical Chunking.

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
