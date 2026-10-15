MSSV: 202602528

# Lab 11 · Scenario Forge — báo cáo

Thị trấn của bạn (`meta.json: _run.town`): D

## 1. Phát lại log có trung thực không? (bước A)

### Bảng số liệu từ `evidence/replay.json`:

| Chỉ số | Giá trị | Chi tiết tiếp xúc (`contact`) |
| :--- | :--- | :--- |
| Tổng số kịch bản khảo sát (`n`) | 400 | — |
| Số log người lái tự gây va chạm bị bỏ (`dropped_human_crash_logs`) | 0 | Giữ lại đủ 400 log hợp lệ (`n_used = 400`) |
| Va chạm khi planner chạy trên **log phát lại** (`replay`) | **108** | **`rear`: 104**, **`side`: 4**, `front`: 0 |
| Va chạm khi planner chạy trong **thế giới phản ứng** (`reactive`) | **0** | Rỗng `{}` (không có va chạm nào) |
| Va chạm xảy ra ở cả hai môi trường (`crash_in_both`) | **0** | Không có kịch bản nào va chạm đồng thời |

### Giải thích:
- **Vì sao phần lớn va chạm của "phát lại" là `rear`:** Trong log phát lại, các phương tiện xung quanh (đặc biệt là xe bám đuôi TG) di chuyển thụ động theo đúng tọa độ thời gian đã ghi từ trước của người lái thật. Khi planner của xe ego lái thận trọng hơn (phanh sớm hơn hoặc giảm tốc nhường đường), xe TG trong log phát lại không có cơ chế nhận biết điều đó để phanh theo, dẫn đến việc húc thẳng vào đuôi xe ego từ phía sau (`contact: rear` chiếm 104/108 vụ, tức 96.3%).
- **Vì sao đó là va chạm giả:** Trong thế giới thực (tương ứng với thế giới phản ứng - reactive), xe phía sau có tài xế quan sát và sẽ lập tức phanh giảm tốc khi thấy xe trước phanh. Minh chứng là khi đưa chính planner này vào thế giới phản ứng tương tác, số va chạm giảm về **0 vụ**. Các vụ tai nạn bị húc đuôi trong log phát lại hoàn toàn là ảo ảnh do mô hình phát lại "mở vòng" (open-loop replay) gây ra.
- **Phân tích một kịch bản trong `replay_only`:** Trích kịch bản `scenario: 1` (`replay.json: replay_only[0]`):
  - Khi phát lại log, xe ego bị xe sau đâm vào đuôi (`contact: "rear"`).
  - Trong khi đó, trong thế giới phản ứng, xe sau nhận diện được sự giảm tốc của ego và phanh theo, khoảng thời gian va chạm nhỏ nhất là `min_ttc_reactive = 2.71 s` và không hề có va chạm nào xảy ra.

---

## 2. Kịch bản bất khả thi (bước B)

### Bốn luật trong `is_feasible` và căn cứ vật lý:
1. **Gia tốc ngang khi chuyển làn ($a_{\text{lat}} \le \mu g$):** Quá trình chuyển làn ngang $3.5\text{ m}$ theo hàm cosine trong `cutDur` giây có gia tốc ngang cực đại $a_{\text{lat}} = (\pi / \text{cutDur})^2 \times 3.5 / 2$. Theo định luật ma sát Coulomb, lực bám ngang cực đại của lốp xe là $F_{\text{lat}} = \mu m g$, do đó gia tốc ngang tối đa không thể vượt quá $\mu g$. Kịch bản có $a_{\text{lat}} > \mu g$ xe sẽ bị trượt văng lật, không thể xảy ra trên đường thực.
2. **Gia tốc phanh sau nhập làn ($\text{cutBrake} \le \mu g$):** Gia tốc giảm tốc dọc của xe cắt làn sau khi vào làn không thể vượt quá ma sát mặt đường $\mu g$.
3. **Giới hạn tốc độ người đi bộ ($\text{pedV} \le 2.5\text{ m/s}$):** Vận tốc đi bộ bình thường của con người là 1.2 – 1.5 m/s. Vận tốc trên 2.5 m/s (~9 km/h) là chạy nước rút, không phù hợp với người đi bộ băng qua đường trong phố.
4. **Giới hạn vận tốc xe cắt làn ($0 < \text{cutV} \le 22.0\text{ m/s}$):** Xe cắt làn phải chuyển động tịnh tiến về phía trước ($> 0$) và tốc độ không vượt quá 22 m/s (~79.2 km/h, giới hạn tốc độ đô thị an toàn).

