# Case 4 - TraVy: Lập lịch trình du lịch (Itinerary Generation)

## Mục tiêu

Case này lấy bối cảnh từ **dự án TraVy** — AI travel assistant của nhóm — và tập trung vào một trong những tính năng cốt lõi: sinh lịch trình du lịch tự động.

Điểm thú vị của case này là output không phải một field đơn giản mà là **một cấu trúc nhiều tầng** (nhiều ngày, nhiều địa điểm mỗi ngày). Điều đó buộc người thiết kế eval phải:

- Biết cách decompose output phức tạp thành các Unit of Work nhỏ hơn để đánh giá
- Phân biệt những gì code có thể kiểm tra được (entity tồn tại, trình tự hợp lệ) với những gì cần LLM hoặc travel expert đánh giá (lịch trình có hợp lý không, có phản ánh đúng sở thích không)
- Đặt release gate cho một output mà không thể có "đáp án đúng duy nhất"

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

TraVy là AI travel assistant giúp người dùng lập kế hoạch du lịch tại Việt Nam.

Khi user nhập yêu cầu (điểm đến, số ngày, sở thích, ngân sách), TraVy:

- Tra cứu entities từ vector database (địa điểm tham quan, nhà hàng, khu vực)
- Xây dựng lịch trình theo từng ngày
- Phân bổ địa điểm hợp lý về địa lý và thời gian
- Gợi ý nhà hàng và hoạt động phù hợp với sở thích và ngân sách

Output cuối cùng là lịch trình chi tiết hiển thị cho user trên giao diện TraVy.

---

## 2. Workflow logic (ASCII)

```text
User nhập yêu cầu:
- điểm đến
- số ngày
- sở thích (biển / núi / ẩm thực / văn hóa / mua sắm)
- ngân sách (thấp / trung bình / cao)
    ↓
AI phân tích yêu cầu và extract preferences
    ↓
AI tra cứu RAG database:
- địa điểm tham quan phù hợp
- nhà hàng theo budget và sở thích ẩm thực
- khu vực địa lý
    ↓
AI xây dựng lịch trình:
- phân bổ địa điểm theo ngày
- sắp xếp theo địa lý (tránh zigzag)
- ước tính thời gian tham quan
- chèn bữa ăn mỗi buổi trưa và tối
    ↓
Output: Lịch trình chi tiết N ngày
    ↓
User xem lại, có thể chỉnh sửa hoặc yêu cầu thay đổi
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+------------------------------------------------------------------+
| TraVy — Lịch trình du lịch của bạn                               |
+------------------------------------------------------------------+
| Đà Nẵng · 3 ngày · Ngân sách: Trung bình · Thích: Biển, Ẩm thực |
|------------------------------------------------------------------|
| NGÀY 1 — Trung tâm & Bãi biển                                    |
| 08:00  Cầu Rồng                         ~1h · Miễn phí           |
| 09:30  Bảo tàng Chăm                    ~1.5h · 60.000đ          |
| 12:00  Cơm gà bà Buội (nhà hàng)        ~1h · ~80.000đ/người    |
| 14:00  Bãi biển Mỹ Khê                  ~2.5h · Miễn phí         |
| 19:00  Bé Mặn (hải sản)                 ~1.5h · ~200.000đ/người  |
|------------------------------------------------------------------|
| NGÀY 2 — Bán đảo Sơn Trà                                        |
| ...                                                               |
|------------------------------------------------------------------|
| [Chỉnh sửa lịch] [Thêm địa điểm] [Xem bản đồ] [Lưu lịch trình] |
+------------------------------------------------------------------+
```

---

## 4. Input mẫu

```json
{
  "destination": "Đà Nẵng",
  "num_days": 3,
  "preferences": ["biển", "ẩm thực địa phương"],
  "budget": "medium",
  "travel_dates": "2026-07-10 to 2026-07-12"
}
```

---

## 5. Business rules / operational rules

