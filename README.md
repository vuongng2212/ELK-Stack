# Báo Cáo Triển Khai ELK Stack trên Docker Swarm

## 1. Giới thiệu hệ thống

Báo cáo này trình bày việc triển khai hệ thống ELK Stack (Elasticsearch, Logstash, Kibana) trên nền tảng Docker Swarm với 5 máy chủ (từ vps0 đến vps4). Hệ thống được thiết kế để thu thập, xử lý và trực quan hóa logs từ 4 service: rng, hasher, worker và webui.

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

Khởi tạo Swarm từ máy vps0:

```bash
docker swarm init --advertise-addr <IP_vps0>
```

Tham gia Swarm từ các máy vps1-vps4:

```bash
docker swarm join --token <TOKEN> <IP_vps0>:2377
```

### 3.2. Triển khai Stack

```bash
docker stack deploy -c docker-stack.yml elk
```

### 3.3. Kiểm tra trạng thái các service

```bash
docker service ls
```

## 4. Các thành phần chính

### 4.1. Elasticsearch

Elasticsearch chạy thành công trên Docker Swarm với cấu hình:
- Single-node
- Tên cluster: `elk-cluster`
- Heap size: 512MB
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

Logstash chạy trên Swarm với 2 replicas để xử lý logs. Cấu hình filter cho phép:
- Phân loại logs theo từng service (rng, hasher, worker, webui)
- Trích xuất thông tin quan trọng: timestamp, log level, log message
- Đánh tag cho các log cấp độ ERROR
- Tìm kiếm các từ khóa quan trọng như exception, error, timeout,...

### 4.3. Filebeat

Filebeat được triển khai để thu thập logs từ:
- Các file log truyền thống trong `/var/log/*.log`
- Logs từ container Docker
- Tự động đính kèm metadata cho Docker và Kubernetes

### 4.4. Kibana

Kibana chạy trên Swarm và kết nối thành công với Elasticsearch, cho phép:
- Tạo Dashboard trực quan về logs
- Phân tích logs theo thời gian
- Tìm kiếm và lọc logs
- Tạo các biểu đồ, bảng và trực quan hóa dữ liệu

## 5. Thao tác với Kibana

### 5.1. Tạo Index Pattern

1. Truy cập Kibana UI tại `http://<IP_vps0>:5601`
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

Hệ thống ELK Stack đã được triển khai thành công trên Docker Swarm, cho phép:
- Thu thập logs từ 4 service: rng, hasher, worker, webui
- Xử lý và lọc các thông tin quan trọng từ logs
- Lưu trữ logs có cấu trúc trong Elasticsearch
- Trực quan hóa và phân tích logs thông qua Kibana
- Hệ thống hoạt động ổn định và có khả năng mở rộng

## 7. Kết luận

Việc triển khai ELK Stack trên Docker Swarm giúp tối ưu hóa việc quản lý và phân tích logs trong hệ thống phân tán. Với các công cụ này, việc giám sát, debug và phát hiện sự cố trong hệ thống trở nên dễ dàng và hiệu quả hơn.