### Thống kê kịch bản bất khả thi trong `hunt.json`:
- `hunt bus_bypass`: loại bỏ 3 kịch bản bất khả thi.
- `hunt cut_in`: loại bỏ 0 kịch bản.
- `hunt crosswalk_block`: loại bỏ **12 kịch bản bất khả thi** (nhiều nhất).
- `hunt rear_ended`: loại bỏ 3 kịch bản bất khả thi.

- **Vì sao CEM bị kéo về phía đó:** Thuật toán Cross-Entropy Method tối ưu hóa dựa trên hàm điểm nguy hiểm (danger score). Để tạo ra tình huống ép xe ego phanh gấp hoặc vi phạm vạch qua đường ở mức độ cao nhất, điểm nguy hiểm sẽ tăng vọt khi người đi bộ lao qua đường với tốc độ cực nhanh (`pedV` vượt quá 2.5 m/s) hoặc xe chuyển làn với thời gian cực ngắn khiến gia tốc ngang vượt quá ma sát lốp. Do đó nếu không có bộ lọc `is_feasible`, CEM sẽ tự nhiên hội tụ về các kịch bản "phi vật lý" này vì chúng dễ đạt điểm số nguy hiểm giả tạo cao nhất.
- **Hệ quả nếu đếm chúng là va chạm:** Xác suất tai nạn sẽ bị **thổi phồng nghiêm trọng (overestimate)** và sai lệch bản chất. Planner sẽ bị điều chỉnh quá thận trọng (over-conservative) để chống lại những tai nạn ma thuật không bao giờ tồn tại ngoài đời thực.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

### Số liệu từ `evidence/mc.json`:
- Số lần chạy: $n = 3\,000$
- Số vụ va chạm: $k = 6$
- Ước lượng điểm: $\hat{p} = 6 / 3000 = 2.00 \times 10^{-3}$ ($0.20\%$)
- Khoảng tin cậy 95% Wilson: $[9.17 \times 10^{-4},\; 4.36 \times 10^{-3}]$ (tức $[0.092\%,\; 0.436\%]$)
- Phân loại kiểu va chạm: 100% thuộc kiểu `bus_bypass` (6/6 vụ).

### Tính toán số lần chạy cần thiết:
Bán độ rộng khoảng tin cậy tương đối $\le 20\%$ của $p$:
$$\text{Margin} = z \sqrt{\frac{p(1-p)}{n}} \le 0.20 \times p \implies n \ge \frac{z^2 (1-p)}{0.04 p} \approx \frac{1.96^2}{0.04 \times p} \approx \frac{96.04}{p}$$
Với $\hat{p} = 2.00 \times 10^{-3}$:
$$n \approx \frac{96.04}{0.002} \approx \mathbf{48\,020 \text{ lần chạy}}$$
Cần khoảng 48.000 đến 50.000 lần chạy để khoảng tin cậy thu hẹp về $\pm 20\%$.

### Kiểu thất bại chưa từng xuất hiện:
- Các kiểu `cut_in`, `crosswalk_block`, `rear_ended`, và `other` đều có $k = 0$ va chạm trong 3 000 lần chạy (`per_mode: k = 0`).
- **Con số đúng cần báo:** Tuyệt đối **không được báo xác suất bằng 0** (do cỡ mẫu hữu hạn chưa quan sát được). Con số đúng cần báo là **chặn trên khoảng tin cậy 95% Wilson**:
  $$p \le 1.28 \times 10^{-3} \quad (\text{trích từ } \texttt{mc.json: per\_mode: hi = 0.001279})$$
  (tương đương quy tắc số 3: $3/n = 3/3000 = 1.0 \times 10^{-3}$).

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