- Mỗi ngày không vượt quá 5–6 địa điểm (tránh overload).
- Các địa điểm trong cùng một buổi (sáng / chiều) phải gần nhau về địa lý — không zigzag qua lại giữa hai khu vực xa.
- Phải có ít nhất 1 nhà hàng cho bữa trưa và 1 cho bữa tối mỗi ngày.
- Không được gợi ý địa điểm đóng cửa vào ngày/giờ trong lịch trình.
- Ngân sách phải được phản ánh trong gợi ý (budget thấp → tránh nhà hàng cao cấp).
- Không được bịa địa điểm không có trong database — mọi entity phải có `entity_id` hợp lệ.
- Không được lặp lại cùng một địa điểm trong toàn bộ lịch trình.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

User nhập:

```text
Tôi muốn đi Đà Nẵng 3 ngày, thích biển và ăn hải sản, ngân sách tầm trung.
```

### Workflow ASCII

```text
User nhập preferences
    ↓
AI extract: destination=Đà Nẵng, days=3, biển+hải sản, budget=medium
    ↓
AI tra cứu RAG: lấy top entities theo tag biển, hải sản, Đà Nẵng
    ↓
AI phân bổ: Ngày 1 trung tâm + bãi biển, Ngày 2 Sơn Trà, Ngày 3 Ngũ Hành Sơn / Hội An half-day
    ↓
AI chèn nhà hàng hải sản phù hợp budget vào mỗi ngày
    ↓
Output: lịch trình 3 ngày hiển thị trên UI
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------+
| Ngày 1 — Trung tâm Đà Nẵng & Bãi biển Mỹ Khê                    |
| 08:00  Cầu Rồng          ~1h    Miễn phí                         |
| 09:30  Bảo tàng Chăm     ~1.5h  60.000đ                          |
| 12:00  Cơm gà bà Buội    ~1h    ~80.000đ                         |
| 14:00  Bãi biển Mỹ Khê   ~2.5h  Miễn phí                         |
| 19:00  Nhà hàng hải sản Bé Mặn  ~1.5h  ~200.000đ                 |
+------------------------------------------------------------------+
| Ngày 2 — Bán đảo Sơn Trà                                         |
| 07:30  Chùa Linh Ứng     ~1h    Miễn phí                         |
| 09:00  Đỉnh Bàn Cờ       ~2h    30.000đ                          |
| 12:30  Nhà hàng Sơn Trà  ~1h    ~150.000đ                        |
| 14:30  Bãi biển Tiên Sa  ~2h    Miễn phí                         |
| 19:00  Phố hàng rong Đà Nẵng    ~1.5h  ~100.000đ                 |
+------------------------------------------------------------------+
```

---

## 7. Seed cases

### Seed A - Happy path

- Input: 3 ngày Đà Nẵng, thích biển và ẩm thực địa phương, ngân sách trung bình.
- Kỳ vọng: lịch trình hợp lý về địa lý, đủ nhà hàng, phản ánh đúng sở thích.

### Seed B - Conflicting preferences

- Input: thích cả biển (Mỹ Khê, phía Đông) lẫn núi (Bà Nà Hills, phía Tây) trong cùng 1 ngày.
- Kỳ vọng: AI phải phân bổ sang 2 ngày khác nhau hoặc chọn một và giải thích lý do, không nhồi cả hai vào 1 ngày gây zigzag.

### Seed C - Budget mismatch

- Input: ngân sách thấp nhưng sở thích gồm nhiều địa điểm có phí cao (Bà Nà Hills ~750.000đ).
- Kỳ vọng: AI phải cảnh báo hoặc điều chỉnh gợi ý phù hợp ngân sách, không lặng lẽ gợi ý vượt budget.

### Seed D - Sparse database

- Input: điểm đến ít phổ biến, ít entity trong DB (ví dụ huyện nhỏ tại miền Trung).
- Kỳ vọng: AI không bịa địa điểm, thay vào đó gợi ý ít ngày hơn hoặc mở rộng ra địa điểm lân cận có trong DB.

### Seed E - Hallucination risk

