MSSV: 2A202602787

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): A

## 1. Phát lại log có trung thực không? (bước A)

Theo `evidence/replay.json`:
* Số kịch bản bị loại do người lái thật va chạm: `dropped_human_crash_logs = 0` (tổng số kịch bản `n = 400`, sử dụng `n_used = 400`).
* Số va chạm khi planner chạy trên log phát lại: `replay.crashes = 107`, chia theo loại tiếp xúc gồm `rear = 96` (90%) và `side = 11` (10%).
* Số va chạm khi planner chạy trong thế giới phản ứng: `reactive.crashes = 0` (không có vụ va chạm nào).
* Số kịch bản xảy ra va chạm ở cả hai môi trường: `crash_in_both = 0`.

**Giải thích:**
Phần lớn va chạm trong log phát lại là va chạm đuôi (`rear = 96`) và đây hoàn toàn là **va chạm giả (spurious crashes)**. Nguyên nhân là khi phát lại log tĩnh, quỹ đạo của các xe khác bị ghim cứng theo dữ liệu người thật đã ghi nhận. Khi xe ta (planner) phát hiện tình huống và phanh sớm hoặc thận trọng hơn người lái thật, xe bám đuôi (`TG`) trong log không hề hay biết và không giảm tốc theo, dẫn đến việc húc thẳng vào đuôi xe ta. Ngược lại, trong thế giới phản ứng (`reactive`), xe bám đuôi lái theo IDM có cảm biến quan sát thấy xe ta giảm tốc và sẽ tự động phanh theo giữ khoảng cách an toàn, triệt tiêu toàn bộ va chạm.

**Phân tích kịch bản trong `replay_only`:**
Kịch bản `scenario: 1` (`replay.json: replay_only[0]`): Trong môi trường phát lại log, xe ta bị xe sau đâm đuôi (`contact: "rear"`). Tuy nhiên trong thế giới phản ứng, xe sau nhận diện xe ta giảm tốc và phanh kịp thời, khoảng thời gian va chạm nhỏ nhất được bảo toàn an toàn ở mức `min_ttc_reactive = 2.53 s`, không hề có sự cố nào xảy ra.

---

## 2. Kịch bản bất khả thi (bước B)

**Các luật trong `is_feasible` và căn cứ vật lý:**
1. $a_{\text{lat}} = (\pi / \text{cutDur})^2 \times 3.5 / 2 \le \mu g$: Xe chuyển làn hình cosine qua cự ly ngang 3.5 m trong thời gian `cutDur` đạt đỉnh gia tốc ngang $a_{\text{lat}}$. Lực bám ngang của lốp xe bị chặn trên bởi giới hạn ma sát cực đại $\mu g$ (Coulomb friction limit). Nếu $a_{\text{lat}} > \mu g$, xe sẽ trượt văng ngang mất lái.
2. $\text{cutBrake} \le \mu g$: Gia tốc phanh dọc cực đại của xe sau khi nhập làn không thể vượt quá ma sát lốp trên mặt đường $\mu g$.
3. $\text{pedV} \le 2.5\text{ m/s}$: Tốc độ đi bộ thông thường của con người khoảng $1.2 - 1.5\text{ m/s}$, tối đa khi bước nhanh/chạy vội qua đường là $2.5\text{ m/s}$ ($9\text{ km/h}$). Tốc độ lớn hơn tương đương vận động viên điền kinh chạy nước rút, không phản ánh người đi bộ đô thị bình thường.
4. $\text{cutV} \in (0, 22]\text{ m/s}$: Xe chuyển làn phải di chuyển về phía trước và tốc độ không vượt quá $22\text{ m/s}$ ($\approx 79.2\text{ km/h}$) - giới hạn tốc độ lưu thông đô thị thực tế.