### Bảng số liệu từ `evidence/estimate.json`:

| Kiểu thất bại | Seed | $p$ | Số va chạm (`hits`) | Kish ESS (`ess`) | `top_share` |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **`bus_bypass`** | 0 | $4.41 \times 10^{-4}$ | 35 | 130.6 | **0.50** |
| | 1 | $4.08 \times 10^{-4}$ | 41 | 120.0 | 0.13 |
| | 2 | $1.99 \times 10^{-4}$ | 32 | 121.1 | 0.16 |
| **`cut_in`** | 0 | 0.0 ($< 2.5 \times 10^{-2}$) | 0 | 150.3 | 0.00 |
| | 1 | 0.0 ($< 2.5 \times 10^{-2}$) | 0 | 111.8 | 0.00 |
| | 2 | 0.0 ($< 2.5 \times 10^{-2}$) | 0 | 132.8 | 0.00 |
| **`crosswalk_block`** | 0 | $1.17 \times 10^{-7}$ | 32 | 107.0 | 0.22 |
| | 1 | $8.76 \times 10^{-7}$ | 39 | 116.1 | **0.58** |
| | 2 | $5.45 \times 10^{-7}$ | 31 | 133.1 | **0.59** |
| **`rear_ended`** | 0 | 0.0 ($< 2.5 \times 10^{-2}$) | 0 | 136.1 | 0.00 |
| | 1 | 0.0 ($< 2.5 \times 10^{-2}$) | 0 | 128.1 | 0.00 |
| | 2 | 0.0 ($< 2.5 \times 10^{-2}$) | 0 | 110.9 | 0.00 |
| **`other`** (từ `mc.json`) | — | 0.0 ($< 1.28 \times 10^{-3}$) | 0 | — | — |
| **TỔNG (`total`)** | 0 | $4.41 \times 10^{-4}$ [0.00, $2.17 \times 10^{-3}$] | — | — | — |
| | 1 | $4.09 \times 10^{-4}$ [$2.02 \times 10^{-4}$, $1.90 \times 10^{-3}$] | — | — | — |
| | 2 | $1.99 \times 10^{-4}$ [$9.54 \times 10^{-5}$, $1.58 \times 10^{-3}$] | — | — | — |

- **Ba seed có đồng ý với nhau không:** Ba seed đồng thuận rất cao về bậc độ lớn của tổng xác suất ($2.0 \times 10^{-4}$ đến $4.4 \times 10^{-4}$, chênh nhau khoảng 2 lần, hoàn toàn bình thường trong lấy mẫu sự kiện hiếm).
- **Có đồng ý với Monte Carlo không:** Con số IS thấp hơn ước lượng Monte Carlo ($\hat{p} = 2.0 \times 10^{-3}$) khoảng 4 – 5 lần. Đây là hiện tượng đã được dự báo trước trong README (Bẫy số 2): đề xuất CEM khi co lại tập trung vào một vùng va chạm cục bộ thường khiến ước lượng IS bị thấp hơn giá trị trung bình toàn cục từ 1.5 đến 4 lần.
- **Nơi `top_share` lớn (> 0.5):** Xuất hiện ở Seed 0 (`bus_bypass`: 0.50), Seed 1 (`crosswalk_block`: 0.58), và Seed 2 (`crosswalk_block`: 0.59).
  - *Giải thích:* `top_share` là tỷ lệ đóng góp của một mẫu có trọng số lớn nhất vào tổng ước lượng ($\max(wy) / \sum wy$). Khi `top_share` > 0.5, điều đó cho thấy hiện tượng suy biến trọng số: một mẫu đơn lẻ mang hơn một nửa giá trị ước lượng, chứng tỏ đề xuất $q$ đã co hẹp lệch khỏi phân phối tự nhiên $p$ ở biên, khiến tỷ lệ trọng số $w = p/q$ của mẫu đó tăng vọt và chi phối con số ước lượng.