- Input: user nhắc đến một địa điểm không có trong DB ("Tôi muốn đi thăm resort XYZ").
- Kỳ vọng: AI không tự thêm "resort XYZ" vào lịch trình, phải thông báo không tìm thấy và gợi ý thay thế.

---

## 8. Mock outcome để soi

Giả sử AI trả về lịch trình ngày 1 như sau:

```text
Ngày 1:
08:00  Chùa Linh Ứng (Sơn Trà, phía Đông Bắc)
10:30  Bà Nà Hills (phía Tây, cách Sơn Trà ~45km)
14:00  Bãi biển Mỹ Khê (trở về phía Đông)
19:00  Nhà hàng trên đỉnh Bà Nà (phía Tây)
```

Kết quả này trông "đầy đủ" nhưng có lỗi nghiêm trọng:

- Di chuyển Sơn Trà → Bà Nà → Mỹ Khê → Bà Nà trong 1 ngày: ~3–4 tiếng di chuyển thuần túy.
- User sẽ kiệt sức trên đường thay vì tham quan.
- AI vi phạm rule địa lý nhưng schema vẫn hợp lệ — code không bắt được lỗi này.

---

## 9. Bạn phải đề xuất thêm 5 Dataset Edge Cases

1. Happy path:
   - Input: 3 ngày Đà Nẵng, sở thích biển + ẩm thực, ngân sách trung bình, mọi entity có trong DB.
   - Kỳ vọng: lịch trình địa lý hợp lý, đủ bữa ăn, không bịa entity.
   - Bắt failure: Baseline — đảm bảo AI hoạt động đúng trong điều kiện lý tưởng.

2. Conflicting geography:
   - Input: user muốn thăm Hội An (30km về phía Nam) và Bà Nà Hills (30km về phía Tây) trong cùng một buổi sáng.
   - Kỳ vọng: AI phân bổ sang 2 ngày khác nhau, không nhồi cả hai vào 1 buổi.
   - Bắt failure: AI có check khoảng cách địa lý giữa các điểm liên tiếp không?

3. Sparse database / unknown destination:
   - Input: điểm đến ít entity trong DB, user yêu cầu 5 ngày.
   - Kỳ vọng: AI gợi ý ít ngày hơn hoặc mở rộng khu vực lân cận, không bịa entity.
   - Bắt failure: AI có hallucinate địa điểm không tồn tại trong DB khi thiếu dữ liệu không?

4. Budget mismatch:
   - Input: ngân sách thấp nhưng sở thích tự nhiên chọn địa điểm cao cấp.
   - Kỳ vọng: AI cảnh báo hoặc tự điều chỉnh gợi ý về budget tier phù hợp.
   - Bắt failure: AI có lặng lẽ gợi ý vượt ngân sách mà không cảnh báo không?

5. Regression case:
   - Input: 3 ngày Đà Nẵng kết hợp thăm Hội An — trước đây AI xếp Hội An sáng và Sơn Trà chiều cùng ngày (zigzag 60km).
   - Kỳ vọng: Hội An và Sơn Trà nằm ở 2 ngày khác nhau.
   - Bắt failure: sau khi fix địa lý routing, đảm bảo không bị regression sau mỗi lần cập nhật prompt.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết RAG pipeline thật,
- viết routing algorithm thật,
- làm full dataset địa điểm,
- code full system.

Cần làm:

- xác định Unit of Work đủ nhỏ cho một output phức tạp nhiều tầng,
- viết Quality Question phù hợp,
- đề xuất Output Contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human / expert,
- đặt release gate hợp lý khi không có "đáp án đúng duy nhất",
- đề xuất 5 edge cases,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 4?

Đây là case scaffold trung bình. Điểm khó là output có nhiều tầng — hãy bắt đầu bằng cách tự hỏi:

> "Tôi có thể eval toàn bộ lịch trình 3 ngày như một Unit of Work không, hay nên chọn lát cắt nhỏ hơn?"

Nên làm theo thứ tự:

1. Quyết định Unit of Work: toàn bộ trip hay từng ngày?
2. Từ mock outcome sai, xác định loại lỗi nào code bắt được và loại nào cần LLM.
3. Chốt Output Contract — chú ý cần field nào để eval địa lý.
4. Điền Eval Decision Map — nhấn mạnh vào ranh giới code/LLM.
5. Nghĩ xem travel expert có cần thiết không và vì sao.

---

## 12. Workbook

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

**Trả lời của bạn:**

> Unit of Work tôi chọn là: **một yêu cầu lập lịch (destination + num_days + preferences + budget) → AI tạo itinerary hoàn chỉnh gồm danh sách địa điểm, nhà hàng và khung giờ cho từng ngày**.
>
> Mặc dù output có nhiều tầng (N ngày × M địa điểm/ngày), tôi chọn đánh giá ở cấp độ **toàn bộ trip** vì đây là đơn vị user thật sự nhận và quyết định "dùng được không". Tuy nhiên, các tiêu chí code check sẽ được áp dụng xuống từng ngày và từng địa điểm riêng lẻ.
>
> Output được dùng bởi user trực tiếp để thực hiện chuyến đi. Nếu sai: user theo lịch AI sinh ra → tốn 4 tiếng di chuyển zigzag, không có đủ thời gian tham quan, hoặc tệ hơn là đến địa điểm đóng cửa — mất trải nghiệm và mất trust vào TraVy.

### 2. Quality Question

**Trả lời của bạn:**

> **AI có tạo được lịch trình địa lý hợp lý (không zigzag), phản ánh đúng sở thích và ngân sách của user, và không bịa bất kỳ entity nào ngoài database không?**
>
> Nếu fail về địa lý: user kiệt sức trên đường và mất tin vào TraVy ngay chuyến đi đầu tiên. Nếu fail về hallucination: user đến địa điểm không tồn tại — đây là lỗi nghiêm trọng nhất vì không thể khắc phục khi đang ở Đà Nẵng.

### 3. Output Contract tối thiểu

**Trả lời của bạn:**

> - `trip_id` (string): định danh để nối output với request khi eval batch.
> - `destination` (string): xác nhận AI hiểu đúng điểm đến.
> - `days` (list of day objects): mỗi day có `date`, `theme` (khu vực chính trong ngày), và `slots` (list of time_slot).
> - Mỗi `time_slot` gồm: `start_time`, `entity_id`, `entity_name`, `duration_minutes`, `estimated_cost`, `slot_type` (attraction | meal | transport).
> - `entity_id` (string): bắt buộc có — dùng để code check entity có trong DB không.
> - `budget_match` (boolean): AI tự đánh giá lịch trình có phù hợp ngân sách không.
> - `geographic_zones_per_day` (list of string per day): khu vực địa lý của từng ngày — dùng để eval địa lý mà không cần tính tọa độ.
> - `warnings` (list of string | null): cảnh báo khi có conflict (budget mismatch, entity ít trong DB).

### 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Mọi `entity_id` trong lịch trình tồn tại trong DB | ✓ | | | | Lookup đơn giản — nếu entity_id không có trong DB là hallucination, code phát hiện ngay |
| Không có entity nào lặp lại trong toàn trip | ✓ | | | | String equality check trên list — deterministic |
| Mỗi ngày có ít nhất 1 meal slot buổi trưa và 1 buổi tối | ✓ | | | | Count slot_type=meal theo time range — rule cứng |
| Tổng thời gian các slot trong ngày ≤ 720 phút (12h) | ✓ | | | | Tính tổng duration_minutes — số học thuần túy |
| `geographic_zones_per_day` không có quá 2 khu vực khác nhau trong 1 ngày | ✓ | | | | Count distinct zones per day — bắt zigzag mà không cần tọa độ thật |
| Lịch trình phản ánh đúng sở thích user (biển, núi, ẩm thực...) | | ✓ | | | Cần đọc hiểu semantic giữa preferences và entities được chọn |
| Narrative / theme mỗi ngày có mạch lạc, hợp lý về trải nghiệm | | ✓ | | | Chất lượng lịch trình tổng thể — code không đánh giá được "ngày 1 trung tâm, ngày 2 biển" có tốt hơn đảo ngược không |
| Budget tier của nhà hàng và địa điểm phù hợp ngân sách user | | ✓ | | | Cần match ngữ nghĩa giữa budget=low và "nhà hàng bình dân" — không thể encode thành rule cứng |
| Spot-check chất lượng tổng thể cho high-demand destinations | | | ✓ | | Travel ops hoặc người địa phương review lịch trình trên Đà Nẵng, Hà Nội, TP.HCM |

