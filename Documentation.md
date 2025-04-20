# Báo Cáo Triển Khai ELK Stack trên Docker Swarm

## 1. Giới thiệu

Báo cáo này trình bày việc triển khai hệ thống ELK Stack (Elasticsearch, Logstash, Kibana) trên nền tảng Docker Swarm để thu thập, xử lý và trực quan hóa logs từ các service đã được triển khai sẵn gồm: rng, hasher, worker và webui trong ứng dụng DockerCoins.

## 2. Kiến trúc Hệ thống

### 2.1. Tổng quan về các thành phần

Hệ thống gồm các thành phần chính:

- **Các service đã triển khai sẵn**: rng, hasher, worker, webui, và redis chạy trong overlay network coinswarm.
- **Filebeat**: Thu thập logs từ các container của 4 service trong ứng dụng DockerCoins.
- **Logstash**: Xử lý logs với 2 replicas, phân tích cú pháp và trích xuất thông tin quan trọng.
- **Elasticsearch**: Lưu trữ và đánh index logs.
- **Kibana**: Giao diện người dùng để trực quan hóa và phân tích logs.

### 2.2. Mô hình triển khai

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

## 3. Triển khai chi tiết

### 3.1. Chuẩn bị môi trường

#### 3.1.1. Kiểm tra môi trường Docker Swarm và network

```bash
# Kiểm tra các node trong Docker Swarm
docker node ls

# Kiểm tra network overlay coinswarm đã tồn tại
docker network ls | grep coinswarm

# Kiểm tra các service đã được triển khai
docker service ls
```

#### 3.1.2. Chuẩn bị các image ELK Stack

```bash
# Pull các image từ docker.elastic.co
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.17.4
docker pull docker.elastic.co/kibana/kibana:8.17.4
docker pull docker.elastic.co/logstash/logstash:8.17.4
docker pull docker.elastic.co/beats/filebeat:8.17.4
```

### 3.2. Triển khai các service ELK Stack

#### 3.2.1. Tạo file docker-stack.yml

File docker-stack.yml định nghĩa các services của ELK Stack, bao gồm:
- Elasticsearch với cấu hình phù hợp
- Logstash với 2 replicas
- Kibana để trực quan hóa dữ liệu
- Filebeat để thu thập logs từ các container

Triển khai stack:

```bash
docker stack deploy -c docker-stack.yml elk
```

Kiểm tra trạng thái các service:

```bash
docker service ls
```

## 4. Minh chứng yêu cầu

### 4.1. Logstash chạy trên swarm với 2 replicas

```bash
# Kiểm tra số lượng replicas của Logstash
docker service ps elk_logstash
```

Kết quả mong đợi: 2 replicas của service Logstash đang chạy trên các node khác nhau của Swarm.

### 4.2. Test log của Logstash đọc logs từ 4 services

```bash
# Xem logs của Logstash
docker service logs elk_logstash
```

Kết quả mong đợi: Logs hiển thị Logstash đang nhận và xử lý dữ liệu từ 4 service (rng, hasher, worker, webui) được triển khai sẵn.

### 4.3. Logstash xử lý dữ liệu log theo filter với thông tin quan trọng

Trong file logstash.conf, chúng ta đã cấu hình các filter để:
- Phân loại logs theo service
- Trích xuất thông tin quan trọng từ mỗi service 
- Tag logs theo mức độ quan trọng (error, warning)
- Phân tích cú pháp nội dung logs để trích xuất thông tin chi tiết

```bash
# Kiểm tra logs Logstash để xác nhận xử lý thành công
docker service logs elk_logstash | grep "filter"
```

### 4.4. Elasticsearch chạy trên swarm

```bash
# Kiểm tra trạng thái Elasticsearch
docker service ps elk_elasticsearch
```

Kết quả mong đợi: Service Elasticsearch đang chạy trên swarm.

### 4.5. Truy vấn dữ liệu trong Elasticsearch

```bash
# Kiểm tra sức khỏe của Elasticsearch
curl -X GET "http://vps0:9200/_cluster/health?pretty"

# Liệt kê các indices trong Elasticsearch
curl -X GET "http://vps0:9200/_cat/indices?v"
```

Truy vấn dữ liệu từ service worker:

```bash
curl -X GET "http://vps0:9200/dockercoins-worker-*/_search?pretty" -H 'Content-Type: application/json' -d'
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
```

Truy vấn logs mức ERROR:

```bash
curl -X GET "http://vps0:9200/dockercoins-*/_search?pretty" -H 'Content-Type: application/json' -d'
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

### 4.6. Kibana chạy trên swarm

```bash
# Kiểm tra trạng thái Kibana
docker service ps elk_kibana
```

Kết quả mong đợi: Service Kibana đang chạy trên swarm.

### 4.7. Kibana kết nối lấy dữ liệu từ Elasticsearch

Truy cập Kibana tại địa chỉ http://vps0:5601 và kiểm tra trạng thái kết nối với Elasticsearch thông qua:
- Status page của Kibana
- Stack Monitoring

Kiểm tra từ command line:

```bash
# Kiểm tra logs của Kibana để xác nhận kết nối với Elasticsearch
docker service logs elk_kibana | grep "elasticsearch"
```

### 4.8. Thao tác Kibana tìm kiếm logs, trực quan, phân tích dữ liệu logs

#### 4.8.1. Tạo Index Pattern

1. Truy cập Kibana UI tại `http://vps0:5601`
2. Vào phần "Stack Management" > "Index Patterns"
3. Tạo index pattern mới: `dockercoins-*`
4. Chọn `@timestamp` làm trường thời gian

#### 4.8.2. Tìm kiếm và lọc logs

Thực hiện trong tab Discover của Kibana:

1. Chọn Index Pattern `dockercoins-*`
2. Tìm kiếm logs với các filter:
   - `app_type: "worker"`
   - `tags: "error_log"`
   - `log_level: "ERROR"`
   - `log_message: *exception*`

3. Lưu các tìm kiếm thường xuyên sử dụng:
   - "Worker Errors"
   - "Web UI HTTP Errors"
   - "All Important Logs"

#### 4.8.3. Tạo Dashboard

1. Vào phần "Dashboard" > "Create dashboard"
2. Thêm các visualization:
   - Biểu đồ phân phối logs theo thời gian
   - Biểu đồ tròn tỷ lệ logs theo service
   - Bảng liệt kê các logs lỗi gần đây
   - Heat map về mức độ hoạt động của các service

#### 4.8.4. Tạo các Visualization

1. Biểu đồ cột "Log Count by Service":
   - Metrics: Count of logs
   - Buckets: Split Series by app_type (Terms aggregation)

2. Biểu đồ đường "Log Trends Over Time":
   - Metrics: Count of logs
   - Buckets: X-axis as @timestamp with interval Auto
   - Buckets: Split Series by log_level (Terms aggregation)

3. Data Table "Recent Errors":
   - Filter: tags:error_log OR tags:http_error
   - Columns: @timestamp, service_name, log_message
   - Sort by @timestamp descending

4. Pie Chart "Log Distribution by Host":
   - Metrics: Count of logs
   - Buckets: Split Slices by host_name (Terms aggregation)

## 5. Xử lý sự cố

### 5.1. Filebeat không thu thập được logs

Nguyên nhân phổ biến:
- Không đủ quyền truy cập vào `/var/lib/docker/containers`
- Cấu hình filebeat.yml không chính xác hoặc không phù hợp với tên image thực tế
- Network không thông giữa Filebeat và Logstash

Giải pháp:
- Kiểm tra quyền của container Filebeat: `docker service logs elk_filebeat`
- Đảm bảo cấu hình `user: root` trong docker-stack.yml
- Kiểm tra kết nối mạng: `docker exec -it <container_id> ping logstash`

### 5.2. Logstash không xử lý logs

Nguyên nhân phổ biến:
- Lỗi cú pháp trong file logstash.conf
- Bộ lọc không phù hợp với định dạng log
- Kết nối đến Elasticsearch bị ngắt

Giải pháp:
- Kiểm tra logs của Logstash: `docker service logs elk_logstash`
- Kiểm tra cú pháp file logstash.conf
- Kiểm tra kết nối đến Elasticsearch: `curl http://elasticsearch:9200`

### 5.3. Elasticsearch không lưu trữ dữ liệu

Nguyên nhân phổ biến:
- Hết dung lượng đĩa
- Cấu hình JVM không phù hợp
- Xung đột index templates

Giải pháp:
- Kiểm tra dung lượng đĩa: `df -h`
- Kiểm tra logs của Elasticsearch: `docker service logs elk_elasticsearch`
- Kiểm tra sức khỏe cluster: `curl -X GET "http://vps0:9200/_cluster/health?pretty"`

## 6. Kết luận và đề xuất

### 6.1. Kết luận

Hệ thống ELK Stack đã được triển khai thành công trên Docker Swarm, đáp ứng các yêu cầu:
- Logstash chạy với 2 replicas
- Thu thập logs từ 4 service DockerCoins đã được triển khai sẵn
- Xử lý và trích xuất thông tin quan trọng từ logs
- Lưu trữ logs trong Elasticsearch
- Trực quan hóa và phân tích dữ liệu qua Kibana

### 6.2. Đề xuất cải tiến

1. **Bảo mật**: Bật xpack.security để bảo vệ dữ liệu và yêu cầu xác thực
2. **Mở rộng**: Cấu hình Elasticsearch cluster với nhiều node cho khả năng mở rộng
3. **Giám sát**: Thêm các công cụ giám sát như Metricbeat để thu thập metrics
4. **Dữ liệu**: Cấu hình Index Lifecycle Management để quản lý vòng đời dữ liệu
5. **Cảnh báo**: Thiết lập cảnh báo qua Kibana Alerting khi phát hiện vấn đề 