- **So sánh ngân sách (`budget`) với Monte Carlo:**
  - Tổng ngân sách IS: `budget = 16 800` lần chạy.
  - Để đo được sự kiện `bus_bypass` ($p \approx 4 \times 10^{-4}$) với độ chính xác tương đương, Monte Carlo cần ít nhất $96 / (4 \times 10^{-4}) \approx \mathbf{240\,000 \text{ lần chạy}}$. Với sự kiện cực hiếm như `crosswalk_block` ($p \approx 10^{-7}$), Monte Carlo cần gần **1 tỷ lần chạy**. Do đó IS tiết kiệm tài nguyên tính toán từ hàng chục đến hàng nghìn lần.
- **Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó:**
  - Lấy từ kết quả Monte Carlo bước C1 (`mc.json: other`, $k = 0, n = 3000$, chặn trên $1.28 \times 10^{-3}$).
  - Không săn `other` vì `other` là tập hợp phần dư không có hàm mục tiêu nguy hiểm (danger score) cụ thể nào để dẫn đường cho thuật toán CEM di chuyển phân phối đề xuất.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

### 1. Kiểu `bus_bypass`:
- **Chuỗi sự kiện:** Xe buýt đỗ ở làn E2 che khuất hoàn toàn tầm nhìn vào góc đường giao lộ. Xe C2 đi chậm qua cạnh xe buýt. Xe ego chạy làn E1 ở vận tốc hành trình ~12 m/s. Do xe buýt chắn tầm nhìn, cảm biến nóc của ego không phát hiện được C2 cho đến khi mũi xe C2 nhô hẳn ra làn E1 ở cự ly quá gần; ego phanh gấp nhưng không kịp triệt tiêu vận tốc và va chạm trực diện/góc vào C2.
- **Ảnh chụp 3D:**
  ![Va chạm lách xe buýt bus_bypass](evidence/shots/bus_bypass.png)

### 2. Kiểu `crosswalk_block`:
- **Chuỗi sự kiện:** Người đi bộ P1 bước qua đường từ lề phía Nam. Do xe buýt đỗ chắn tầm quan sát ban đầu, ego phát hiện P1 muộn. Xe ego phanh khẩn cấp dừng lại hoàn toàn trước người đi bộ, nhưng thân xe đã trôi vào và nằm chắn ngang vạch qua đường của P1 (`event["kind"] == "block"`), buộc người đi bộ phải đứng khựng lại trước thân xe.
- **Ảnh chụp 3D:**
  ![Chặn lối người đi bộ crosswalk_block](evidence/shots/crosswalk_block.png)

### Các kiểu cuộc săn không tìm thấy:
- Cuộc săn `cut_in` và `rear_ended` không tìm thấy va chạm nào thuộc chính kiểu của nó (trong `hunt.json`, cuộc săn `cut_in` chỉ tìm thấy `bus_bypass: 16`, cuộc săn `rear_ended` tìm thấy `bus_bypass: 2, crosswalk_block: 12`).
- **Mức độ tin cậy:** Khá tin cậy rằng trong điều kiện giao thông chuẩn mực của thị trấn D với các xe lái theo IDM có ý thức nhường đường (gap acceptance), xe C2 thông thường luôn chờ khoảng trống đủ an toàn mới chuyển làn nên không gây va chạm cắt làn trực diện; xe sau TG cũng giữ khoảng cách an toàn khi ego không phanh bất thường. Tuy nhiên ở góc độ an toàn xác thực, ta không được khẳng định rủi ro bằng 0 mà phải áp dụng chặn trên 95% lý thuyết $3/(0.2n) = 2.5 \times 10^{-2}$.
- **Nếu chỉ chạy MỘT cuộc săn chung cho mọi kiểu:** Thuật toán CEM sẽ bị **suy biến và co cụm vào duy nhất một kiểu va chạm dễ nhất** (ở đây là `bus_bypass` vốn có xác suất xuất hiện cao nhất cỡ $10^{-3}$). Cuộc săn chung sẽ hoàn toàn bỏ sót các kiểu va chạm hiếm hơn như `crosswalk_block` ($10^{-7}$), khiến chúng ta lầm tưởng rằng planner an toàn tuyệt đối với người đi bộ.