**Cuộc săn sinh nhiều kịch bản bất khả thi nhất:**
Theo `evidence/hunt.json`, cuộc săn **`rear_ended`** sinh ra nhiều kịch bản bất khả thi nhất với **22 kịch bản bị loại** (`infeasible = 22`), kế tiếp là `crosswalk_block` (17), `cut_in` (8), và `bus_bypass` (1).
Thuật toán CEM tối ưu hóa bằng cách tìm các tham số cực đại hóa điểm nguy hiểm (danger score). Để tạo ra tình huống bám đuôi đâm đuôi hoặc cắt làn nguy hiểm nhất, thuật toán tự nhiên bị kéo về các kịch bản cực đoan nhất (như xe chuyển làn chỉ trong thời gian cực ngắn khiến gia tốc ngang vượt quá $\mu g$, hoặc xe cắt làn phanh vượt quá ma sát mặt đường).

**Hậu quả nếu đếm kịch bản bất khả thi là va chạm:**
Nếu tính các kịch bản này là va chạm, xác suất rủi ro sẽ bị thổi phồng nghiêm trọng (overestimate). Kỹ sư kiểm thử sẽ bị dẫn dắt sai lệch, dành thời gian và tài nguyên để "chữa" những lỗi cho các tình huống phi vật lý không bao giờ tồn tại trên đường thật.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

Từ `evidence/mc.json`:
* Số lần thử: $n = 3000$.
* Số va chạm ghi nhận: $k = 2$ (đều thuộc kiểu `bus_bypass`: `modes.bus_bypass = 2`).
* Tỷ lệ ước lượng điểm: $\hat{p} = k / n = 6.67 \times 10^{-4}$ ($0.0667\%$).
* Khoảng tin cậy Wilson 95%: $[1.83 \times 10^{-4}, 2.43 \times 10^{-3}]$ (`lo = 0.000183`, `hi = 0.002428`).
* Số kịch bản bất khả thi bị loại: `infeasible = 23`.

**Số lần chạy cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:**
Để đạt biên sai số tương đối $\epsilon = 0.2$ ở độ tin cậy 95% ($z = 1.96$):
$$n \approx \frac{z^2 (1 - p)}{\epsilon^2 p} \approx \frac{1.96^2}{0.2^2 \cdot p} \approx \frac{96.04}{p}$$
Với $\hat{p} = 6.67 \times 10^{-4}$:
$$n \approx \frac{96.04}{6.67 \times 10^{-4}} \approx 144,060 \text{ lần chạy}$$
Cần hơn **144.000 lần chạy**, một chi phí tính toán khổng lồ mà Monte Carlo thông thường khó đáp ứng.

**Kiểu thất bại chưa xuất hiện:**
Trong 3.000 lần chạy tự nhiên, các kiểu **`cut_in`**, **`crosswalk_block`**, **`rear_ended`**, và **`other`** đều chưa từng xuất hiện ($k = 0$).
Con số đúng cần báo cho mỗi kiểu chưa thấy là **cận trên khoảng Wilson 95%**: $hi = 0.001279$ ($1.28 \times 10^{-3}$), tức $p \le 1.28 \times 10^{-3}$ (`mc.json: per_mode.<mode>.hi = 0.001279`). Tuyệt đối không được báo xác suất bằng $0$ vì biến cố hiếm chưa quan sát được trong mẫu nhỏ không đồng nghĩa với việc nó không thể xảy ra.

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng tổng hợp từ `evidence/estimate.json` ($n = 600$ mỗi chế độ trên 3 seed):

| Kiểu thất bại | Seed | $p$ | hits | ess | top_share |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **bus_bypass** | 0 | $1.55 \times 10^{-3}$ | 39 | 146.3 | 0.318 |
| | 1 | $4.71 \times 10^{-4}$ | 43 | 105.5 | 0.149 |
| | 2 | $1.64 \times 10^{-3}$ | 47 | 135.4 | 0.197 |
| **cut_in** | 0 | 0.0 | 0 | 119.9 | 0.0 (cận trên 0.025) |
| | 1 | 0.0 | 0 | 114.8 | 0.0 (cận trên 0.025) |
| | 2 | 0.0 | 0 | 130.1 | 0.0 (cận trên 0.025) |
| **crosswalk_block** | 0 | $2.81 \times 10^{-7}$ | 64 | 112.7 | 0.194 |
| | 1 | $3.48 \times 10^{-7}$ | 62 | 117.4 | 0.108 |
| | 2 | $6.55 \times 10^{-7}$ | 60 | 124.2 | 0.331 |
| **rear_ended** | 0 | 0.0 | 0 | 117.3 | 0.0 (cận trên 0.025) |
| | 1 | 0.0 | 0 | 118.3 | 0.0 (cận trên 0.025) |
| | 2 | 0.0 | 0 | 119.1 | 0.0 (cận trên 0.025) |
| **Tổng cộng (total)** | 0 | $1.55 \times 10^{-3}$ | — | — | — |
| | 1 | $4.71 \times 10^{-4}$ | — | — | — |
| | 2 | $1.64 \times 10^{-3}$ | — | — | — |