### 5. Kiểm tra tự động bằng code

- Kiểm tra: Tất cả `entity_id` trong output tồn tại trong entity database.
  Vì sao nên giao cho code: Lookup key-value thuần túy — entity_id không có trong DB là hallucination, cần block ngay.

- Kiểm tra: Không có `entity_id` nào xuất hiện 2 lần trong cùng một trip.
  Vì sao nên giao cho code: String equality check trên list — không cần ngữ nghĩa.

- Kiểm tra: Mỗi ngày có ít nhất 1 slot `slot_type=meal` trong khung 11:00–14:00 và 1 slot trong khung 18:00–21:00.
  Vì sao nên giao cho code: Rule cứng về cấu trúc lịch trình — code đếm chính xác hơn LLM.

- Kiểm tra: Tổng `duration_minutes` của các slot trong mỗi ngày ≤ 720 phút.
  Vì sao nên giao cho code: Tính tổng số học — nếu vượt thì lịch trình không khả thi về thời gian.

- Kiểm tra: Số lượng `distinct geographic_zones` trong mỗi ngày ≤ 2.
  Vì sao nên giao cho code: Bắt zigzag địa lý mà không cần tọa độ GPS — count string values.

- Kiểm tra: `budget_match = true` khi không có entity nào có `estimated_cost` vượt ngưỡng budget tier.
  Vì sao nên giao cho code: So sánh số với ngưỡng định nghĩa sẵn (low/medium/high → giá trần).

- Kiểm tra: `warnings` không null khi `budget_match = false` hoặc số entity trong ngày < 3.
  Vì sao nên giao cho code: Rule bắt buộc có explanation khi có vấn đề — structural check.

- Kiểm tra: `trip_id` không rỗng và khớp request_id đầu vào.
  Vì sao nên giao cho code: Định danh cần để trace khi review batch.

### 6. Tiêu chí chấm bằng LLM

- Tiêu chí: Lịch trình có phản ánh đúng sở thích user không? (ví dụ: thích biển thì phần lớn địa điểm gần biển, thích ẩm thực thì nhà hàng được chú trọng hơn)
  Vì sao code không bắt tốt: Cần map ngữ nghĩa giữa tag sở thích và loại entity — code chỉ check tag cứng, không đánh giá mức độ phản ánh tổng thể.

- Tiêu chí: Theme mỗi ngày có mạch lạc không? (ví dụ: "ngày 1 trung tâm thành phố, ngày 2 thiên nhiên" tốt hơn ngẫu nhiên mỗi ngày)
  Vì sao code không bắt tốt: Narrative quality là judgment call — cần LLM so sánh theme các ngày và đánh giá tính nhất quán.

- Tiêu chí: Budget tier của nhà hàng và địa điểm có phù hợp ngân sách user không?
  Vì sao code không bắt tốt: "Nhà hàng bình dân" vs "nhà hàng cao cấp" không chỉ dựa vào giá tiền cứng — tên và mô tả cần đọc hiểu để phân loại đúng.

- Tiêu chí: Với Seed B (conflicting geography): AI có giải thích tại sao phân bổ sang 2 ngày khác nhau thay vì lặng lẽ tách ra không?
  Vì sao code không bắt tốt: Chất lượng explanation cần đọc hiểu ngữ nghĩa — code chỉ biết có/không có `warnings`, không biết warning đó có đủ rõ không.

