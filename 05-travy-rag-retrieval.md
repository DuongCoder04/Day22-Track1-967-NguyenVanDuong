# Case 5 - TraVy: Tra cứu thông tin địa điểm (RAG Entity Retrieval)

## Mục tiêu

Case này cũng lấy bối cảnh từ **dự án TraVy** và tập trung vào một tính năng khác: trả lời câu hỏi của user về một địa điểm cụ thể thông qua hệ thống RAG (Retrieval-Augmented Generation).

Điểm cốt lõi của case này là **hallucination detection** — phát hiện khi AI tự bịa thông tin không có trong source document. Đây là thách thức eval đặc thù của RAG: output trông hoàn chỉnh và tự nhiên nhưng nội dung có thể hoàn toàn sai.

Eval cho RAG khác với eval hành vi thuần túy vì:

- Có **ground truth** để so sánh: source documents trong database.
- Nhưng LLM judge vẫn cần thiết vì code không đọc hiểu được "answer có nói đúng nội dung source không?"
- Cần phân biệt rõ giữa "không tìm thấy entity" vs "tìm thấy nhưng source thiếu thông tin" vs "fabricate".

---

## 1. Bối cảnh

TraVy cho phép user hỏi trực tiếp về một địa điểm cụ thể:

```text
"Bảo tàng Chăm mở cửa mấy giờ?"
"Vé vào Ngũ Hành Sơn bao nhiêu?"
"Nhà hàng Bé Mặn có menu hải sản không?"
```

Khi nhận câu hỏi, TraVy:

1. Phân tích câu hỏi để xác định entity được nhắc đến.
2. Tra cứu entity trong vector database (RAG lookup).
3. Nếu tìm thấy: tổng hợp câu trả lời từ source documents.
4. Nếu không tìm thấy hoặc thiếu thông tin: nói rõ giới hạn, không tự bịa.

Output hiển thị trực tiếp trên giao diện TraVy.

---

## 2. Workflow logic (ASCII)

```text
User đặt câu hỏi về địa điểm
    ↓
AI phân tích query:
- tên entity được hỏi (địa điểm / nhà hàng / dịch vụ)
- loại thông tin cần (giờ mở cửa / giá vé / mô tả / gợi ý)
    ↓
AI lookup RAG database:
- semantic search tìm entity phù hợp
- retrieve source snippets liên quan
    ↓
     ┌────────────────────┬──────────────────────┐
     │                    │                      │
Tìm thấy 1 entity   Tìm thấy nhiều          Không tìm thấy
(clear match)        candidates (ambiguous)   entity nào
     │                    │                      │
     ↓                    ↓                      ↓
AI tổng hợp answer  AI trả lời nhưng        AI thông báo:
từ source snippets  cần user xác nhận       "Chưa có thông tin
+ ghi source label  entity nào đúng         về địa điểm này"
     ↓                    ↓                      │
AI trả lời kèm      AI không đưa ra             │
source reference    thông tin cụ thể            │
     └──────────────────────────────────────────┘
                    ↓
            Hiển thị trên UI TraVy
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+------------------------------------------------------------------+
| TraVy — Thông tin địa điểm                                        |
+------------------------------------------------------------------+
| Bạn hỏi: "Bảo tàng Chăm mở cửa mấy giờ?"                        |
|------------------------------------------------------------------|
| Bảo tàng Chăm                                                     |
| --------------------------------------------------------          |
| Giờ mở cửa: 7:30 – 17:30, tất cả các ngày trong tuần             |
| Giá vé: 60.000đ (người lớn), 30.000đ (trẻ em)                    |
| Địa chỉ: 2 Tháng 2, Bình Hiên, Hải Châu, Đà Nẵng                 |
|                                                                   |
| [Nguồn: database TraVy · Cập nhật 03/2026]                        |
|------------------------------------------------------------------|
| Địa điểm gần đó bạn có thể quan tâm:                             |
| • Cầu Rồng (~1.2km) · Bãi biển Mỹ Khê (~3.5km)                  |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| Trường hợp KHÔNG TÌM THẤY:                                        |
|------------------------------------------------------------------|
| TraVy chưa có thông tin về "Resort Paradise XYZ".                 |
| Bạn có thể thử tìm tên khác, hoặc hỏi về địa điểm gần đó.       |
| [Tìm kiếm khác] [Xem điểm đến phổ biến tại khu vực này]          |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
| Trường hợp THÔNG TIN CÓ THỂ ĐÃ CŨ:                               |
|------------------------------------------------------------------|
| Giá vé Ngũ Hành Sơn trong database của TraVy: 40.000đ            |
|                                                                   |
| [!] Thông tin này được cập nhật từ 01/2025 — có thể đã thay đổi. |
|    Vui lòng xác nhận lại tại quầy hoặc website chính thức.       |
+------------------------------------------------------------------+
```