---

## 6. Sửa planner (bước E + 3D)

### Bảng đối sánh từ `evidence/fix.json`:
- **Policy được chọn:** Ứng viên số 6: `{"cutinLat": 2.6}` (tăng ngưỡng nhận diện xe cắt làn từ 2.15 m lên 2.6 m).

| Tập dữ liệu / Chỉ số | Policy gốc (Baseline) | Policy được chọn (Chosen) | Nhận xét |
| :--- | :---: | :---: | :--- |
| **TRAIN** `fail` | $5.61 \times 10^{-4}$ | **$4.77 \times 10^{-7}$** | Giảm rủi ro trên 1.000 lần |
| **TRAIN** `fail_hi` | $1.01 \times 10^{-3}$ | **$1.03 \times 10^{-6}$** | Chặn trên TRAIN cực thấp |
| **TEST** `fail` | $3.91 \times 10^{-4}$ | **$2.23 \times 10^{-7}$** | Giảm rủi ro trên 1.700 lần trên tập kiểm thử độc lập |
| **TEST** `fail_hi_seen` | $6.43 \times 10^{-4}$ | **$4.98 \times 10^{-7}$** | Chặn trên của các kiểu có va chạm |
| **TEST** `fail_hi` (hợp lệ) | $1.92 \times 10^{-3}$ | **$1.28 \times 10^{-3}$** | Chặn trên 95% hợp lệ toàn diện |
| **Quãng đường trung bình** | $217.9\text{ m}$ | **$217.8\text{ m}$** | Bảo toàn **$99.95\%$** tiến độ (giảm chưa đầy $0.1\text{ m} \ll 5\%$) |
| **Số frame phanh gấp** ($a < -3$) | 7.56 frame | **7.56 frame** | Độ êm dịu hành khách hoàn toàn tương đương |

- **Vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật:**
  Vì `fail_hi` trên TEST là chặn trên bảo toàn 95% có tính đến các kiểu `unseen` (không có va chạm nào trong mẫu thử). Khi một kiểu va chạm có 0 vụ va chạm, cận trên thống kê hợp lệ của nó là $3/(0.2n)$ (khoảng $15/n \approx 3.75 \times 10^{-2}$ với $n = 400$) cộng với cận của tập progress $3/3000 \approx 1.0 \times 10^{-3}$. Với ngân sách kiểm thử hữu hạn, không có tập dữ liệu nào chứng minh được một rủi ro đã triệt tiêu dưới $10^{-3}$. `fail_hi` phản ánh giới hạn đo lường của mẫu hữu hạn chứ không phải lỗi của thuật toán.
- **Ảnh chụp 3D có bóng mờ (ghost):**
  ![Bóng mờ xe gốc va chạm trong khi bản sửa dừng an toàn](evidence/shots/fix_ghost.png)
- **Núm chỉnh "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại:**
  - Ví dụ: Giảm tốc độ thận trọng `cautionV` xuống quá thấp (ví dụ 5.0 m/s) hoặc tăng quãng đường thận trọng `cautionRange` lên quá xa (ví dụ 85 m).
  - *Giải thích bằng phản ứng của thế giới:* Khi xe ego chủ động giảm tốc độ quá sớm từ xa trên làn E1 để "thận trọng", nó tạo ra một khoảng trống lớn không cần thiết phía trước. Xe cắt làn C2 ở làn bên cạnh quan sát thấy khoảng trống này sẽ lập tức quyết định chen vào làn sớm hơn. Đồng thời, việc ego phanh chậm bất thường sẽ làm xe bám đuôi TG phía sau không kịp thích ứng, dẫn đến nguy cơ bị đâm đuôi hoặc xung đột luồng giao thông tăng vọt. Môi trường giao thông là hệ thống tương tác đa tác nhân (multi-agent reactive system), sự an toàn đơn phương thái quá của ego sẽ kích thích hành vi nguy hiểm từ các phương tiện khác.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định trong `space.py` ảnh hưởng nhiều nhất đến con số đánh giá:

