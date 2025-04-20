# Báo Cáo Triển Khai ELK Stack trên Docker Swarm

## 1. Giới thiệu hệ thống

Báo cáo này trình bày việc triển khai hệ thống ELK Stack (Elasticsearch, Logstash, Kibana) trên nền tảng Docker Swarm với máy ảo đơn VirtualBox. Hệ thống được thiết kế để thu thập, xử lý và trực quan hóa logs từ 4 service: rng, hasher, worker và webui.

## 2. Mô hình triển khai

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌──────────────┐
│ Services    │      │ Filebeat    │      │ Logstash    │      │ Elasticsearch│
│ rng,hasher, ├─────►│ Thu thập    ├─────►│ Xử lý và    ├─────►│ Lưu trữ và   │
│ worker,webui│      │ logs        │      │ lọc logs    │      │ đánh index   │
└─────────────┘      └─────────────┘      └─────────────┘      └───────┬──────┘
                                                                       │
                                                                       ▼
                                                               ┌─────────────┐
                                                               │ Kibana      │
                                                               │ Trực quan   │
                                                               │ hóa dữ liệu │
                                                               └─────────────┘
```

## 3. Cấu hình và triển khai

### 3.1. Cài đặt Docker Swarm

Khởi tạo Swarm trên máy ảo:

```bash
docker swarm init --advertise-addr <IP_máy_ảo>
```

### 3.2. Triển khai ELK Stack

Tất cả các thành phần (Elasticsearch, Logstash, Kibana và Filebeat) được triển khai hoàn toàn qua Docker Swarm và được cấu hình để chạy trên cùng một node với yêu cầu tài nguyên tối thiểu phù hợp cho môi trường máy ảo.

```bash
# Chuyển đến thư mục chứa các file cấu hình
cd /path/to/elk-stack

# Triển khai stack
docker stack deploy -c docker-stack.yml elk_stack
```

#### 3.2.1. Về quyền root cho Filebeat

Filebeat cần chạy với quyền root để có thể truy cập logs từ các container khác. Đây là phần quan trọng trong cấu hình docker-stack.yml:

```yaml
filebeat:
  image: docker.elastic.co/beats/filebeat:8.17.4
  user: root  # Chỉ định container chạy với quyền root
  command: ["filebeat", "-e", "--strict.perms=false"]  # Sử dụng flag đúng cú pháp
  volumes:
    - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
    - /var/run/docker.sock:/var/run/docker.sock:ro
```

**Lưu ý quan trọng:**
1. Phải chỉ định `user: root` để Filebeat có quyền đọc logs
2. Sử dụng flag `--strict.perms=false` (với hai dấu gạch ngang) để tắt kiểm tra quyền nghiêm ngặt
3. Mount các volume cần thiết để truy cập logs và socket Docker
4. Không cần chạy script cài đặt Filebeat trên host, mọi thứ đã được container hóa

### 3.3. Kiểm tra trạng thái các service

```bash
# Xem tất cả các service trong stack
docker service ls

# Xem chi tiết service cụ thể
docker service ps elk_stack_elasticsearch
docker service ps elk_stack_logstash
docker service ps elk_stack_kibana
docker service ps elk_stack_filebeat
```

### 3.4. Cập nhật và xóa Stack

Để cập nhật cấu hình sau khi chỉnh sửa file docker-stack.yml:

```bash
# Cập nhật stack với cấu hình mới
docker stack deploy -c docker-stack.yml elk_stack
```

Để xóa stack ELK khỏi Docker Swarm:

```bash
# Xóa toàn bộ stack ELK
docker stack rm elk_stack

# Kiểm tra xem stack đã được xóa chưa
docker stack ls
```

## 4. Các thành phần chính

### 4.1. Elasticsearch

Elasticsearch chạy với cấu hình:
- Single-node
- Heap size: 256MB
- Tắt security để đơn giản hóa việc triển khai

#### Truy vấn dữ liệu trong Elasticsearch:

```bash
# Kiểm tra trạng thái cluster
curl -X GET "http://localhost:9200/_cluster/health?pretty"

# Liệt kê tất cả các index
curl -X GET "http://localhost:9200/_cat/indices?v"

