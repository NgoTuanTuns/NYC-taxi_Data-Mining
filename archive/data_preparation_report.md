# Báo cáo Data Preparation — NYC Yellow Taxi Trip Data

## 1. Mục tiêu

Giai đoạn Data Preparation biến đổi dữ liệu thô NYC Yellow Taxi Trip
Records (TLC) thành hai tập dữ liệu sẵn sàng cho Modeling: một tập phục vụ
bài toán Classification (DM1 — dự đoán `high_tip`) và một tập phục vụ bài
toán Clustering (DM2 — phân cụm mẫu hình di chuyển). Toàn bộ quyết định xử
lý đều dựa trên các phát hiện đã kiểm chứng ở giai đoạn Data Understanding,
không áp dụng quy tắc mặc định chung chung.

**Dữ liệu đầu vào:** 4,090,836 dòng, 20 cột (tháng 05/2026).

## 2. Quy trình tổng quan

Theo CRISP-DM, Data Preparation được chia thành 5 quá trình con:

| # | Quá trình | Nội dung |
|---|---|---|
| 1 | Select Data | Chọn các trường liên quan tới DM1/DM2 |
| 2 | Clean Data | Xử lý null, loại trùng lặp, lọc outlier (đơn biến + đa biến) |
| 3 | Construct Data | Xây dựng đặc trưng mới (thời gian, tip) |
| 4 | Integrate Data | Ghép tên khu vực (Borough/Zone) |
| 5 | Format Data | Chọn cột cuối cùng, xuất file cho từng bài toán |

## 3. Chi tiết thực hiện

### 3.1. Select Data

Giữ lại 14/20 trường có liên quan trực tiếp tới ít nhất một trong hai bài
toán mining, loại bỏ các trường mang tính vận hành nội bộ không phục vụ
mục tiêu nào (`extra`, `mta_tax`, `tolls_amount`, `improvement_surcharge`,
`cbd_congestion_fee`, `total_amount`).

### 3.2. Clean Data

**a) Xử lý giá trị thiếu (Null Handling)**

Năm cột `passenger_count`, `RatecodeID`, `store_and_fwd_flag`,
`congestion_surcharge`, `Airport_fee` có tỷ lệ null giống hệt nhau
(23.35%), gợi ý null không ngẫu nhiên mà mang tính hệ thống. Kiểm định bằng
đối chiếu theo `VendorID` xác nhận giả thuyết:

| VendorID | Tỷ lệ null (5 cột trên) |
|---|---|
| 1 | 15.30% |
| 2 | 25.56% |
| 6 | 100.00% |
| 7 | 0.00% |

Kiểm định Chi-square cho $\chi^2 = 78{,}814.58$, $p\text{-value} \approx 0$,
bác bỏ giả thuyết null độc lập với VendorID. Đây là trường hợp **MNAR**
(Missing Not At Random) có chủ đích ở cấp hệ thống — VendorID 6 (Myle
Technologies) không gửi 5 trường này trong feed dữ liệu của mình.

**Chiến lược xử lý:**
- `congestion_surcharge`, `Airport_fee` → điền 0 (null ngụ ý không phát
  sinh phí)
- `passenger_count`, `store_and_fwd_flag` → điền mode (tính thực tế từ dữ
  liệu, không hardcode)
- `RatecodeID` → điền 99 (mã "Null/unknown" chính thức theo quy ước TLC)
- Một cột `data_quality_flag` được tạo **trước khi điền**, đánh dấu dòng
  nào từng thiếu dữ liệu gốc (`Not_recorded`, `Incomplete_vendor_feed`,
  `Partial_null`, `Complete`), giữ lại minh bạch cho các bước sau.

**b) Loại bản ghi trùng lặp**

Kiểm tra bằng `drop_duplicates()`. Kết quả thực tế: **0 dòng trùng lặp**
(0.0000%) trên tập dữ liệu tháng 05/2026 — bước này vẫn được giữ trong
pipeline như một biện pháp phòng ngừa chuẩn, dù không phát huy tác dụng ở
lần chạy này.

**c) Lọc giá trị bất thường (Outlier / Invalid)**

*Ngưỡng đơn biến* (theo hiểu biết nghiệp vụ, đã đối chiếu với IQR và kiểm
tra chi tiết ở Data Understanding):

| Trường | Ngưỡng | Số dòng vượt ngưỡng | Tỷ lệ |
|---|---|---|---|
| `fare_amount` | 0 < x < 500 | 74 | 0.0018% |
| `trip_distance` | 0 < x < 100 | 136 | 0.0033% |
| `passenger_count` | 1 ≤ x ≤ 6 | 4 | 0.0001% |

Ngưỡng IQR thuần thống kê ($Q_3 + 1.5 \times IQR$) cho ra 52 USD (fare) và
7.96 miles (distance) — thấp hơn đáng kể so với ngưỡng nghiệp vụ. Ngưỡng
IQR **không được áp dụng trực tiếp** vì phân phối fare/distance lệch phải
tự nhiên (tồn tại chuyến sân bay/đi xa hợp lệ). Kiểm tra chi tiết 74 dòng
`fare_amount > 500` phát hiện 13 giá trị lặp lại giống hệt nhau (VD:
5,525.99 USD xuất hiện 3 lần) và các chuyến `RatecodeID = 4`
(Nassau/Westchester, cách NYC thực tế ~20-40 miles) có `trip_distance` ghi
nhận tới 130-258 miles — xác nhận đây là lỗi ghi nhận, không phải chuyến đi
thật, củng cố ngưỡng nghiệp vụ đã chọn.