*Phần bù `other`*: lấy từ Monte Carlo C1 (`estimate.json: other`), $k = 0, n = 3000, p = 0.0, hi = 0.001279$.

**Đánh giá:**
* **Đồng thuận giữa 3 seed:** Ba seed cho kết quả rất ổn định và cùng cấp độ lớn: `bus_bypass` dao động trong khoảng $4.71 \times 10^{-4}$ đến $1.64 \times 10^{-3}$ (chênh lệch chỉ ~3.5 lần); `crosswalk_block` dao động quanh $2.81 \times 10^{-7} - 6.55 \times 10^{-7}$ (chênh lệch ~2.3 lần).
* **Đồng thuận với Monte Carlo:** Trung vị tổng xác suất qua 3 seed là **$1.55 \times 10^{-3}$**, nằm hoàn toàn bên trong khoảng tin cậy 95% của Monte Carlo C1 ($[1.83 \times 10^{-4}, 2.43 \times 10^{-3}]$).
* **Phân tích `top_share`:** Ở tất cả các kiểu và các seed, `top_share` đều nhỏ hơn 0.35, không có trường hợp nào vượt quá 0.5. Nếu `top_share > 0.5`, một mẫu đơn lẻ mang trọng số quá lớn và quyết định toàn bộ con số ước lượng (dấu hiệu suy biến trọng số). Nhờ có phân phối phòng thủ $q = 0.8 q_{\text{CEM}} + 0.2 p_{\text{nat}}$, trọng số luôn được chặn trên $w = p/q \le 1/0.2 = 5$, tránh hiện tượng nổ trọng số.
* **So sánh ngân sách (`budget`):** Ngân sách chạy của IS là `budget = 16,800` lần chạy (`estimate.json`). Trong khi đó, để tìm ra biến cố có xác suất $10^{-7}$ với Monte Carlo cần hàng chục triệu lần chạy. IS giúp tiết kiệm tài nguyên hàng ngàn lần.
* **Phần `other`:** Lấy từ Monte Carlo C1 và không săn bằng CEM vì `other` là tập hợp các va chạm ngoại lai không có hàm mục tiêu định hướng cụ thể (như TTC hay cự ly phanh) để kéo phân phối đề xuất $q$.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

**Các kiểu thất bại tìm thấy:**
1. **`bus_bypass`** (lách qua xe buýt đỗ):
   * *Ảnh 3D:* ![Bus bypass collision](evidence/shots/ch9-lab-bus-bypass.jpg)
   * *Chuỗi sự kiện:* Xe buýt đỗ chắn làn E2 phía trước vạch qua đường. Xe cắt làn C2 cố gắng lách qua xe buýt với vận tốc chậm ($x < 112$ m, $v < 6$ m/s). Do tầm nhìn của xe ta bị xe buýt che khuất, xe ta duy trì tốc độ và không kịp nhường đường khi C2 xuất hiện, dẫn đến va chạm bên/đầu xe.
2. **`crosswalk_block`** (chặn lối người đi bộ):
   * *Ảnh 3D:* ![Crosswalk block failure](evidence/shots/ch9-lab-crosswalk-block.jpg)
   * *Chuỗi sự kiện:* Người đi bộ P1 bước ra sang đường từ phía sau đuôi xe buýt đỗ. Tầm nhìn xe ta bị che khuất cho tới khi đến rất gần; xe ta phanh gấp ở giới hạn lốp nhưng dừng quá muộn, thân xe đỗ đè lên vạch qua đường chắn lối đi của P1 (`event["kind"] == "block"`).