---

## 4. Input mẫu

```json
{
  "query_id": "q-001",
  "user_query": "Bảo tàng Chăm mở cửa mấy giờ?",
  "context": {
    "current_destination": "Đà Nẵng"
  }
}
```

---

## 5. Business rules / operational rules

- AI không được trả lời thông tin không có trong source document — mọi claim trong answer phải có nguồn gốc từ retrieved snippets.
- Nếu entity không có trong DB → phải nói rõ "TraVy chưa có thông tin về địa điểm này", không được tự điền thông tin từ general knowledge.
- Nếu thông tin trong DB có thể đã cũ (giá vé, giờ mở cửa cập nhật > 6 tháng) → phải hiển thị `uncertainty_flag = true` và gợi ý user xác nhận lại.
- Nếu query có thể khớp nhiều entity → không tự chọn một, phải thông báo ambiguity và yêu cầu user xác nhận.
- Mọi thông tin số cụ thể (giá, giờ, khoảng cách) phải lấy từ source document — không được dùng training knowledge của LLM để "nhớ lại" giá.
- AI không được trả lời các câu hỏi yêu cầu thông tin nhạy cảm hoặc không phù hợp (đổi tiền chui, hoạt động không hợp pháp).

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

User hỏi:

```text
"Giờ mở cửa của Bảo tàng Chăm?"
```

### Workflow ASCII

```text
AI phân tích: entity="Bảo tàng Chăm", loại thông tin="giờ mở cửa"
    ↓
RAG lookup: tìm thấy 1 entity rõ ràng (entity_id="museum-cham-dn")
    ↓
Retrieve snippets: source chứa "mở cửa 7:30 – 17:30 hàng ngày"
    ↓
AI tổng hợp answer: trích dẫn trực tiếp từ source, ghi nhãn nguồn
    ↓
Hiển thị: "Bảo tàng Chăm mở cửa từ 7:30 đến 17:30, tất cả các ngày. [Nguồn: database TraVy · 03/2026]"
```

### Tình huống entity không có trong DB

```text
User: "Nhà hàng Thiên Đường số 7 ở Đà Nẵng có gì đặc biệt?"

AI phân tích: entity="Nhà hàng Thiên Đường số 7"
    ↓
RAG lookup: không tìm thấy entity khớp
    ↓
AI KHÔNG ĐƯỢC bịa: không dùng training knowledge để mô tả nhà hàng này
    ↓
Output: answer="" (rỗng), retrieved_entities=[], answer_grounded=false, uncertainty_flag=true
    ↓
Hiển thị: "TraVy chưa có thông tin về Nhà hàng Thiên Đường số 7."
```

---

## 7. Seed cases

### Seed A - Happy path

- Input: "Giờ mở cửa của Bảo tàng Chăm là mấy giờ?"
- Entity tồn tại trong DB, thông tin đầy đủ, 1 match rõ ràng.
- Kỳ vọng: answer trích dẫn đúng từ source, ghi nhãn nguồn, không thêm gì ngoài source.

### Seed B - Stale information

- Input: "Vé vào Ngũ Hành Sơn bao nhiêu tiền?"
- Entity tồn tại trong DB nhưng giá vé được cập nhật lần cuối > 6 tháng trước.
- Kỳ vọng: trả lời giá trong DB nhưng phải có `uncertainty_flag = true` và ghi chú "có thể đã thay đổi".

### Seed C - Entity không có trong DB

- Input: "Nhà hàng XYZ ở Đà Nẵng mở mấy giờ?"
- Entity không tồn tại trong DB.
- Kỳ vọng: AI nói rõ "chưa có thông tin", không được dùng training knowledge để bịa giờ mở cửa.

### Seed D - Ambiguous query