1. **Khoảng cách khe chấp nhận của xe chuyển làn (`cutGap`):**
   - *Giả định:* Phân phối lognormal với trung vị 10 – 14 m.
   - *Dữ liệu thật thay thế:* Dữ liệu quỹ đạo lái xe tự nhiên từ camera/drone hoặc các bộ dữ liệu lớn (như HighD, Waymo Open Dataset, Argoverse) ghi nhận phân phối khoảng cách thực tế giữa các đầu xe khi tài xế bắt đầu đánh lái nhập làn trong đô thị.
   - *Hướng sai lệch nếu sai:* Nếu ngoài đời thực tài xế liều lĩnh hơn (chấp nhận khe dưới 8 m), mô phỏng sẽ đánh giá thấp (underestimate) nguy cơ va chạm cắt làn.
2. **Tỉ lệ người đi bộ quan sát trước khi bước qua xe buýt đỗ (`pedLook`):**
   - *Giả định:* Xác suất người đi bộ ngó nhìn qua xe buýt là 80% – 90% (thị trấn D là 90%).
   - *Dữ liệu thật thay thế:* Video quan sát camera giao thông tại các trạm xe buýt đô thị phân tích hành vi người đi bộ qua đường (tỷ lệ người nhìn đường so với người cắm đầu đi hoặc nhìn màn hình điện thoại).
   - *Hướng sai lệch nếu sai:* Nếu ngoài đời thực tỷ lệ người đi bộ bất cẩn cao hơn (chỉ 40 – 50% người ngó nhìn), số vụ chặn vạch qua đường và va chạm người đi bộ ngoài đời thực sẽ cao hơn rất nhiều so với mô phỏng.
3. **Hệ số ma sát mặt đường ướt (`wet` và `mu`):**
   - *Giả định:* Xác suất trời mưa làm ướt đường 15% – 30% với $\mu = 0.5$ (so với $\mu = 0.8$ khi khô).
   - *Dữ liệu thật thay thế:* Số liệu khí tượng thủy văn về số ngày mưa trung bình tại thành phố và dữ liệu đo hệ số ma sát động mặt đường thực tế từ máy đo ma sát chuyên dụng (Skid Resistance Tester).
   - *Hướng sai lệch nếu sai:* Nếu mặt đường bị trơn trượt nghiêm trọng hơn do dầu mỡ hay rêu mốc ($\mu < 0.4$), quãng đường phanh thực tế sẽ kéo dài hơn nhiều, khiến tỷ lệ tai nạn thực tế vượt xa con số ước lượng.

---

## 8. Giới hạn

### Ba điều mà xác suất báo cáo KHÔNG phản ánh:
1. **Hình học đường sá và kịch bản tĩnh giản lược:** Thế giới mô phỏng chỉ gồm một đoạn đường thẳng 2 làn mỗi chiều với một trạm dừng xe buýt, không có nút giao ngã tư phức tạp có đèn tín hiệu, bùng binh, đường dốc, góc cua khuất tầm nhìn, hay chướng ngại vật ngẫu nhiên trên mặt đường (nắp cống, rào chắn công trình).
2. **Hành vi tác nhân giao thông theo mô hình IDM lý tưởng hóa:** Các xe khác lái hoàn toàn theo mô hình toán học IDM với các quy tắc nhường đường có trật tự; không mô phỏng các hành vi bất quy tắc ngoài đời thực như tài xế say rượu, tài xế vượt ẩu lấn làn ngược chiều, người đi xe máy luồn lách (đặc thù giao thông Việt Nam).
3. **Mô hình cảm biến nhận thức hoàn hảo:** Cảm biến trên nóc xe giả định nhìn thấy vật thể lập tức với kích thước bounding box chuẩn 100% khi không bị xe buýt che khuất; không có độ trễ nhận thức (perception latency), không có sai số phân lớp mạng AI (false positive/false negative), không bị suy giảm tầm nhìn do mưa lớn hay lóa nắng.