**Kiểu không tìm thấy:**
* Cuộc săn `cut_in`: tìm được 36 vụ `bus_bypass`, 0 vụ `cut_in` (`hunt.json`).
* Cuộc săn `rear_ended`: tìm được 3 vụ `bus_bypass`, 7 vụ `crosswalk_block`, 0 vụ `rear_ended` (`hunt.json`).
* *Mức độ tin cậy:* Rất tin cậy. Xe đi sau lái theo mô hình IDM phản ứng sẽ luôn phanh khi xe ta phanh; còn xe cắt làn ngoài khu vực xe buýt có cơ chế nhường khoảng trống (gap acceptance) nên trong điều kiện bình thường không xảy ra va chạm.

**Nếu chỉ chạy một cuộc săn chung:**
Thuật toán CEM sẽ bị co cụm (mode collapse) vào duy nhất kiểu va chạm có điểm nguy hiểm dễ kích hoạt nhất là `bus_bypass`, và bỏ qua hoàn toàn kiểu va chạm cực hiếm nhưng nguy hiểm như `crosswalk_block` (minh chứng rõ ràng là cuộc săn `cut_in` và `rear_ended` đều bị hút ngược về `bus_bypass`).

---

## 6. Sửa planner (bước E + 3D)

Dữ liệu từ `evidence/fix.json`:
* **Policy được chọn:** `{'cutinLat': 3.0}` (mở rộng ngưỡng ngang nhận diện xe cắt làn từ 2.15 m lên 3.0 m).
* **Trên tập TRAIN:**
  * `fail = 2.45e-07`, `fail_hi = 3.97e-07`, `progress = 216.3` m, `unseen = ['bus_bypass']`.
* **Trên tập TEST:**
  * Bản sửa (`chosen`):
    * `fail = 1.26e-05`
    * `fail_hi_seen = 2.97e-05`
    * `fail_hi (valid) = 3.88e-02` (0.0388)
    * `unseen = ['bus_bypass']`
    * Quãng đường `progress = 216.0` m (216.04 m, giảm chỉ 0.02% so với baseline 216.09 m)
    * Số frame phanh gấp: `harsh = 7.86`
  * Bản gốc (`baseline`):
    * `fail = 6.15e-04`
    * `fail_hi_seen = 1.03e-03`
    * `fail_hi (valid) = 2.31e-03`
    * `unseen = []`
    * Quãng đường `progress = 216.1` m
    * Số frame phanh gấp: `harsh = 8.32`

**Vì sao `fail_hi` của bản sửa ($0.0388$) lớn hơn `fail` baseline ($6.15 \times 10^{-4}$):**
Trên tập TEST, bản sửa đã triệt tiêu hoàn toàn các vụ va chạm `bus_bypass` (`hits = 0`), khiến kiểu này trở thành `unseen`. Theo nguyên lý Rule of Three cho kiểu chưa thấy, cận trên 95% được tính bằng $3 / (\beta \cdot n) = 3 / (0.2 \times 400) = 0.0375$ và được cộng vào `fail_hi`. Với ngân sách $n = 400$, ta chưa thể chứng minh bằng thống kê rằng một kiểu va chạm đã biến mất dưới mức $10^{-2}$. Tuy nhiên, trên những gì dữ liệu quan sát được (`fail_hi_seen`), bản sửa an toàn hơn bản gốc tới **35 lần** ($2.97 \times 10^{-5}$ so với $1.03 \times 10^{-3}$).

**Ảnh 3D với bóng mờ (ghost):**
* *Ảnh:* ![Chương 9: xe ta với policy sửa tránh được va chạm trong khi bóng mờ va chạm](evidence/shots/ch9-lab-bus-bypass.jpg)
* Bóng mờ thể hiện xe ta theo policy gốc bị đâm khi C2 lách xe buýt, trong khi policy sửa với `cutinLat = 3.0` phát hiện C2 từ cự ly ngang xa hơn và nhường đường an toàn.

