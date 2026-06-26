# Case 1 - Support Ticket Triage

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. Happy path:
   - Input: `subject: "Cannot login after password reset"`, `message: "I reset my password twice but still cannot log in. This is blocking my work."`, `customer_tier: enterprise`
   - Kỳ vọng: `category=technical`, `urgency=high`, `requires_human=true`, `route_to=technical_support`
   - Bắt failure: AI có nhận đúng tín hiệu "blocking work" + enterprise để đẩy urgency đủ cao không?

2. Ambiguous input:
   - Input: `subject: "Help"`, `message: "Please help asap"`, `customer_tier: free`
   - Kỳ vọng: `category=unknown` hoặc `confidence` thấp (< 0.5), không tự confident gán category mạnh
   - Bắt failure: AI có tự gán category sai khi thiếu tín hiệu rõ ràng, hoặc confidence cao bất thường không?

3. Missing information:
   - Input: `subject: "Error"`, `message: ""` (rỗng), `customer_tier: free`
   - Kỳ vọng: `category=unknown`, `confidence` thấp, `requires_human=false` (free tier + không có context)
   - Bắt failure: AI có hallucinate lý do hoặc tự suy diễn category khi input gần như trống không?

4. High-risk / escalation:
   - Input: `subject: "URGENT: payment failed and account disabled"`, `message: "Our team is locked out because your billing system failed. Fix this now."`, `customer_tier: enterprise`
   - Kỳ vọng: `category=billing`, `urgency=critical`, `requires_human=true`, `route_to=billing_ops` hoặc `human_escalation`
   - Bắt failure: AI có route sai sang product_team, hoặc đánh urgency thấp hơn critical không?

5. Regression case:
   - Input: `subject: "I need help with my invoice"`, `message: "There's a discrepancy in my last invoice, can someone check?"`, `customer_tier: enterprise`
   - Kỳ vọng: `category=billing`, `route_to=billing_ops` (không được route sang `product_team` hoặc `l1_support`)
   - Bắt failure: trước đây model nhầm "invoice" là product question → case này giữ trong dataset để đảm bảo fix không bị regression sau mỗi lần cập nhật prompt.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval.

> Unit of Work tôi chọn là: **một ticket đầu vào (subject + message + customer_tier) → AI triage → output gồm category, urgency, route_to, requires_human, confidence, và reason_codes**.
>
> Đây là đơn vị đủ nhỏ để eval vì toàn bộ quyết định định tuyến nằm trong một lần inference duy nhất, không phụ thuộc vào trạng thái ngoài. Output cuối cùng được dùng bởi nhân viên hỗ trợ nội bộ để xử lý hàng đợi — nếu AI sai ở đây, ticket bị route nhầm team, mất escalation, hoặc khách enterprise bị phản hồi chậm mà không có lý do nào được ghi nhận.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì ticket sẽ đi sai hoặc gây mất trust.

> **AI có gán đúng urgency và route_to để ticket enterprise có dấu hiệu blocking work không bị xếp vào hàng thường, và có trigger đúng requires_human khi cần escalation không?**
>
> Nếu fail ở đây, ticket của khách doanh nghiệp bị "chìm" trong hàng L1 thay vì được xử lý ưu tiên — nhân viên sẽ phát hiện ra sai khi khách leo thang qua email trực tiếp hoặc gọi vào, lúc đó trust đã mất. Ngoài ra, nếu reason_codes không khớp nội dung ticket, người review sẽ nghi ngờ toàn bộ output của AI dù các field khác đúng.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, escalation, hoặc eval.

> - `ticket_id` (string): định danh để nối output với input gốc khi chạy eval — không có thì không biết output thuộc ticket nào.
> - `category` (enum: `technical | billing | product_question | feature_request | unknown`): điều khiển routing logic; nếu sai category thì route_to sẽ sai theo.
> - `urgency` (enum: `low | medium | high | critical`): điều khiển hàng đợi ưu tiên và trigger rule escalation cho enterprise.
> - `route_to` (enum: `technical_support | billing_ops | product_team | l1_support | human_escalation`): quyết định trực tiếp ticket đi đến team nào — đây là field quan trọng nhất với vận hành.
> - `requires_human` (boolean): cờ "cần xử lý ngay" hiển thị trên UI, đồng thời là điều kiện trong business rule enterprise + high/critical.
> - `confidence` (float 0–1): cần để gate review — confidence thấp là tín hiệu cần human look.
> - `reason_codes` (list of string): giải thích ngắn về lý do triage; cần để LLM judge kiểm tra hallucination và human review kiểm tra tính hợp lý.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema hợp lệ (enum đúng, confidence 0–1, requires_human boolean) | ✓ | | | | Hoàn toàn deterministic — chỉ cần kiểm tra kiểu dữ liệu và giá trị nằm trong tập cho phép |
| Business rule: enterprise + high/critical → requires_human = true | ✓ | | | | Rule cứng, đủ điều kiện AND rõ ràng — code kiểm tra chính xác hơn LLM |
| Business rule: billing → route_to ≠ product_team | ✓ | | | | Rule dạng "điều cấm" — code kiểm tra nhanh và không bị bỏ sót |
| Dấu hiệu "blocking work / locked out" → urgency ≠ low | ✓ | | | | Có thể dùng keyword matching trên tập từ khóa đã định nghĩa sẵn |
| Category phản ánh đúng nội dung ticket | | ✓ | | | Cần đọc hiểu ngữ nghĩa — "account disabled" có thể là billing hoặc technical tùy context |
| Urgency phản ánh đúng mức ảnh hưởng mà khách mô tả | | ✓ | | | Mức độ khẩn mang sắc thái ý nghĩa, không thể map bằng keyword cứng |
| reason_codes không hallucinate ngoài nội dung input | | ✓ | | | Cần so sánh lý do với input gốc để phát hiện thông tin bịa — LLM judge tốt hơn regex |
| Spot-check cases confidence thấp hoặc requires_human nhưng lý do mơ hồ | | | ✓ | | Ops team review để xác nhận AI không bỏ sót escalation thật — không cần domain chuyên sâu |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

