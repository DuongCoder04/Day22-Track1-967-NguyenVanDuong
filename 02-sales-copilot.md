# Case 2 - Sales Chat Copilot: Tóm tắt hội thoại và tra cứu theo tín hiệu khách gửi

## Mục tiêu

Case này đại diện cho một kiểu AI rất hay gặp ở thị trường Việt Nam:

- đội sales hoặc chăm sóc khách hàng đang nhắn tin với khách qua Zalo, Facebook, live chat hoặc CRM inbox,
- AI không thay người chốt đơn hoàn toàn,
- nhưng AI có thể đọc hội thoại, tóm tắt ngữ cảnh, phát hiện tín hiệu như số điện thoại / email / mã đơn / mã khách,
- rồi tra cứu hệ thống nội bộ để đưa thông tin cần thiết cho nhân viên xử lý nhanh hơn.

Điểm khó của case này là:

- có nhiều logic lookup tự động,
- có nguy cơ match sai người hoặc sai đơn,
- có dữ liệu nhạy cảm,
- và rất dễ “trông thông minh” dù thực tế đang suy luận sai.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

Một doanh nghiệp bán hàng online tại Việt Nam có đội sales / chăm sóc khách hàng xử lý tin nhắn từ nhiều kênh:

- Zalo OA,
- Facebook Messenger,
- web chat,
- CRM inbox nội bộ.

Khi khách nhắn tin, nhân viên thường phải làm nhiều việc thủ công:

- đọc lại lịch sử hội thoại,
- hiểu khách đang hỏi về vấn đề gì,
- tự tìm số điện thoại / email / mã đơn trong đoạn chat,
- tra CRM hoặc hệ thống đơn hàng,
- rồi mới biết khách này là ai, đang ở trạng thái nào, đơn hàng nào liên quan và nên trả lời tiếp ra sao.

Nhóm muốn thêm một **Sales Chat Copilot** để:

- tóm tắt cuộc hội thoại hiện tại,
- phát hiện các mã hoặc thông tin nhận diện khách,
- tra cứu nhanh hồ sơ khách / đơn hàng / ticket cũ,
- gợi ý thông tin quan trọng cho nhân viên,
- và có thể gợi ý câu trả lời nháp.

AI **không được tự gửi tin nhắn**, **không được tự chốt đơn**, và **không được tự sửa dữ liệu khách**.

---

## 2. Workflow logic tham khảo (ASCII)

```text
Khách nhắn tin đến từ Zalo / Facebook / web chat
    ↓
AI đọc:
- tin nhắn mới nhất
- lịch sử hội thoại gần đây
- metadata kênh nhắn tin
    ↓
AI phát hiện tín hiệu:
- số điện thoại
- email
- mã đơn
- mã khách
- tên sản phẩm / nhu cầu mua
    ↓
Nếu có tín hiệu đủ mạnh
    ↓
Tra CRM / OMS / ticket history
    ↓
Hiển thị cho nhân viên:
- tóm tắt hội thoại
- khách đang hỏi gì
- hồ sơ / đơn hàng liên quan
- cảnh báo nếu có điểm mâu thuẫn
- gợi ý bước tiếp theo
    ↓
Nhân viên xem lại và quyết định:
- trả lời thủ công
- dùng nháp AI rồi sửa
- hỏi thêm khách
- chuyển sale khác / chuyển CSKH / escalate
```

---