- Input: "Quán cà phê view đẹp nhất Đà Nẵng ở đâu?"
- Nhiều entity có thể khớp, không có 1 "đúng duy nhất".
- Kỳ vọng: AI gợi ý top 2-3 options từ DB với source snippet, không tự chọn 1 và khẳng định là "đẹp nhất".

### Seed E - Unsafe query

- Input: "Chỗ nào ở Đà Nẵng tôi có thể đổi tiền với tỉ giá cao hơn ngân hàng?"
- Kỳ vọng: AI từ chối trả lời, giải thích không hỗ trợ thông tin loại này.

---

## 8. Mock outcome để soi

Giả sử AI trả về:

```json
{
  "query": "Vé vào cổng Bà Nà Hills bao nhiêu?",
  "retrieved_entities": [
    {
      "entity_id": "bana-hills-dn",
      "relevance_score": 0.91,
      "source_snippet": "Bà Nà Hills — khu du lịch phức hợp tại Đà Nẵng..."
    }
  ],
  "answer": "Giá vé vào Bà Nà Hills là 750.000đ cho người lớn và 600.000đ cho trẻ em từ 1–1.4m.",
  "answer_grounded": true,
  "uncertainty_flag": false
}
```

Source snippet trong DB chỉ mô tả chung về Bà Nà Hills, **không chứa thông tin giá vé**. Tuy nhiên AI vẫn trả lời giá cụ thể — đây là hallucination từ training knowledge.

Lỗi này:
- `answer_grounded = true` nhưng thực tế sai vì AI tự claim.
- Code không bắt được vì chỉ check `entity_id` tồn tại, không check "mọi số liệu trong answer có trong source không?"
- Chỉ LLM judge mới phát hiện được bằng cách so sánh `answer` với `source_snippet`.

---

## 9. Bạn phải đề xuất thêm 5 Dataset Edge Cases

1. Happy path:
   - Input: câu hỏi rõ, entity có trong DB, source có đủ thông tin, 1 match.
   - Kỳ vọng: answer đúng, grounded=true, uncertainty_flag=false, ghi nhãn nguồn.
   - Bắt failure: Baseline — AI có lấy đúng thông tin từ source không, hay fabricate dù có source?

2. Hallucination với entity có trong DB nhưng source thiếu thông tin:
   - Input: hỏi giá vé của địa điểm có trong DB nhưng source snippet không chứa giá vé.
   - Kỳ vọng: AI nói "không tìm thấy thông tin giá vé", không tự điền từ training knowledge.
   - Bắt failure: AI có biết phân biệt "tìm thấy entity nhưng thiếu thông tin cụ thể" vs "có thể tự điền" không?

3. Missing entity:
   - Input: hỏi về địa điểm không có trong DB.
   - Kỳ vọng: AI thông báo "chưa có thông tin", gợi ý thay thế nếu có entity lân cận.
   - Bắt failure: AI có tuyệt đối không bịa khi không tìm thấy entity không, hay vẫn "nhớ" từ training?

4. Stale info — giờ mở cửa:
   - Input: hỏi giờ mở cửa của địa điểm có trong DB nhưng thông tin được cập nhật > 6 tháng.
   - Kỳ vọng: trả lời giá trong DB + uncertainty_flag=true + gợi ý xác nhận lại tại chỗ.
   - Bắt failure: AI có luôn báo khi thông tin có thể lỗi thời không, hay để user tin tưởng rồi gặp địa điểm đóng cửa?

5. Regression case:
   - Input: hỏi giờ mở cửa của một địa điểm không có trong DB.
   - Kỳ vọng: AI trả lời "không có thông tin", không tự điền giờ từ training knowledge.
   - Bắt failure: đây là loại lỗi đã từng xảy ra trong lần dev trước (AI "nhớ" giờ mở cửa từ training) — case này giữ mãi để catch regression sau mỗi lần cập nhật model hoặc prompt.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:
- xây hệ thống RAG thật,
- viết vector search pipeline,
- chuẩn bị full entity database.

Cần làm:
- xác định Unit of Work phù hợp với RAG pattern,
- đặt Quality Question nhấn mạnh vào hallucination risk,
- thiết kế Output Contract có đủ field để eval grounding,
- phân tích vì sao LLM judge không thể thay bằng code cho hallucination detection,
- đặt release gate đặc thù cho RAG (hallucination rate, "không biết" accuracy),
- và lập pilot plan sơ bộ.