---

- Kiểm tra: `category` nằm trong tập `{technical, billing, product_question, feature_request, unknown}`
  Vì sao nên giao cho code: Enum check thuần túy, không cần ngữ nghĩa — code nhanh và chắc chắn hơn.

- Kiểm tra: `urgency` nằm trong tập `{low, medium, high, critical}`
  Vì sao nên giao cho code: Tương tự — tập giá trị cố định, sai thì hệ thống routing sẽ vỡ.

- Kiểm tra: `route_to` nằm trong tập `{technical_support, billing_ops, product_team, l1_support, human_escalation}`
  Vì sao nên giao cho code: Route_to là input trực tiếp vào hệ thống phân luồng — nếu sai giá trị thì ticket bị dropped.

- Kiểm tra: `confidence` là số float trong khoảng `[0.0, 1.0]`
  Vì sao nên giao cho code: Kiểm tra kiểu dữ liệu và range số học — không cần phán xét ngữ nghĩa.

- Kiểm tra: `requires_human` là boolean (`true` hoặc `false`)
  Vì sao nên giao cho code: Type check đơn giản, không có sắc thái.

- Kiểm tra: `ticket_id` trong output khớp `ticket_id` trong input
  Vì sao nên giao cho code: String equality — phòng trường hợp AI nhầm lẫn output với ticket khác khi chạy batch.

- Kiểm tra: Nếu `customer_tier = enterprise` AND `urgency IN (high, critical)` → `requires_human = true`
  Vì sao nên giao cho code: Business rule dạng điều kiện AND rõ ràng, không có sắc thái — vi phạm là lỗi nghiêm trọng cần block ngay.

- Kiểm tra: Nếu `category = billing` → `route_to ≠ product_team`
  Vì sao nên giao cho code: Điều cấm trong business rules, check logic cứng — code không bỏ sót.

- Kiểm tra: Nếu message chứa bất kỳ từ khóa trong `{blocking work, locked out, account disabled, blocked}` → `urgency ≠ low`
  Vì sao nên giao cho code: Keyword matching trên danh sách đã định nghĩa — xác định được chính xác khi input chứa các cụm từ rõ ràng.

- Kiểm tra: `reason_codes` không phải list rỗng khi `confidence < 0.6`
  Vì sao nên giao cho code: Nếu confidence thấp mà không có reason thì reviewer không biết AI đang "nghi ngờ" điều gì — rule này bắt buộc explain low-confidence.

- Kiểm tra: `reason_codes` là list of string, không có phần tử rỗng hoặc null
  Vì sao nên giao cho code: Structural check — phòng model trả về list `[null]` hoặc `[""]` khi không biết phải nói gì.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

---

- Tiêu chí: `category` phản ánh đúng chủ đề chính của ticket (không phải theo keyword mà theo ý nghĩa)
  Vì sao code không bắt tốt: Một ticket có thể chứa từ "account" vừa dùng được cho billing vừa cho technical — chỉ có đọc hiểu context mới phân biệt được. Code keyword matching sẽ bị nhầm trong các case mơ hồ.

- Tiêu chí: `urgency` phản ánh đúng mức độ ảnh hưởng thực tế mà khách mô tả, không chỉ theo từ "URGENT" hay "asap"
  Vì sao code không bắt tốt: Khách có thể viết "URGENT" cho vấn đề nhỏ, hoặc mô tả "cả team bị chặn" mà không dùng từ urgent — cần hiểu mức độ impact thật, không phải từ khóa.

- Tiêu chí: `reason_codes` phản ánh đúng nội dung ticket, không thêm thông tin không có trong input (hallucination check)
  Vì sao code không bắt tốt: Code không thể so sánh ngữ nghĩa giữa reason và input — cần LLM đọc cả hai và xác nhận reason có được support bởi input thật sự.

- Tiêu chí: Với Seed B (input gần như trống — "Help, please help asap"), AI không được tự tin gán category mạnh (confidence > 0.6 trên input thiếu thông tin là dấu hiệu nguy hiểm)
  Vì sao code không bắt tốt: Code chỉ kiểm tra được confidence có trong range hợp lệ — không biết confidence đó "có xứng đáng" với lượng thông tin trong input hay không.