- Tiêu chí: Với Seed D (sparse DB): AI có thông báo rõ ràng về giới hạn dữ liệu thay vì im lặng bỏ qua hoặc bịa thêm không?
  Vì sao code không bắt tốt: Chất lượng thông báo và tính trung thực cần LLM đọc và đánh giá mức độ rõ ràng.

### 7. Human / Expert Review

**Trả lời của bạn:**

> **Ai cần review:** Travel ops hoặc người địa phương quen thuộc với các điểm đến phổ biến (Đà Nẵng, Hà Nội, TP.HCM, Hội An). Không bắt buộc phải là travel agent chuyên nghiệp.
>
> **Vì sao:** Họ biết ngay khi lịch trình có vấn đề về địa lý hoặc thực tế (ví dụ: "Chùa Linh Ứng buổi sáng + Bà Nà buổi chiều là không khả thi") — kiến thức này không thể encode thành rule cứng mà phải có người biết địa lý thực địa xác nhận.
>
> **Review những case nào:** (1) lịch trình cho top 5 điểm đến phổ biến nhất của TraVy; (2) tất cả cases có `warnings` không rỗng; (3) sample ngẫu nhiên 10% tổng pilot.
>
> **Không bắt buộc travel agent chuyên nghiệp** vì lịch trình TraVy là gợi ý, không phải booking chính thức — người địa phương hoặc travel ops nội bộ đủ thẩm quyền xác nhận chất lượng cơ bản.

**Không áp dụng domain expert chuyên sâu.** Người địa phương hoặc travel ops đủ thẩm quyền review — không cần chứng chỉ travel agent.

#### 7A. Màn hình cho Domain Expert

Không áp dụng.

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

### 8. Release Gate

**Điều kiện chặn tuyệt đối:**
- Hallucinated entity (entity_id không có trong DB) > 0% → block ngay, đây là lỗi nghiêm trọng nhất.
- Schema violation bất kỳ → block.
- Lịch trình có > 2 geographic zones trong 1 ngày > 5% cases → block (zigzag địa lý quá phổ biến).

**Ngưỡng chất lượng tối thiểu:**
- Entity DB match rate: 100% (không thương lượng).
- Preference alignment (LLM judge): ≥ 85%.
- Budget tier accuracy (LLM judge): ≥ 85%.
- Narrative coherence (LLM judge): ≥ 80%.
- Human spot-check pass rate (top destinations): ≥ 80%.

**Trước lần deploy đầu tiên:**
- Tối thiểu review thủ công 20 lịch trình cho top 5 điểm đến phổ biến nhất của TraVy.

### 9. Kế hoạch chạy thử và dự toán chi phí

**Giả định giá API:**
- Model judge: Claude Haiku 4.5 — $0.80/M input tokens, $4.00/M output tokens (nguồn: trang giá Anthropic, tháng 6/2026)
- Mỗi call judge: cần pass cả itinerary output (dài hơn các case trước) ~1,500 input + 300 output ≈ $0.00120 + $0.00120 = **$0.00240/call**

**Quy mô pilot:**
- 50 itineraries × 30 lần chạy = **1,500 total calls**
- Chi phí API: 1,500 × $0.00240 ≈ **~$3.60**

**Giờ công:**
- PM / thiết kế eval: 4h (thiết kế judge prompt cho preference matching + geographic check)
- Ops / kỹ thuật: 6h (build mock entity DB cho pilot, eval harness, chạy batch)
- Human review (travel ops / local): 4h (review 50 itineraries cho top destinations)
- **Tổng: ~14h**

**Tổng chi phí pilot:** ~$3–4 (API) + ~14h nhân công.

Giá API lấy từ trang pricing chính thức của Anthropic cho Claude Haiku 4.5. Chi phí API rất thấp (~$4) nhưng 4h human review là thành phần quan trọng không thể bỏ qua — đây là loại output mà automated eval không thể bắt hết lỗi địa lý thực tế. Với 50 itineraries bao phủ các lát cắt happy path, conflicting preferences, sparse DB, và regression, team có đủ bằng chứng để trả lời "TraVy có tạo được lịch trình đáng tin cậy không?" trước khi ship cho user thật.