# Truy vấn logs từ service rng
curl -X GET "http://localhost:9200/logs-rng-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  },
  "size": 10,
  "sort": [
    {
      "@timestamp": {
        "order": "desc"
      }
    }
  ]
}'

# Truy vấn logs mức ERROR
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "tags": "error_log"
    }
  },
  "size": 10,
  "sort": [
    {
      "@timestamp": {
        "order": "desc"
      }
    }
  ]
}'
```

### 4.2. Logstash

Logstash chạy với một replica để xử lý và lọc logs:
- Phân loại logs theo từng service (rng, hasher, worker, webui)
- Trích xuất thông tin quan trọng: timestamp, log level, log message
- Đánh tag cho các log cấp độ ERROR
- Tìm kiếm các từ khóa quan trọng như exception, error, timeout,...

### 4.3. Filebeat

Filebeat chạy như một service trong Docker Swarm để thu thập logs từ:
- Logs từ container Docker
- Tự động đính kèm metadata cho Docker
- Gửi dữ liệu đến Logstash qua cổng 5044

### 4.4. Kibana

Kibana kết nối với Elasticsearch, cho phép:
- Tạo Dashboard trực quan về logs
- Phân tích logs theo thời gian
- Tìm kiếm và lọc logs
- Tạo các biểu đồ, bảng và trực quan hóa dữ liệu

## 5. Thao tác với Kibana

### 5.1. Tạo Index Pattern

1. Truy cập Kibana UI tại `http://<IP_máy_ảo>:5601`
2. Vào phần "Stack Management" > "Index Patterns"
3. Tạo index pattern mới: `logs-*`
4. Chọn `@timestamp` làm trường thời gian

### 5.2. Khám phá dữ liệu

1. Vào phần "Discover"
2. Tìm kiếm logs với các filter:
   - `service_name: "rng"`
   - `tags: "error_log"`
   - `log_level: "ERROR"`
   - `message: *exception*`

### 5.3. Tạo Dashboard

1. Vào phần "Dashboard" > "Create dashboard"
2. Thêm các visualization:
   - Biểu đồ phân phối logs theo thời gian
   - Biểu đồ tròn tỷ lệ logs theo service
   - Bảng liệt kê các logs lỗi gần đây
   - Heat map về mức độ hoạt động của các service

## 6. Kết quả và đánh giá

Hệ thống ELK Stack đã được triển khai thành công, cho phép:
- Thu thập logs từ 4 service: rng, hasher, worker, webui
- Xử lý và lọc các thông tin quan trọng từ logs
- Lưu trữ logs có cấu trúc trong Elasticsearch
- Trực quan hóa và phân tích logs thông qua Kibana
- Hệ thống được tối ưu hóa để hoạt động trên môi trường có giới hạn tài nguyên

## 7. Kết luận

Việc triển khai ELK Stack trên Docker Swarm ngay cả trong môi trường giới hạn tài nguyên vẫn cho phép quản lý và phân tích logs hiệu quả. Giải pháp này phù hợp cho cả môi trường phát triển và môi trường sản xuất nhỏ.

## 8. Khắc phục sự cố thường gặp

### 8.1. Filebeat liên tục khởi động lại

Nếu Filebeat liên tục khởi động lại với lỗi, hãy kiểm tra:

1. **Đảm bảo chạy với quyền root**: Phải có dòng `user: root` trong cấu hình Filebeat
2. **Kiểm tra cú pháp command**: Sử dụng đúng cú pháp `--strict.perms=false` (hai dấu gạch ngang)
3. **Kiểm tra logs**: Sử dụng lệnh `docker service logs elk_stack_filebeat` để xác định lỗi
4. **Kiểm tra quyền truy cập volume**: Đảm bảo có quyền truy cập vào `/var/lib/docker/containers` và `/var/run/docker.sock`

### 8.2. Không có dữ liệu trong Elasticsearch

Nếu logs không xuất hiện trong Elasticsearch:

1. Kiểm tra kết nối từ Filebeat đến Logstash: `curl -v telnet://logstash:5044`
2. Kiểm tra cấu hình output trong filebeat.yml: `output.logstash.hosts: ["logstash:5044"]`
3. Kiểm tra logs của Logstash: `docker service logs elk_stack_logstash`
4. Kiểm tra lọc trong Logstash: Đảm bảo filters trong logstash.conf hoạt động đúng