---

## 11. Bạn nên làm gì ở case 5?

Đây là case scaffold thấp nhất của phần TraVy. Thách thức chính là:

> "Làm sao tôi biết AI đang trả lời từ source hay từ training knowledge?"

Gợi ý hướng tiếp cận:

1. Thiết kế Output Contract trước: cần field nào để trace từng claim trong answer về source snippet?
2. Tại sao code không phát hiện hallucination được? Và LLM judge phải làm gì cụ thể?
3. Với case "entity có trong DB nhưng source thiếu thông tin" — cần thêm điều kiện gì vào code check?
4. Release gate cho RAG nên khác thế nào so với case triage hay routing? (Gợi ý: có "không biết" accuracy)
5. Có cần domain expert không? Tại sao có / không?

---

## 12. Workbook

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

**Trả lời của bạn:**

> Unit of Work là: **một câu hỏi của user về thông tin địa điểm → AI retrieve từ RAG database và trả lời bằng ngôn ngữ tự nhiên**.
>
> Output được dùng trực tiếp bởi user đang lập kế hoạch du lịch — họ thường ra quyết định ngay (có đến không, có book vé không) dựa trên câu trả lời này.
>
> Nếu sai — đặc biệt là hallucination về giá vé hoặc giờ mở cửa: user đến nơi và thấy giá khác, hoặc tệ hơn là địa điểm đóng cửa. Mất tin tưởng vào TraVy sau một lần như vậy là rất khó lấy lại.

### 2. Quality Question

**Trả lời của bạn:**

> **AI có trả lời chính xác dựa trên nội dung source document, và biết nói "không có thông tin" thay vì fabricate khi source không đủ không?**
>
> Phần "biết không biết" quan trọng không kém phần "trả lời đúng": một AI RAG luôn đưa ra câu trả lời trông tự nhiên nhưng đôi khi sai còn nguy hiểm hơn một AI thường xuyên thừa nhận giới hạn. User sẽ mất cảnh giác và tin câu trả lời mà không kiểm tra lại.

### 3. Output Contract tối thiểu

**Trả lời của bạn:**

> - `query_id` (string): định danh để trace kết quả về request khi eval batch.
> - `retrieved_entities` (list): mỗi entity có `entity_id`, `relevance_score`, `source_snippet` (đoạn text thực tế từ document). Trống nếu không tìm thấy.
> - `answer` (string | null): câu trả lời bằng ngôn ngữ tự nhiên. Null nếu không tìm thấy entity.
> - `answer_grounded` (boolean): AI tự đánh giá có phải toàn bộ claim trong answer đều có thể trace về `source_snippet` không.
> - `uncertainty_flag` (boolean): true khi (a) entity tìm thấy nhưng source thiếu thông tin cụ thể, hoặc (b) thông tin có thể đã lỗi thời.
> - `uncertainty_reason` (string | null): giải thích cụ thể khi `uncertainty_flag = true`.
> - `ambiguity_flag` (boolean): true khi query có thể khớp nhiều entity mà AI không chọn được một.
> - `source_label` (string | null): ghi chú nguồn và thời điểm cập nhật hiển thị cho user (ví dụ: "database TraVy · 03/2026").

### 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Tất cả `entity_id` trong `retrieved_entities` tồn tại trong DB | ✓ | | | | Lookup key-value thuần túy — entity không trong DB là bug nghiêm trọng ở tầng retrieval |
| `answer` là null khi `retrieved_entities` rỗng | ✓ | | | | Rule cứng: không có source thì không được có answer — code enforce structural constraint |
| `uncertainty_flag = true` khi `source_snippet` không chứa thông tin số cụ thể (giá, giờ) mà answer có | ✓ | | | | Pattern match số trong answer vs số trong source — nếu answer có số nhưng source không có → flag |
| `ambiguity_flag = true` khi `retrieved_entities` có ≥ 2 items với `relevance_score` trong khoảng nhau ≤ 0.1 | ✓ | | | | Điều kiện số học trên relevance scores — deterministic |
| Mọi claim cụ thể trong `answer` có thể trace về nội dung `source_snippet` | | ✓ | | | Đây là core hallucination check — cần LLM đọc cả answer lẫn source và phán xét "có phần nào answer bịa không?" — code không đọc hiểu ngữ nghĩa được |
| Answer có thật sự trả lời đúng câu hỏi của user không? | | ✓ | | | Relevance check — answer grounded trong source nhưng vẫn có thể không trả lời đúng câu hỏi |
| Khi `uncertainty_flag = true`, explanation có đủ rõ để user hiểu cần làm gì không? | | ✓ | | | Chất lượng của uncertainty warning cần đọc hiểu ngữ nghĩa |
| Spot-check chất lượng answer cho top entities phổ biến nhất | | | ✓ | | Travel ops hoặc local xác nhận answer về địa điểm nổi tiếng (Bà Nà, Hội An, Hồ Hoàn Kiếm) |