- Tiêu chí: `route_to` có logic nhất quán với `category` được chọn (ví dụ: billing → billing_ops, technical → technical_support)
  Vì sao code không bắt tốt: Code chỉ kiểm tra được điều cấm (billing → không phải product_team) nhưng không verify route có *phù hợp nhất* với category không — ví dụ billing mà route về l1_support là sai về judgment dù không vi phạm rule cứng.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

> **Ai cần review:** Support ops lead (trưởng nhóm hỗ trợ vận hành), không cần domain expert chuyên sâu.
>
> **Vì sao đúng nhóm này:** Họ là người hiểu rõ nhất routing taxonomy nội bộ (technical vs billing vs product), biết khách nào thuộc enterprise với SLA nào, và đã xử lý đủ loại ticket để nhận ra khi AI route sai. Đây là kiến thức vận hành, không phải chuyên môn kỹ thuật hay y tế.
>
> **Review những case nào:** (1) `confidence < 0.6` — AI tự báo không chắc chắn; (2) `requires_human = true` nhưng reason_codes mơ hồ hoặc thiếu dấu hiệu rõ ràng; (3) `category = unknown` — để xác nhận AI đúng khi dừng lại thay vì đoán mò; (4) sample ngẫu nhiên 10–15% tổng cases để spot-check chất lượng chung.
>
> **Không cần domain expert** vì taxonomy của hệ thống này (technical/billing/product) là quy ước nội bộ của công ty, không đòi hỏi kiến thức chuyên môn ngoài (y tế, pháp lý, tài chính...). Support ops team đủ thẩm quyền xác nhận correctness và điều chỉnh routing policy nếu cần.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

**Không áp dụng.** Taxonomy routing (technical/billing/product) là quy ước nội bộ — support ops lead đủ thẩm quyền xác nhận, không cần chuyên môn ngoài.

#### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng.

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Điều kiện chặn tuyệt đối (không thương lượng — fail 1 case là block):**
- Schema violation > 0%: bất kỳ output nào có enum sai, confidence ngoài [0,1], hoặc requires_human không phải boolean → chặn ngay vì hệ thống routing sẽ vỡ.
- Business rule vi phạm > 0%: enterprise + high/critical mà requires_human = false → chặn vì bỏ sót escalation thật.
- Billing route sang product_team > 0% → chặn vì vi phạm rule cứng.

**Ngưỡng chất lượng tối thiểu (chạy trên pilot dataset):**
- Category accuracy (LLM judge): ≥ 85%
- Urgency accuracy (LLM judge): ≥ 85%
- requires_human recall trên enterprise high/critical cases: ≥ 95% (bỏ sót escalation thật nguy hiểm hơn false alarm)
- reason_codes hallucination rate (LLM judge): ≤ 5%
- Human spot-check pass rate: ≥ 80%

**Trường hợp cần human review trước khi ship:**
- Bất kỳ case nào confidence < 0.5 được route tự động mà không có human-in-loop
- Tất cả cases `category = unknown` khi urgency là high/critical
- 100% cases pilot nếu đây là lần deploy đầu tiên

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài triage này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- cần thêm những checkpoint nào trước khi đề xuất triển khai tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / kỹ thuật
- tổng giờ human review
- nếu có `domain expert`, tổng số giờ expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- và vì sao plan này đủ để chứng minh case có thể pilot được.

---

**Giả định giá API:**
- Model dùng làm LLM judge: Claude Haiku 4.5 — $0.80/M input tokens, $4.00/M output tokens (nguồn: trang giá Anthropic, tháng 6/2026)
- Mỗi lần chạy judge: ~800 input tokens + ~200 output tokens ≈ $0.00064 + $0.00080 = **$0.00144/call**

**Quy mô pilot:**
- 75 cases × 40 lần chạy (bao gồm iterate prompt judge + re-run sau fix) = **3,000 total calls**
- Chi phí API: 3,000 × $0.00144 ≈ **~$4.33**

**Giờ công:**
- PM / thiết kế eval: 4h (lên kế hoạch, viết judge prompt ban đầu, phân tích kết quả)
- Ops / kỹ thuật: 6h (build eval harness đơn giản với Python, chạy batch, debug schema)
- Human review: 3h (support ops lead spot-check 75 cases + ghi nhận lỗi)
- **Tổng: ~13h**

**Tổng chi phí pilot:** ~$4–5 (API) + ~13h nhân công nội bộ.

Giá API lấy từ trang pricing chính thức của Anthropic cho Claude Haiku 4.5. Với quy mô 75 cases và ~$5 API, đây là pilot chi phí rất thấp — đủ để trả lời câu hỏi "hệ thống có làm đúng trên các lát cắt cơ bản không?" trước khi đầu tư vào dataset lớn hơn. Nếu pass được 75 cases này với ngưỡng đề xuất, team có đủ bằng chứng để đề xuất chạy thử thật với ~200 ticket live.

---