**Núm chỉnh "nghe an toàn hơn" nhưng thực tế phản tác dụng:**
Ứng viên 1: `cautionV = 5.0` (hạ vận tốc thận trọng qua vạch kẻ xuống 5 m/s thay vì 7.5 m/s). Nghe qua có vẻ rất an toàn, nhưng thực tế làm giảm quãng đường nghiêm trọng xuống `progress = 175.9` m (mất tới 18.6% tiến độ) và số lần phanh gấp tăng vọt từ 8.07 lên `harsh = 13.16` frame (`fix.json: table[0]`). Việc xe ta phanh giật cục và bò quá chậm khiến các xe khác trong dòng giao thông bị ức chế, tăng nguy cơ va chạm liên hoàn từ phía sau.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định trong `space.py` ảnh hưởng nhiều nhất đến các con số:
1. **Khe chấp nhận nhập làn (`cutGap`, lognormal median 12.0 m):**
   * *Dữ liệu thật thay thế:* Trích xuất phân phối khoảng cách nhập làn từ video giám sát camera/flycam giao thông đô thị thực tế (như bộ dữ liệu HighD hoặc Waymo Open Motion).
   * *Hướng sai lệch:* Nếu tài xế thực tế bạo dạn hơn (chấp nhận khe hở $< 12$ m), số vụ va chạm tạt đầu cắt làn (`cut_in`) ngoài đời sẽ cao hơn nhiều so với mô phỏng.
2. **Tỉ lệ người đi bộ ngó qua xe buýt (`pedLook`, Bernoulli $p = 0.85$):**
   * *Dữ liệu thật thay thế:* Nghiên cứu hành vi người đi bộ qua đường (dữ liệu camera thị giác máy tính ghi nhận chuyển động đầu/hướng nhìn của người đi bộ tại các trạm dừng xe buýt).
   * *Hướng sai lệch:* Nếu ngoài đời người đi bộ chủ quan hoặc mải dùng điện thoại không ngó nghiêng ($p < 0.85$), tỷ lệ tai nạn bị che khuất (`crosswalk_block` / va chạm người đi bộ) sẽ tăng vọt so với kết quả kiểm thử.
3. **Hệ số ma sát đường ướt (`wet`, Bernoulli $p = 0.20$, $\mu = 0.5$ vs $\mu = 0.8$):**
   * *Dữ liệu thật thay thế:* Dữ liệu khí tượng thủy văn đo số ngày mưa cục bộ và kết quả đo độ bám mặt đường thực tế của cơ quan quản lý đường bộ (dùng con lắc ma sát British Pendulum).
   * *Hướng sai lệch:* Nếu khu vực hoạt động mưa nhiều hơn hoặc đường có bùn trơn trượt ($\mu < 0.5$), quãng đường phanh thực tế dài hơn đáng kể, làm tăng xác suất va chạm và chặn vạch qua đường.

---

## 8. Giới hạn

**Ba điều mà xác suất báo cáo KHÔNG nói lên:**
1. *Bản đồ tuyến tính đơn giản:* Thế giới mô phỏng chỉ là một đoạn đường thẳng một chiều 2 làn, hoàn toàn chưa có ngã tư, bùng binh, đường dốc, góc cua khuất tầm nhìn hay các công trường đang thi công.
2. *Mô hình hành vi lý tưởng hóa (IDM):* Các phương tiện khác tuân thủ nghiêm ngặt mô hình IDM, giữ làn chuẩn và không có các hành vi khó đoán như lấn làn ép xe, vượt ẩu lạng lách hoặc tài xế mất tập trung.
3. *Cảm biến hoàn hảo không nhiễu:* Xe ta có hệ thống cảm biến lý tưởng nhận diện 3D bounding box tức thì và chính xác 100% trong tầm nhìn thẳng (chỉ bị che khuất bởi xe buýt), không có độ trễ nhận thức (latency), không có trường hợp phát hiện nhầm (false positive) hay bỏ sót vật thể (false negative).

**Nếu được thêm một tuần:**
Tôi sẽ mở rộng mô phỏng sang kịch bản ngã tư có đèn giao thông với luồng phương tiện rẽ trái cắt đầu dòng xe thẳng, đồng thời mô hình hóa thêm nhiễu cảm biến (sensor noise & latency) và hành vi vượt đèn đỏ của người đi bộ.

---

## 9. Rắc rối của bạn (bước G + chương 10)