### Hướng nghiên cứu tiếp nếu có thêm một tuần:
Mô hình hóa thêm **độ trễ xử lý nhận thức (perception-to-actuation latency)** và độ nhiễu ngẫu nhiên của cảm biến; mở rộng kịch bản sang mạng lưới giao thông ngã tư đô thị hỗn hợp có sự tham gia của dòng xe máy (motorcycles) luồn lách ở khe hẹp giữa các làn xe.

---

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** Tình huống **Xe đạp ở làn bên cạnh (E2) bất ngờ lạng sang trái lấn vào giữa làn E1 của xe ta** để tránh ổ gà / chướng ngại vật mặt đường khi xe ta đang tiếp cận phía sau ở cự ly gần (9 – 14 m).
  - *Tình huống ngoài đời thực:* Xe hai bánh đi ở làn ngoài/sát lề bất ngờ đánh lái né vũng nước sâu, ổ gà hoặc nắp cống hỏng, tạt ngang đầu ô tô đang chạy tới.
- **Căn cứ:** Planner xe tự lái chỉ sử dụng mô hình IDM dọc theo hành lang làn xe E1. Khi vật cản ban đầu nằm ở làn E2, planner không coi nó là xe dẫn đầu cho đến khi nó lọt vào ngưỡng ngang `cutinLat`. Khi xe đạp lạng nhanh ở cự ly gần (9 – 14 m), planner phát hiện muộn và chỉ phanh theo gia tốc IDM êm dịu $b = 2.0\text{ m/s}^2$, hoàn toàn không đủ để dừng xe trước khi va chạm.
- **Giả định:**
  - `space`: `h0_x: [30.0, 50.0]`, `h0_swerve: [2.4, 3.0]`, `h0_swDur: [2.2, 2.6]`, `h0_trig: [9.0, 14.0]`.
  - `exposure`: 0.04 (trung bình 1 lần trong 25 chuyến đi phố cự ly 5 – 10 km).
  - *Căn cứ trong `hazard.json: why`:* Theo các nghiên cứu an toàn xe hai bánh của NHTSA (DOT HS 812 018) và NACTO, các cú đánh lái tránh chướng ngại vật của xe đạp diễn ra trong 2.2 – 2.6 s với gia tốc ngang đỉnh $\le 3.0\text{ m/s}^2$ (đảm bảo xe đạp không bị trượt ngã). Cự ly kích hoạt 9 – 14 m ở tốc độ 10 – 12 m/s là hoàn toàn tránh được nếu phanh khẩn cấp hết cỡ ngay khi thấy (khoảng phanh dừng cực đại mất ~9.5 m).