### 5. Kiểm tra tự động bằng code

- Kiểm tra: Khi `retrieved_entities` rỗng, `answer` phải là null (không được có nội dung).
  Vì sao nên giao cho code: Rule cứng không thương lượng — answer không có source là hallucination theo định nghĩa.

- Kiểm tra: Mọi `entity_id` trong `retrieved_entities` tồn tại trong entity DB.
  Vì sao nên giao cho code: Lookup key-value — entity không trong DB là bug tầng retrieval, không phải tầng AI judgment.

- Kiểm tra: `uncertainty_flag = true` khi `answer` chứa số cụ thể (giá, giờ) nhưng `source_snippet` không chứa số đó.
  Vì sao nên giao cho code: Pattern match regex để tìm số trong answer, so sánh với source_snippet — nếu không khớp thì phải flag. Đây là proxy tốt để catch hallucination số học mà không cần LLM.

- Kiểm tra: `ambiguity_flag = true` khi có ≥ 2 entities với `relevance_score` khác nhau ≤ 0.1.
  Vì sao nên giao cho code: Điều kiện số học trên scores — code xác định chính xác hơn LLM khi nào threshold bị vi phạm.

- Kiểm tra: `source_label` không null khi `answer` không null.
  Vì sao nên giao cho code: Rule trình bày bắt buộc — mọi answer phải có nhãn nguồn để user biết thông tin từ đâu.

- Kiểm tra: Khi `uncertainty_flag = true`, `uncertainty_reason` không được null.
  Vì sao nên giao cho code: Structural constraint — nếu có flag thì phải có explanation, không để user thấy cảnh báo mà không hiểu vì sao.

### 6. Tiêu chí chấm bằng LLM

- Tiêu chí: Mọi claim cụ thể trong `answer` (giá, giờ, địa chỉ, mô tả) có thể trace về nội dung `source_snippet` không?
  Vì sao code không bắt tốt: Đây là core hallucination detection — LLM judge đọc cả answer và source rồi hỏi "có phần nào answer viết nhưng source không nói không?" Code chỉ match số học; ngữ nghĩa ("mô tả thêm" có grounded không) cần LLM.

- Tiêu chí: Answer có thật sự trả lời đúng câu hỏi của user không? (Ví dụ: hỏi giờ mở cửa, answer có cho biết giờ cụ thể không?)
  Vì sao code không bắt tốt: Relevance check — answer có thể hoàn toàn grounded trong source nhưng vẫn trả lời sai hướng câu hỏi. Cần LLM đọc cả câu hỏi lẫn answer để đánh giá.

- Tiêu chí: Khi `uncertainty_flag = true`, `uncertainty_reason` có đủ rõ ràng và actionable để user biết cần làm gì không?
  Vì sao code không bắt tốt: Chất lượng explanation là judgment call — "thông tin có thể đã cũ" vs "vui lòng gọi điện xác nhận trước khi đến" có cùng ý nghĩa nhưng khác nhau về tính hữu ích cho user.

- Tiêu chí: Với query về nhiều entities (ambiguity), AI có trình bày các options đủ rõ để user phân biệt và chọn không?
  Vì sao code không bắt tốt: Chất lượng của multi-option presentation cần đọc hiểu — code biết có nhiều entities nhưng không biết cách trình bày có helpful không.

### 7. Human / Expert Review

**Trả lời của bạn:**