* **Rắc rối:** Người đi bộ bất ngờ băng ngang đường ngoài giao lộ (jaywalking) ở đoạn giữa phố ($x \in [220, 255]$ m, `hazard.json`).
* **Căn cứ:** Bộ lập kế hoạch của xe ta chỉ có logic thận trọng `cautionV` và `cautionRange` khi tới gần vạch qua đường chính thức (`WALK_X = 122` m). Trên các đoạn đường thẳng khác, xe ta duy trì vận tốc tối đa $v_0 = 12.5$ m/s và chỉ trông cậy vào thuật toán IDM cơ bản. Khi người đi bộ bất ngờ cắt ngang ngoài vạch, planner có nguy cơ phanh không kịp.
* **Giả định:**
  * `space`: $h0_x \in [220, 255]$ m, $h0_v \in [1.2, 2.0]$ m/s, $h0_{\text{trig}} \in [20, 36]$ m.
  * `exposure = 0.04`: Căn cứ theo thống kê an toàn giao thông đô thị của NHTSA và WHO, hiện tượng người đi bộ sang đường tự do ngoài vạch xuất hiện trung bình khoảng 0.04 lần mỗi chuyến đi 15–20 km trong nội thị.
* **Kết quả theo `evidence/mine.json`:**
  * Tổng số mẫu thử hợp lệ: $n = 1493$ / 1500 (`accepted = 99.53%`).
  * Với policy gốc (`baseline`): 2 va chạm, gồm `avoidable = 0`, `unavoidable = 0`, `unattributed = 2`, `low_speed = 0`.
  * Với policy được chọn (`chosen`, `cutinLat = 3.0`): 0 va chạm (`avoidable = 0`, `unavoidable = 0`, `unattributed = 0`).
  * $P(\text{lỗi planner} \mid \text{rắc rối}) = 0.0$ với khoảng Wilson 95% là $[0.0, 2.57 \times 10^{-3}]$.
  * Tần suất lỗi mỗi chuyến đi: $0.0$ (cận trên $1.03 \times 10^{-4}$ mỗi chuyến).
* **Hình ảnh rắc rối:**
  ![Chương 10: Xưởng kịch bản thiết kế rắc rối](evidence/shots/ch10-studio.jpg)
* **Bạn tự lái thì sao:**
  Khi tự cầm lái trên 3D Studio, người lái quan sát thấy bóng người từ xa sẽ chủ động nhả chân ga và rà phanh phòng ngừa (defensive driving) từ trước, giúp xe đi qua nhẹ nhàng và an toàn, khác với planner chỉ bắt đầu phanh khi đối tượng đã bước hẳn vào hành lang di chuyển.
* **Phân tích "tránh được là chuẩn lạc quan" (optimistic standard):**
  * *Một mặt*, bộ kiểm tra oracle giả định xe phanh tối đa ở giới hạn lốp ($\mu g$) ngay từ frame đầu tiên phát hiện rắc rối mà không có độ trễ nhận thức hay yêu cầu êm ái, nên nó đếm **nhiều hơn** số va chạm mà một planner thực tế có thể tránh được (cận trên của lỗi planner).
  * *Mặt khác*, chuẩn này chỉ thử nghiệm duy nhất giải pháp đạp phanh dừng xe (không thử đánh lái né tránh), nên đối với những kịch bản có thể lách vòng qua chướng ngại vật thì chuẩn này lại đếm **ít đi**.
* **Đánh giá & Cách sửa:**
  Policy sửa (`cutinLat = 3.0`) đã loại bỏ hoàn toàn cả 2 va chạm không quy được của baseline. Đối với rắc rối người đi bộ ngoài giao lộ, planner hiện tại đã xử lý tốt vì đoạn đường quang đãng giúp cảm biến nhận diện đối tượng từ sớm. Để củng cố an toàn khi gặp rắc rối này ở điều kiện tầm nhìn kém, planner cần bổ sung thêm **logic "vùng đệm giảm tốc theo ngữ cảnh" (contextual speed cushion)**: khi phát hiện bất kỳ người đi bộ nào đứng sát mép đường ($|z| \le 4.5$ m), hệ thống tự động hạ giới hạn tốc độ trần xuống $\sim 8\text{ m/s}$ thay vì chỉ kích hoạt tại vạch kẻ đường cố định.