- **Kết quả kiểm thử từ `evidence/mine.json`:**
  - Tỉ lệ kịch bản khả thi vật lý: **`accepted = 99.5%`** (1 493 / 1 500 kịch bản được chấp nhận, chỉ 7 kịch bản bị loại do vượt giới hạn ma sát).
  - **Với policy gốc (baseline):**
    - Tổng va chạm: **755 vụ**
    - Tránh được bằng phanh hết cỡ ngay khi thấy (`avoidable`): **752 vụ**
    - Không thể tránh được (`unavoidable`): 3 vụ
    - Không quy được (`unattributed`): 0 vụ
    - Va chạm tốc độ thấp (`low_speed` $< 1\text{ m/s}$): 1 vụ
    - **$P(\text{lỗi planner} \mid \text{hazard}) = 5.04 \times 10^{-1}$** (Wilson 95%: $[4.78 \times 10^{-1},\; 5.29 \times 10^{-1}]$)
    - Tần suất mỗi chuyến: **$2.01 \times 10^{-2}$ vụ/chuyến** (chặn trên $2.12 \times 10^{-2}$)
  - **Với policy đã chọn ở bước E (`cutinLat: 2.6`):**
    - Tổng va chạm: **458 vụ**
    - Tránh được bằng phanh hết cỡ ngay khi thấy (`avoidable`): **455 vụ**
    - Không thể tránh được (`unavoidable`): 2 vụ
    - Không quy được (`unattributed`): 1 vụ
    - Va chạm tốc độ thấp (`low_speed` $< 1\text{ m/s}$): 11 vụ
    - **$P(\text{lỗi planner} \mid \text{hazard}) = 3.05 \times 10^{-1}$** (Wilson 95%: $[2.82 \times 10^{-1},\; 3.29 \times 10^{-1}]$)
    - Tần suất mỗi chuyến: **$1.22 \times 10^{-2}$ vụ/chuyến** (chặn trên $1.31 \times 10^{-2}$)
  - *Nhận xét:* Bản sửa ở bước E (`cutinLat = 2.6`) giúp mở rộng hành lang quan sát và giảm được ~40% số vụ va chạm avoidable (từ 752 xuống 455 vụ), nhưng **vẫn thất bại ở 455 kịch bản** vì bản chất IDM vẫn phanh quá êm ($b = 2.0\text{ m/s}^2$).
  - *Tần suất so với lách xe buýt:* Tần suất tai nạn do rắc rối này ở policy gốc là $2.01 \times 10^{-2}$ vụ/chuyến, **cao gấp 50 lần** so với tần suất tai nạn lách xe buýt trong điều kiện tự nhiên ($4.4 \times 10^{-4}$ vụ/chuyến).
- **Hình ảnh minh chứng:**
  ![Va chạm xe đạp lạng lấn làn trong xưởng kịch bản](evidence/shots/hazard_mine.png)
- **Bạn tự lái thì sao:** Khi người tự lái xe trong chương 10, người lái quan sát thấy góc đánh lái và hướng di chuyển của xe đạp từ sớm sẽ lập tức chủ động phanh dứt khoát hơn (gia tốc phanh $-4$ đến $-6\text{ m/s}^2$) hoặc đánh lái nhẹ né sang trái nếu làn bên cạnh thoáng, hoàn toàn tránh được va chạm. Planner máy móc chỉ bám theo vạch làn và phanh êm theo IDM nên bị đâm vào xe đạp.
- **Vì sao "tránh được" là chuẩn lạc quan:**
  Chuẩn `avoidable` dựa trên bộ oracle phanh hết cỡ ở giới hạn ma sát lốp ($\mu g \approx 7.85\text{ m/s}^2$) ngay từ frame đầu tiên cảm biến nhìn thấy rắc rối. Đây là một chuẩn lạc quan vì:
  1. Nó giả định hệ thống phanh phản ứng tức thời không trễ cảm biến, không trễ cơ cấu chấp hành thủy lực và phanh kịch lốp không cần quan tâm đến độ giật gây khó chịu/chấn thương cho hành khách (đếm nhiều hơn số va chạm mà một planner thực tế có thể tránh an toàn).
  2. Mặt khác, oracle chỉ thử phanh thẳng đứng yên mà không thử đánh lái né tránh (nếu cho phép đánh lái vòng qua thì số kịch bản cứu được trên thực tế còn cao hơn nữa).
- **Cách sửa:**
  - *Bằng núm chỉnh:* Có thể tăng `b` (gia tốc phanh IDM) lên 3.5 m/s² và tăng `cutinLat` lên 3.2 m để phanh gấp hơn, tuy nhiên điều này sẽ làm xe chạy rất giật cục và gây khó chịu trong giao thông bình thường.
  - *Logic mới cần thêm cho planner:* Cần bổ sung một **module phanh khẩn cấp độc lập (AEB - Autonomous Emergency Braking)**: khi phát hiện bất kỳ vật thể nào có thành phần vận tốc ngang (lateral velocity) cắt vào làn xe phía trước với thời gian Time-to-Collision (TTC) dưới 1.5 giây, hệ thống tự động ghi đè lên bộ điều khiển IDM và kích hoạt lực phanh khẩn cấp tối đa $\mu g$ để dừng xe ngay lập tức.