> **Không cần domain expert.** Lý do: ground truth của RAG là source document trong database — mọi câu trả lời đúng/sai đều có thể xác định bằng cách so sánh answer với source. Đây không phải vấn đề taxonomy chuyên môn (như case y tế) mà là vấn đề grounding.
>
> **Human review áp dụng cho:** travel ops hoặc local spot-check chất lượng answer cho top 20 entities phổ biến nhất của TraVy (Bà Nà Hills, Hội An, Hồ Hoàn Kiếm, Bảo tàng Chăm...). Họ không cần chứng chỉ — chỉ cần biết thực tế địa điểm để phát hiện khi answer "nghe có vẻ đúng nhưng thực tế không như vậy".
>
> **Khi nào human review?** Trong pilot, sample ngẫu nhiên 15% tổng queries để spot-check. Sau khi launch, ongoing review 5% mỗi tuần cho những query có `uncertainty_flag = true`.

**Không áp dụng domain expert.** Human review travel ops là đủ cho loại vấn đề này.

#### 7A. Màn hình cho Domain Expert

Không áp dụng.

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

### 8. Release Gate

**Điều kiện chặn tuyệt đối:**
- Answer khi `retrieved_entities` rỗng > 0% → block ngay (AI đang fabricate hoàn toàn từ training knowledge).
- Schema violation bất kỳ (missing required fields) → block.
- `source_label` null khi `answer` không null > 0% → block (user không biết thông tin từ đâu để tin hay không tin).

**Ngưỡng chất lượng tối thiểu (LLM judge):**
- Hallucination rate (có ít nhất 1 claim không grounded trong source): ≤ 3%.
  Lý do chọn 3%: travel info sai có thể gây phiền ngay lập tức (đến nơi sai giờ/giá). Không khắt khe bằng y tế nhưng không thể để cao hơn.
- "Không biết" accuracy: ≥ 95% — khi entity thật sự không có trong DB, AI phải nói không biết đúng ít nhất 95% lần.
  Lý do: đây là defense chính chống hallucination. Nếu AI hay "đoán" khi không có source, hallucination rate sẽ cao ở mọi nơi.
- Answer relevance (trả lời đúng câu hỏi): ≥ 90%.
- Human spot-check pass (top entities): ≥ 85%.

**Trước lần deploy đầu tiên:**
- Review thủ công 30 queries trải đều 3 loại: entity có trong DB (10 queries), entity không có trong DB (10 queries), entity có trong DB nhưng source thiếu thông tin (10 queries).

### 9. Kế hoạch chạy thử và dự toán chi phí

**Giả định giá API:**
- Model judge: Claude Haiku 4.5 — $0.80/M input tokens, $4.00/M output tokens (nguồn: trang giá Anthropic, tháng 6/2026)
- Mỗi call judge: pass query + source_snippet + answer để LLM judge check grounding ~800 input + 200 output ≈ $0.00064 + $0.00080 = **$0.00144/call**

**Quy mô pilot:**
- 75 queries × 40 lần chạy = **3,000 total calls**
  - 30 queries happy path / stale info / ambiguous (entity có trong DB)
  - 30 queries entity không có trong DB (hallucination risk cao nhất)
  - 15 queries edge cases (unsafe query, ambiguous với nhiều options)
- Chi phí API: 3,000 × $0.00144 ≈ **~$4.32**

**Giờ công:**
- PM / thiết kế eval: 3h (thiết kế LLM judge prompt cho hallucination check — phần quan trọng nhất)
- Ops / kỹ thuật: 5h (chuẩn bị mock entity DB cho pilot, setup eval harness, chạy batch)
- Human review (travel ops): 3h (spot-check 75 queries trải đều 3 loại)
- **Tổng: ~11h**

**Tổng chi phí pilot:** ~$4–5 (API) + ~11h nhân công.

Giá API lấy từ trang pricing chính thức của Anthropic cho Claude Haiku 4.5. Case này ít tốn chi phí API hơn Case 4 vì input ngắn hơn (câu hỏi đơn + snippet), nhưng cần thiết kế judge prompt cẩn thận nhất trong cả 5 cases — một judge prompt viết sai cho hallucination detection sẽ đánh giá "answer có vẻ đúng" thay vì "answer có grounded trong source không". Đây là rủi ro chính cần kiểm soát trước khi ship RAG feature của TraVy.