Cũng loại các chuyến có `trip_duration_min` < 1 hoặc > 180 phút, và
`VendorID` không nằm trong danh sách hợp lệ chính thức của TLC ({1, 2, 6,
7}) — không đánh đồng vendor nhỏ (VendorID 6) với dữ liệu lỗi.

*Ngưỡng đa biến (bivariate) — bổ sung dựa trên phát hiện ở Data
Understanding:*

$$
\text{speed\_mph} = \frac{\text{trip\_distance}}{\text{trip\_duration\_min}} \times 60
$$

1,222 chuyến (0.0311% tổng dữ liệu) có tốc độ suy ra vượt quá 65 mph — đây
là outlier "ẩn" lọt qua toàn bộ điều kiện lọc đơn biến ở trên, vì
`fare_amount`/`trip_distance` khi xét riêng lẻ vẫn nằm trong ngưỡng hợp lệ.
Điều kiện `speed_mph <= 65` được bổ sung vào bước Clean Data để loại nhóm
này.

### 3.3. Construct Data

Sáu đặc trưng mới được tạo:

| Đặc trưng | Mô tả | Phục vụ |
|---|---|---|
| `pickup_hour` | Giờ đón khách (0-23) | DM1, DM2 |
| `pickup_dayofweek` | Thứ trong tuần (0=Thứ 2 … 6=CN) | DM1, DM2 |
| `is_weekend` | Cờ cuối tuần | DM1, DM2 |
| `time_of_day` | Khung giờ (early_morning/midday/peak_hour/late_night) | DM1 |
| `tip_percentage` | `tip_amount / fare_amount` | Cơ sở tạo nhãn |
| `high_tip` | Nhãn nhị phân: 1 nếu `tip_percentage ≥ 0.15` | Nhãn DM1 |

### 3.4. Integrate Data

Ghép `pickup_borough`, `pickup_zone`, `dropoff_borough`, `dropoff_zone` từ
bảng tra cứu `taxi_zone_lookup.csv` theo `PULocationID`/`DOLocationID`,
chuyển mã định danh số thành thông tin địa lý có thể diễn giải.

### 3.5. Format Data

Hai bài toán mining có bản chất khác nhau (có giám sát vs không giám sát),
nên dùng hai tập đặc trưng riêng biệt, không gộp chung:

**DM1 (Classification)** — chỉ giữ giao dịch thanh toán thẻ
(`payment_type = 1`), vì thanh toán tiền mặt không được TLC ghi nhận tip
(`tip_amount` luôn bằng 0 một cách giả tạo), giữ lại sẽ gây lệch nhãn
nghiêm trọng (biến `payment_type` trở thành confounding variable lấn át
các đặc trưng nghiệp vụ thật).

Cột sử dụng: `trip_distance`, `trip_duration_min`, `passenger_count`,
`pickup_hour`, `pickup_dayofweek`, `is_weekend`, `time_of_day`,
`pickup_borough`, `dropoff_borough`, `fare_amount`, `high_tip` (nhãn).

**DM2 (Clustering)** — giữ toàn bộ giao dịch (không lọc `payment_type`),
loại bỏ các biến liên quan tới giá/tip (`fare_amount`, `high_tip`,
`passenger_count`) để tránh thuật toán K-Means (dựa trên khoảng cách
Euclid) vô tình phân cụm theo tiêu chí giá cả thay vì đúng mục tiêu mẫu
hình di chuyển.

Cột sử dụng: `trip_distance`, `trip_duration_min`, `pickup_hour`,
`pickup_dayofweek`, `is_weekend`, `pickup_borough`, `dropoff_borough`.

## 4. Kết quả đầu ra

| File | Nội dung | Dùng cho |
|---|---|---|
| `classification_data.parquet` | Chỉ giao dịch thẻ, có nhãn `high_tip` | DM1 |
| `clustering_data.parquet` | Toàn bộ giao dịch hợp lệ, không có nhãn | DM2 |

Train/Test Split (stratified theo `high_tip` cho DM1) được thực hiện ở đầu
notebook **Modeling**, đúng theo cấu trúc chuẩn CRISP-DM (task "Generate
Test Design" thuộc giai đoạn Modeling, không phải Data Preparation).

## 5. Giới hạn cần lưu ý

- Tập DM1 nhỏ hơn đáng kể so với DM2 do chỉ giữ giao dịch thẻ — kết luận từ
  mô hình DM1 chỉ đại diện cho nhóm khách thanh toán thẻ, không nên khái
  quát hoá cho toàn bộ hành khách taxi.
- Giá trị điền cho các cột null của VendorID 6 (null 100%) được lấy từ
  phân phối chung của VendorID 1 và 2 — một giả định cần nêu rõ vì có thể
  không phản ánh đúng đặc điểm riêng của vendor này.
- Ngưỡng lọc outlier (fare, distance, tốc độ) dựa trên hiểu biết nghiệp vụ
  kết hợp kiểm tra chi tiết ở cấp bản ghi, không phải công thức thống kê
  thuần túy — phù hợp với đặc thù dữ liệu taxi nhưng cần nêu rõ căn cứ khi
  trình bày để tránh bị hiểu là chọn ngưỡng cảm tính.
