# Hướng dẫn sử dụng Kibana để phân tích logs

## 1. Truy cập Kibana

- Mở trình duyệt và truy cập: `http://<IP_vps0>:5601`
- Lần đầu truy cập, Kibana sẽ yêu cầu tạo index pattern. Nếu không, có thể vào menu **Stack Management** > **Index Patterns** để tạo.

## 2. Tạo Index Pattern

1. Vào **Stack Management** > **Index Patterns**
2. Chọn **Create index pattern**
3. Nhập pattern: `logs-*` để bắt tất cả các index của logs
4. Chọn **Next step**
5. Chọn trường thời gian: `@timestamp`
6. Chọn **Create index pattern**

## 3. Khám phá dữ liệu với Discover

### 3.1. Tìm kiếm cơ bản

1. Vào tab **Discover** từ menu chính
2. Chọn index pattern `logs-*` nếu chưa được chọn
3. Sử dụng thanh tìm kiếm phía trên để tìm kiếm logs:
   - Tìm logs theo service: `service_name: "rng"`
   - Tìm logs có lỗi: `log_level: "ERROR"` hoặc `tags: "error_log"`
   - Tìm logs có chứa từ khóa: `message: exception` hoặc `message: timeout`

### 3.2. Sử dụng bộ lọc (Filters)

1. Từ giao diện Discover, chọn các giá trị từ các trường để thêm filter:
   - Click vào biểu tượng "+" bên cạnh giá trị trong danh sách kết quả
   - Hoặc click vào giá trị trong panel Available Fields bên trái
2. Kết hợp nhiều filter để tìm kiếm chi tiết, ví dụ:
   - Service name: `rng`
   - Log level: `ERROR`
   - Tags: `important`

### 3.3. Sử dụng Kibana Query Language (KQL)

```
service_name: "rng" and (log_level: "ERROR" or message: *exception*)
```

### 3.4. Điều chỉnh khoảng thời gian

- Sử dụng bộ chọn thời gian ở góc trên bên phải để xem logs trong khoảng thời gian cụ thể
- Có thể chọn các preset như "Last 15 minutes", "Today", hoặc tùy chỉnh khoảng thời gian

## 4. Xây dựng Dashboard

### 4.1. Tạo Visualization

1. Từ menu chính, chọn **Dashboard** > **Create new dashboard**
2. Chọn **Create new visualization**
3. Chọn loại biểu đồ phù hợp:
   - Line chart: Hiển thị số lượng logs theo thời gian
   - Pie chart: Hiển thị tỷ lệ logs theo service hoặc log level
   - Data table: Hiển thị danh sách logs chi tiết
   - Metric: Hiển thị số lượng logs quan trọng

### 4.2. Tạo Line Chart theo thời gian

1. Chọn **Line**
2. Thiết lập:
   - Y-axis (Metrics): Count
   - X-axis (Buckets): Date Histogram với trường @timestamp
   - Split Series (nếu muốn): Terms với trường service_name hoặc log_level
3. Chọn **Update** để xem kết quả và điều chỉnh
4. Chọn **Save and return** khi hoàn tất

### 4.3. Tạo Pie Chart phân bố logs theo service

1. Chọn **Pie**
2. Thiết lập:
   - Metrics: Count
   - Buckets: Split slices với Aggregation: Terms, Field: service_name
3. Chọn **Update** để xem kết quả và điều chỉnh
4. Chọn **Save and return** khi hoàn tất

### 4.4. Tạo Data Table hiển thị logs lỗi

1. Chọn **Data Table**
2. Thiết lập:
   - Metrics: Count
   - Buckets: Split rows với Aggregation: Terms, Field: service_name
   - Buckets: Split rows với Aggregation: Terms, Field: log_message
3. Thêm Filter: `log_level: "ERROR"` hoặc `tags: "error_log"`
4. Chọn **Update** và **Save and return**

### 4.5. Hoàn thiện Dashboard

1. Sắp xếp các visualization bằng cách kéo thả
2. Điều chỉnh kích thước bằng cách kéo góc các visualization
3. Chọn **Save** ở góc trên bên phải để lưu dashboard
4. Đặt tên có ý nghĩa, ví dụ: "Service Logs Overview"

## 5. Phân tích nâng cao

### 5.1. Lens - Công cụ tạo biểu đồ đơn giản

1. Từ Dashboard, chọn **Create new** > **Lens**
2. Kéo thả trường dữ liệu từ panel bên trái vào khu vực thiết kế
3. Lens sẽ tự động đề xuất các biểu đồ phù hợp
4. Điều chỉnh các thiết lập và chọn **Save and return**

### 5.2. Canvas - Tạo báo cáo trực quan

Canvas cho phép tạo báo cáo tùy chỉnh cao với các yếu tố thiết kế và tương tác:

1. Từ menu chính, chọn **Canvas**
2. Chọn **Create new canvas**
3. Thêm các element như text, image, chart
4. Liên kết dữ liệu từ Elasticsearch với các element

### 5.3. Machine Learning

Sử dụng tính năng Machine Learning để phát hiện bất thường trong logs:

1. Từ menu chính, chọn **Machine Learning**
2. Chọn **Anomaly Detection** > **Create job**
3. Chọn index pattern `logs-*`
4. Thiết lập các tham số phát hiện bất thường

## 6. Tìm kiếm logs với các trường đặc biệt

### 6.1. Tìm kiếm logs theo timestamp

```
@timestamp >= "2023-06-01T00:00:00" and @timestamp <= "2023-06-30T23:59:59"
```

### 6.2. Tìm kiếm logs theo service và log level

```
service_name: "worker" and log_level: "ERROR"
```

### 6.3. Tìm kiếm logs có chứa chuỗi cụ thể

```
message: *connection timeout* or message: *exception*
```

### 6.4. Tìm kiếm logs có tag cụ thể

```
tags: "important" and tags: "error_log"
```

## 7. Lưu và chia sẻ kết quả tìm kiếm

1. Từ giao diện Discover, sau khi đã thiết lập các filter và khoảng thời gian phù hợp
2. Chọn **Save** ở góc trên bên phải
3. Đặt tên có ý nghĩa và mô tả (tùy chọn)
4. Để chia sẻ, chọn **Share** > **Permalinks** hoặc **Share** > **Embed code**

## 8. Thiết lập cảnh báo

1. Từ menu chính, chọn **Stack Management** > **Rules and Connectors**
2. Chọn **Create rule**
3. Chọn loại rule, ví dụ: **Threshold**
4. Thiết lập điều kiện, ví dụ: Số lượng logs ERROR vượt ngưỡng
5. Thiết lập hành động khi điều kiện được đáp ứng (gửi email, webhook,...)

## 9. Mẹo sử dụng Kibana hiệu quả

1. **Sử dụng Dashboard Templates**: Tạo các dashboard mẫu cho từng loại phân tích
2. **Tùy chỉnh thời gian làm mới**: Điều chỉnh tần suất làm mới dashboard
3. **Sử dụng filter toàn cục**: Áp dụng filter cho toàn bộ Kibana
4. **Lưu và chia sẻ URL**: Lưu URL với các filter và thời gian để dễ dàng chia sẻ
5. **Sử dụng chế độ toàn màn hình**: Cho phép xem dashboard không bị phân tâm
6. **Xuất dữ liệu**: Xuất kết quả tìm kiếm dưới dạng CSV hoặc PDF 