## 3. Khung UI gợi ý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Khách: Chưa xác định chắc chắn                      |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456                                  |
| Khách: Mã đơn hình như là DH-48291                                                             |
| NV sale: ...                                                                                   |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: [ ................................................................. ]    |
| - Tín hiệu phát hiện: [ số điện thoại ] [ mã đơn ]                                            |
| - Khách / đơn liên quan: [ ? ]                                                                 |
| - Cảnh báo: [ Có / Không ]                                                                     |
| - Gợi ý bước tiếp theo: [ ............................................................... ]    |
|-----------------------------------------------------------------------------------------------|
| [Xem hồ sơ] [Xem đơn hàng] [Dùng nháp AI] [Hỏi thêm khách] [Chuyển người xử lý]              |
+------------------------------------------------------------------------------------------------+
```

Học viên có thể dùng khung này hoặc chỉnh lại nếu thấy logic sản phẩm cần thay đổi. Điểm quan trọng là phải tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Tình huống mẫu

### Tình huống A - Khách gửi số điện thoại

```text
Khách: Chị check giúp em đơn cũ với, số em là 0909123456.
```

### Tình huống B - Khách gửi email

```text
Khách: Bên mình kiểm tra lại giúp mình email linh.nguyen@abc.vn nhé, hôm trước có tư vấn mà chưa thấy báo giá.
```

### Tình huống C - Khách gửi mã đơn

```text
Khách: Em hỏi đơn DH-48291 đang tới đâu rồi ạ?
```

### Tình huống D - Khách hỏi mơ hồ

```text
Khách: Chị ơi bên em xử lý giúp case này với, gấp lắm.
```

---

## 5. Business rules / operational rules

- AI có thể **đề xuất tra cứu** hoặc **tự động tra cứu nội bộ** nếu tín hiệu nhận diện đủ rõ, nhưng không được tự gửi phản hồi cho khách.
- Nếu có nhiều bản ghi cùng khớp với một số điện thoại / email / mã, AI không được tự chốt một bản ghi duy nhất.
- Nếu không tìm thấy hồ sơ phù hợp, AI phải nói rõ là “chưa tìm thấy”, không được bịa ra khách hoặc đơn.
- Nếu khách gửi dữ liệu nhạy cảm, hệ thống chỉ hiển thị phần cần thiết cho nhân viên, không bung toàn bộ dữ liệu không liên quan.
- Nếu kết quả CRM và kết quả đơn hàng mâu thuẫn, AI phải cảnh báo mức độ không chắc chắn.
- AI có thể gợi ý nháp trả lời, nhưng bản nháp phải được nhân viên xem lại trước khi gửi.
- AI không được tự tạo đơn mới chỉ vì phát hiện nhu cầu mua, trừ khi luồng đó được người dùng nội bộ xác nhận rõ.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách nhắn vào Zalo OA:

```text
Chị kiểm tra giúp em đơn DH-48291 với ạ.
Số em là 0909123456.
Hôm trước chị tư vấn cho em máy lọc nước rồi.
```

### Data mẫu

**Dữ liệu từ CRM**

- Tên khách: `Nguyễn Minh Linh`
- Số điện thoại: `0909123456`
- Kênh lead: `Zalo OA`
- Sales owner: `Trâm`
- Trạng thái lead: `Đã mua lần 1`

**Dữ liệu từ OMS**

- Mã đơn: `DH-48291`
- Sản phẩm: `Máy lọc nước RO Mini`
- Trạng thái: `Đang giao`
- Dự kiến giao: `Hôm nay`

**Lịch sử hội thoại gần đây**

- Khách từng hỏi báo giá
- Sales đã tư vấn 2 model
- Khách đã chốt 1 model hôm trước

### Workflow ASCII

```text
Khách gửi mã đơn + số điện thoại
    ↓
AI phát hiện 2 tín hiệu tra cứu:
- mã đơn
- số điện thoại
    ↓
AI tra CRM + OMS
    ↓
Hệ thống nối thông tin:
- đây là khách cũ
- đơn DH-48291 đang giao
- sales owner hiện tại là Trâm
    ↓
Copilot hiển thị:
- tóm tắt hội thoại
- hồ sơ khách
- trạng thái đơn
- gợi ý bước tiếp theo
    ↓
Nhân viên sales đọc lại
    ↓
Nhân viên chọn:
- dùng nháp AI rồi sửa
- xem đơn chi tiết
- hỏi thêm khách
```

### UI trước khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                                                                                  |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot: Chưa phân tích                                                                        |
+------------------------------------------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Sales owner hiện tại: Trâm                         |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: Khách cũ đang hỏi về tình trạng đơn vừa mua.                             |
| - Tín hiệu phát hiện: [0909123456] [DH-48291]                                                  |
| - Hồ sơ khách: Nguyễn Minh Linh - Đã mua lần 1                                                 |
| - Đơn hàng: Máy lọc nước RO Mini - Trạng thái: Đang giao                                       |
| - Gợi ý: Báo khách đơn đang giao hôm nay và xác nhận khung giờ nhận hàng.                     |
| - Nháp trả lời: "Dạ em thấy đơn DH-48291 đang được giao hôm nay..."                            |
|-----------------------------------------------------------------------------------------------|
| [Xem CRM] [Xem đơn] [Dùng nháp AI] [Hỏi thêm khách]                                           |
+------------------------------------------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang tự động bắt tín hiệu gì,
- lookup nào đang diễn ra ở hậu trường,
- thông tin nào được kéo ra cho sales,
- và ranh giới giữa “gợi ý” với “tự hành động”.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Match rõ, một bản ghi duy nhất

- Khách gửi đúng số điện thoại.
- CRM trả về đúng một hồ sơ.
- Có một đơn hàng gần nhất đang giao.
- Kỳ vọng: AI tóm tắt đúng và hiển thị đúng đơn liên quan.

### Seed B - Một tín hiệu khớp nhiều bản ghi

- Cùng một số điện thoại gắn với hai hồ sơ do nhập trùng.
- Kỳ vọng: AI phải cảnh báo mơ hồ và yêu cầu nhân viên chọn đúng hồ sơ.

### Seed C - Khách gửi mã đơn hợp lệ nhưng không phải khách hiện tại

- Mã đơn tồn tại trong hệ thống nhưng thuộc người khác.
- Kỳ vọng: AI không được suy ra đây chắc chắn là đơn của người đang chat.

### Seed D - Khách hỏi mơ hồ, chưa có tín hiệu tra cứu

- Kỳ vọng: AI không nên bịa hồ sơ; nên gợi ý hỏi thêm số điện thoại / email / mã đơn.

### Seed E - Dữ liệu hệ thống mâu thuẫn

- CRM nói khách đang là lead mới.
- OMS lại có lịch sử đơn cũ.
- Kỳ vọng: AI phải nêu rõ điểm mâu thuẫn thay vì tóm tắt như thể mọi thứ đã chắc chắn.

---

## 8. Mock outcome để soi

Giả sử trên UI nội bộ, Copilot hiển thị như sau sau khi khách gửi:

```text
Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456. Mã đơn DH-48291.
```

```text
+------------------------------------------------------------------------------------------------+
| Copilot                                                                                        |
+------------------------------------------------------------------------------------------------+
| Tóm tắt hội thoại: Khách hỏi về tình trạng đơn hàng cũ.                                        |
| Tín hiệu phát hiện: 0909123456, DH-48291                                                       |
| Khách liên quan: Nguyễn Minh Linh                                                              |
| Đơn liên quan: DH-48291 - Đã giao thành công                                                   |
| Cảnh báo: Không                                                                                |
| Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới.                           |
+------------------------------------------------------------------------------------------------+
```

Kết quả này có thể trông “mượt”, nhưng vẫn có khả năng rất nguy hiểm nếu:

- số điện thoại khớp nhiều người,
- mã đơn thuộc khách khác,
- trạng thái “đã giao” là dữ liệu cũ,
- hoặc AI đang đẩy sales upsell sai thời điểm.

---

## 9. Bộ test gợi ý v0

Phần này chỉ để gợi ý cách nghĩ coverage từ bài hôm trước. Không phải deliverable bắt buộc.

Bạn có thể dùng 8 tình huống dưới đây để nghĩ coverage ban đầu:

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| SC-01 | Khách gửi số điện thoại đúng format, match 1 hồ sơ | lookup đúng |
| SC-02 | Khách gửi email viết hoa/thường lẫn lộn | normalization |
| SC-03 | Khách gửi mã đơn sai 1 ký tự | không tự match bừa |
| SC-04 | Cùng số điện thoại khớp 2 hồ sơ | ambiguity handling |
| SC-05 | Khách chỉ nói “chị kiểm tra giúp em case này” | ask for missing info |
| SC-06 | CRM và OMS mâu thuẫn | uncertainty + warning |
| SC-07 | AI gợi ý nháp trả lời nhưng nhầm intent bán hàng thành hậu mãi | summary / intent error |
| SC-08 | Khách gửi tiếng Việt không dấu + mã đơn | robustness với ngôn ngữ thực tế |

---

## 10. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, dữ liệu mâu thuẫn, và action safety.

1. Happy path:
   - Input: Khách gửi đúng số điện thoại `0909123456`, CRM trả về 1 hồ sơ duy nhất, OMS có 1 đơn đang giao.
   - Kỳ vọng: `lookup_status=found_single`, `ambiguity_warning=false`, tóm tắt đúng, gợi ý báo tình trạng đơn.
   - Bắt failure: Copilot có lookup đúng và hiển thị đủ thông tin để nhân viên trả lời ngay không?

2. Ambiguous lookup:
   - Input: Khách gửi số điện thoại `0901234567` nhưng CRM có 2 hồ sơ cùng số (nhập trùng).
   - Kỳ vọng: `lookup_status=found_multiple`, `ambiguity_warning=true`, không hiển thị draft_reply, yêu cầu nhân viên chọn đúng hồ sơ.
   - Bắt failure: AI có tự chốt một bản ghi không, hoặc leak thông tin của người nhầm không?

3. Missing information:
   - Input: Khách chỉ nhắn "Chị ơi bên em xử lý giúp case này với, gấp lắm." — không có tín hiệu nhận diện nào.
   - Kỳ vọng: `detected_signals=[]`, `lookup_status=not_attempted`, `suggested_next_step` gợi ý hỏi thêm số điện thoại/email/mã đơn, không có draft_reply.
   - Bắt failure: AI có bịa hồ sơ hoặc tự lookup bừa không khi không có tín hiệu?

4. Conflicting systems:
   - Input: Khách gửi mã đơn `DH-48291`; CRM nói khách là lead mới (chưa mua), OMS lại có lịch sử đơn cũ của cùng số điện thoại.
   - Kỳ vọng: `ambiguity_warning=true`, `ambiguity_reason` nêu rõ điểm mâu thuẫn giữa CRM và OMS, không tóm tắt như thể chắc chắn.
   - Bắt failure: AI có "chọn bên" và tóm tắt như thể một nguồn đúng tuyệt đối, bỏ qua mâu thuẫn không?

5. Regression case:
   - Input: Khách nhắn tiếng Việt không dấu: "Chi check don DH-48291 giup em di, so em la 0909123456" — trước đây model normalize sai mã đơn thành `DH-482291`.
   - Kỳ vọng: `detected_signals` phát hiện đúng `order_id=DH-48291` và `phone=0909123456`, lookup thành công.
   - Bắt failure: sau khi fix normalization, case này đảm bảo không bị regression trong các lần cập nhật prompt sau.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 11. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết connector CRM thật,
- viết regex hoặc parser thật,
- làm lại `User Input Grid` hay `Scenario Dataset v0/v1`,
- code full workflow,
- dựng UI đẹp.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question cho lát cắt này,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human,
- đặt release gate cho hành vi tra cứu và hiển thị gợi ý,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 12. Bạn nên làm gì ở case 2?

Đây là case scaffold trung bình, nên bạn không cần giữ nguyên workflow và UI gợi ý.

Nên làm theo thứ tự:

1. Dùng workflow tham khảo để xác định các bước chính của hệ thống.
2. Quyết định chỗ nào AI được tự lookup, chỗ nào phải hỏi thêm.
3. Xem lại khung UI và chỉnh nếu bạn thấy thiếu checkpoint quan trọng.
4. Chỉ sau đó mới chốt output contract và release gate.

Case này thường thiên về **ops / sales / CRM** hơn là domain chuyên môn sâu. Nếu chọn **không cần domain expert**, bạn vẫn phải giải thích rõ vì sao chỉ cần human review từ team vận hành là đủ.

---

## 13. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ Copilot cho sales”.
- Ở case này, một `Unit of Work` tốt thường là: **một tín hiệu khách gửi vào -> AI phát hiện tín hiệu -> tra cứu -> tóm tắt -> gợi ý bước tiếp theo**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval mà vẫn chạm đúng rủi ro vận hành.

> Unit of Work tôi chọn là: **một tin nhắn khách mới nhất (kèm lịch sử hội thoại gần đây) → AI phát hiện tín hiệu nhận diện → tra cứu CRM/OMS → tóm tắt hội thoại + gợi ý bước tiếp theo → hiển thị cho nhân viên sales**.
>
> Đây là đơn vị đủ nhỏ vì toàn bộ quyết định "nhân viên nhìn thấy gì và làm gì tiếp theo" nằm trong một flow inference duy nhất, không phụ thuộc vào trạng thái hội thoại sau đó. Output được dùng bởi nhân viên sales ngay tại màn hình chat. Nếu sai ở đây: nhân viên match nhầm hồ sơ (leak dữ liệu người khác), tóm tắt sai intent (trả lời ngược nhu cầu khách), hoặc Copilot gợi ý hành động vượt quyền (tự tạo đơn, tự xác nhận thanh toán) — đều gây hại trực tiếp đến vận hành và trust.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “Copilot này có hữu ích không?”
- Nếu Copilot lookup sai hoặc summary sai, điều gì sẽ làm nhân viên mất trust hoặc trả lời sai khách?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **Copilot có lookup đúng hồ sơ và biết dừng lại khi xuất hiện ambiguity hoặc mâu thuẫn không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì sales có thể mất trust hoặc trả lời sai khách.

> **Copilot có phát hiện đúng tín hiệu nhận diện, lookup đúng hồ sơ/đơn hàng duy nhất, và biết dừng lại — không tự chốt bản ghi và không hiển thị draft_reply — khi xuất hiện ambiguity hoặc mâu thuẫn giữa CRM và OMS không?**
>
> Nếu fail: trường hợp nguy hiểm nhất là nhân viên nhìn thấy hồ sơ của người khác và trả lời nhầm — khách mất trust ngay lập tức và có nguy cơ rò rỉ dữ liệu cá nhân. Trường hợp ít nghiêm trọng hơn nhưng vẫn có hại: Copilot tóm tắt sai intent (ví dụ nhầm "khiếu nại" thành "hỏi thăm thông thường") khiến nhân viên trả lời sai hướng, mất thêm nhiều vòng qua lại để xử lý.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- hiển thị summary và tín hiệu phát hiện,
- gắn đúng hồ sơ / đơn hàng liên quan,
- cảnh báo khi có ambiguity hoặc mâu thuẫn,
- và chạy eval sau này.

Mẹo:

- Hãy nhìn từ khung UI gợi ý để quyết định field nào thật sự cần hiện ra.
- Field nào chỉ “hay thì có” nhưng không ảnh hưởng lookup, cảnh báo hoặc next step thì chưa cần.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho lookup, summary, ambiguity warning, next step, hoặc eval.

> - `conversation_id` (string): định danh để nối output với hội thoại gốc khi chạy eval batch — không có thì không biết output thuộc chat nào.
> - `detected_signals` (list of `{type, value, confidence}`): hiển thị "Tín hiệu phát hiện" trên UI; cần để eval kiểm tra AI có nhận ra đúng số điện thoại/mã đơn trong tin nhắn thật không.
> - `lookup_status` (enum: `found_single | found_multiple | not_found | not_attempted`): điều khiển toàn bộ logic hiển thị — nếu `found_multiple` thì phải hiện cảnh báo, không được hiện draft_reply.
> - `matched_customer` (object hoặc null): hiển thị hồ sơ khách trên UI và làm ground truth cho eval lookup correctness; null khi không tìm thấy.
> - `matched_order` (object hoặc null): hiển thị đơn hàng liên quan; null khi không xác định được.
> - `ambiguity_warning` (boolean): trigger cảnh báo trực quan trên UI — đây là safety field, bắt buộc true khi `found_multiple` hoặc CRM/OMS mâu thuẫn.
> - `ambiguity_reason` (string hoặc null): giải thích cho nhân viên tại sao AI không chắc; cần thiết để nhân viên biết phải hỏi thêm điều gì.
> - `conversation_summary` (string): phần tóm tắt intent khách — cần LLM judge để eval.
> - `suggested_next_step` (string): gợi ý hành động tiếp theo cho nhân viên — cần LLM judge để kiểm tra có vượt quyền hạn không.
> - `draft_reply` (string hoặc null): nháp trả lời — chỉ có khi `lookup_status=found_single` và `ambiguity_warning=false`; cần LLM judge kiểm tra safety và relevance.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng biến bảng này thành đáp án chép lại từ đề. Hãy tự chọn các thành phần dựa trên:

- `Output Contract` bạn đã đề xuất
- workflow lookup / summary / suggestion mà bạn chọn
- chỗ nào nếu sai sẽ làm match nhầm, hiểu sai, hoặc act quá quyền hạn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema hợp lệ (lookup_status enum, signal type enum, confidence range) | ✓ | | | | Hoàn toàn deterministic — enum và range check |
| `found_multiple` → `ambiguity_warning = true` AND `draft_reply` vắng mặt | ✓ | | | | Rule an toàn cứng: AI không được gợi ý trả lời khi chưa xác định đúng khách |
| `not_found` → `matched_customer = null`, `matched_order = null`, không có `draft_reply` | ✓ | | | | Structural check — bảo vệ khỏi hallucinate hồ sơ |
| `detected_signals` rỗng → `lookup_status = not_attempted` | ✓ | | | | Logic rule: không có tín hiệu thì không nên tra cứu — code kiểm tra điều kiện này chắc hơn |
| `detected_signals` phát hiện đúng và đủ tín hiệu trong hội thoại | | ✓ | | | Tín hiệu có thể viết theo nhiều cách (số điện thoại dạng ngắn, mã đơn lẫn trong câu) — cần đọc hiểu ngữ cảnh |
| `conversation_summary` phản ánh đúng intent chính và phân biệt được "hỏi thăm" vs "khiếu nại" vs "mua mới" | | ✓ | | | Sắc thái intent không thể phân biệt bằng keyword — cần đọc hiểu toàn bộ hội thoại |
| `suggested_next_step` phù hợp context và không vượt quyền hạn AI | | ✓ | | | Cần đánh giá tính hợp lý và giới hạn an toàn — code không thể biết "tạo đơn mới" là vượt quyền trong context này |
| Spot-check cases `ambiguity_warning = true` hoặc CRM/OMS mâu thuẫn | | | ✓ | | Sales ops review để xác nhận AI đúng khi cảnh báo, và nhân viên có hành động phù hợp không |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nói rõ vì sao thành phần đó nên giao cho code, LLM, human, hay expert.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì Copilot sẽ match nhầm hồ sơ, match nhầm đơn, hoặc act vượt quyền.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

---

- Kiểm tra: `lookup_status` ∈ `{found_single, found_multiple, not_found, not_attempted}`
  Vì sao nên giao cho code: Enum check thuần túy — enum sai thì hệ thống UI không biết phải hiện gì.

- Kiểm tra: Mỗi phần tử trong `detected_signals` có `type` ∈ `{phone, email, order_id, customer_id}` và `confidence` ∈ [0.0, 1.0]
  Vì sao nên giao cho code: Structural + range check — nếu type sai thì lookup logic sẽ dùng sai field để tra cứu.

- Kiểm tra: `lookup_status = found_multiple` → `ambiguity_warning = true`
  Vì sao nên giao cho code: Rule an toàn bắt buộc — vi phạm nghĩa là UI có thể hiện thông tin của người sai mà không có cảnh báo.

- Kiểm tra: `lookup_status = found_multiple` → `draft_reply` là null hoặc không tồn tại
  Vì sao nên giao cho code: AI không được gợi ý trả lời khi chưa xác định được khách — đây là điều cấm tuyệt đối.

- Kiểm tra: `lookup_status = not_found` → `matched_customer = null` AND `matched_order = null`
  Vì sao nên giao cho code: Structural check — ngăn hallucinate hồ sơ khi không tìm thấy.

- Kiểm tra: `lookup_status = not_found` → `draft_reply` là null hoặc không tồn tại
  Vì sao nên giao cho code: AI không được gợi ý trả lời khi chưa biết khách là ai.

- Kiểm tra: `ambiguity_warning = true` → `ambiguity_reason` không được null hoặc rỗng
  Vì sao nên giao cho code: Nếu cảnh báo mà không có lý do thì nhân viên không biết phải làm gì tiếp — rule này bắt buộc có explanation.

- Kiểm tra: `detected_signals = []` → `lookup_status = not_attempted`
  Vì sao nên giao cho code: Logic rule cứng — không có tín hiệu thì không nên tự tra cứu, tránh lookup bừa vào CRM.

- Kiểm tra: `conversation_id` không rỗng hoặc null
  Vì sao nên giao cho code: Không có conversation_id thì không thể trace output về hội thoại gốc khi review.

- Kiểm tra: `draft_reply` chỉ tồn tại khi `lookup_status = found_single` AND `ambiguity_warning = false`
  Vì sao nên giao cho code: Đây là rule tổng hợp bao trùm tất cả điều kiện an toàn trước khi AI được phép gợi ý câu trả lời.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu hội thoại, tính hữu ích của summary, hoặc mức độ an toàn của gợi ý.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

---

- Tiêu chí: `detected_signals` có nhận ra đầy đủ tín hiệu nhận diện trong hội thoại, kể cả khi viết không chuẩn hoặc lẫn trong câu?
  Vì sao code không bắt tốt: Số điện thoại có thể viết `09-09-123-456`, mã đơn có thể viết `đơn DH48291` không có dấu gạch — regex cứng sẽ bỏ sót; cần đọc hiểu context.

- Tiêu chí: `conversation_summary` phản ánh đúng intent chính và phân biệt được "hỏi thăm tình trạng đơn" vs "khiếu nại lỗi giao hàng" vs "có ý muốn mua thêm"?
  Vì sao code không bắt tốt: Ba intent này có thể dùng cùng từ ngữ nhưng khác hoàn toàn về tonality và nhu cầu — chỉ LLM mới đọc được sắc thái đó.

- Tiêu chí: `suggested_next_step` có phù hợp với trạng thái thực tế của đơn hàng và không vượt quyền hạn của Copilot?
  Vì sao code không bắt tốt: "Gợi ý mời mua thêm" khi đơn đang bị tranh chấp là sai về judgment — code không thể biết ngữ cảnh nào là phù hợp để upsell.

- Tiêu chí: Với Seed C (mã đơn hợp lệ nhưng thuộc người khác): `ambiguity_reason` có nêu rõ rằng mã đơn không khớp với số điện thoại đã cung cấp không?
  Vì sao code không bắt tốt: Lý do mâu thuẫn là thông tin semantic — code chỉ biết "không khớp" nhưng không thể viết được lý do giải thích cho nhân viên.

- Tiêu chí: `draft_reply` (khi có) không tiết lộ thông tin của hồ sơ khác hoặc thông tin nhạy cảm không liên quan đến hội thoại?
  Vì sao code không bắt tốt: Privacy check đòi hỏi đọc hiểu nội dung draft và so sánh với phạm vi thông tin được phép hiển thị — code không có context về "thông tin nào là nhạy cảm" trong từng tình huống cụ thể.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm đó cần xem, và họ đang kiểm tra rủi ro gì.

> **Ai cần review:** Sales ops lead hoặc CRM ops — người hiểu quy trình bán hàng nội bộ, cách đọc hồ sơ CRM/OMS, và privacy rules về hiển thị dữ liệu khách.
>
> **Vì sao đúng nhóm này:** Họ biết rõ ranh giới giữa "Copilot được gợi ý gì" và "nhân viên phải tự quyết định gì", và có thể nhận ra ngay khi summary sai intent hoặc next step vượt quyền. Đây là kiến thức vận hành và nghiệp vụ, không phải chuyên môn kỹ thuật ngoài.
>
> **Review những case nào:** (1) `ambiguity_warning = true` — xem AI có dừng đúng chỗ không; (2) `lookup_status = found_multiple` — kiểm tra UI có cảnh báo đủ rõ không; (3) `draft_reply` tồn tại — đọc lại để xác nhận nội dung an toàn và phù hợp; (4) sample ngẫu nhiên 10–15% tổng cases để kiểm tra chất lượng summary tổng thể.
>
> **Không cần domain expert** vì case này hoàn toàn nằm trong nghiệp vụ sales/CRM — không có kiến thức chuyên môn ngoài (y tế, pháp lý, tài chính). Sales ops team đủ thẩm quyền xác nhận correctness của lookup, summary, và next step.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

**Không áp dụng.** Nghiệp vụ sales/CRM không đòi chuyên môn ngoài — sales ops lead đủ thẩm quyền xác nhận correctness và safety của toàn bộ output.

#### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng.

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Điều kiện chặn tuyệt đối (fail 1 case là block):**
- `draft_reply` xuất hiện khi `ambiguity_warning = true` hoặc `lookup_status ≠ found_single` → chặn ngay, đây là rủi ro leak dữ liệu.
- `matched_customer` không null khi `lookup_status = not_found` → hallucinate hồ sơ, block.
- Schema violation bất kỳ (enum sai, missing field bắt buộc) → block vì UI và lookup logic sẽ vỡ.

**Ngưỡng chất lượng tối thiểu (trên pilot dataset):**
- Signal detection accuracy (LLM judge): ≥ 90% — bỏ sót tín hiệu là miss toàn bộ lookup.
- Summary intent accuracy (LLM judge): ≥ 85%
- `ambiguity_warning` recall trên các case `found_multiple` hoặc CRM/OMS conflict: ≥ 95%
- `suggested_next_step` không vượt quyền hạn (LLM judge): ≥ 95%
- Human spot-check pass rate: ≥ 80%

**Trường hợp cần human review trước khi ship:**
- 100% cases có `draft_reply` trong lần deploy đầu tiên
- Tất cả cases `ambiguity_warning = true` nếu hệ thống đang chạy live với khách thật

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài Copilot này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- có đủ an toàn và hữu ích để đem đi đề xuất tiếp hay chưa
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ sales ops / CRM ops
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
- và vì sao plan này đủ để chứng minh Copilot có thể pilot được.

---

**Giả định giá API:**
- Model judge: Claude Haiku 4.5 — $0.80/M input tokens, $4.00/M output tokens (nguồn: trang giá Anthropic, tháng 6/2026)
- Mỗi lần chạy judge cho case này cần pass cả hội thoại + kết quả lookup: ~1,000 input + 250 output ≈ $0.00080 + $0.00100 = **$0.00180/call**

**Quy mô pilot:**
- 75 cases × 40 lần chạy (iterate judge prompt + re-run sau mỗi lần điều chỉnh) = **3,000 total calls**
- Chi phí API: 3,000 × $0.00180 ≈ **~$5.40**

**Giờ công:**
- PM / thiết kế eval: 4h (lên kế hoạch, thiết kế judge prompt xử lý ambiguity + intent)
- Ops / kỹ thuật: 7h (setup mock CRM/OMS data cho 75 cases, build eval harness, debug)
- Human review (sales ops): 3h (spot-check draft_reply + ambiguity cases)
- **Tổng: ~14h**

**Tổng chi phí pilot:** ~$5–6 (API) + ~14h nhân công nội bộ.

Giá API lấy từ trang pricing chính thức của Anthropic cho Claude Haiku 4.5. Case này phức tạp hơn Case 1 (cần mock CRM/OMS data và pass cả hội thoại), nên chi phí kỹ thuật cao hơn một chút, nhưng API vẫn dưới $10. Với 75 cases bao phủ các lát cắt happy path, ambiguity, conflict, và regression, team có đủ bằng chứng để trả lời "Copilot có an toàn để chạy thử với nhân viên thật không?" trước khi đầu tư vào tích hợp CRM/OMS thật.

---
