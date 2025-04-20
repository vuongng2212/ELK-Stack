# Hướng dẫn triển khai và kiểm tra ELK Stack

Tài liệu này hướng dẫn thực hiện triển khai và kiểm tra ELK Stack theo 8 yêu cầu cụ thể của đề bài.

## Các yêu cầu triển khai

1. Logstash chạy trên swarm với 2 replicas
2. Test log của logstash đọc logs từ 4 services: rng, hasher, worker, webui
3. Logstash xử lý lấy dữ liệu log theo filter với các thông tin quan trọng
4. Elasticsearch chạy trên swarm
5. Truy vấn dữ liệu trong elasticsearch
6. Kibana chạy trên swarm
7. Kibana kết nối lấy dữ liệu thành công từ ElasticSearch
8. Thao tác Kibana tìm kiếm logs, trực quan, phân tích dữ liệu logs

## Cấu trúc của Project

```
elk-stack/
├── docker-stack.yml   # Cấu hình Docker Swarm Stack
├── filebeat.yml       # Cấu hình Filebeat
└── logstash.conf      # Cấu hình filter Logstash
```

## Bước 1: Triển khai ELK Stack

### 1.1. Khởi tạo Docker Swarm (nếu chưa có)

```bash
docker swarm init --advertise-addr <IP_máy_ảo>
```

### 1.2. Triển khai toàn bộ stack

```bash
# Triển khai stack
docker stack deploy -c docker-stack.yml elk_stack

# Kiểm tra các service đã được triển khai
docker service ls
```

## Bước 2: Kiểm tra các yêu cầu

### Yêu cầu 1: Logstash chạy trên swarm với 2 replicas

#### Kiểm tra:

```bash
# Kiểm tra số lượng replicas của service logstash
docker service ls | grep logstash

# Kiểm tra chi tiết của service logstash
docker service ps elk_stack_logstash
```

Thông tin cấu hình quan trọng trong `docker-stack.yml`:

```yaml
logstash:
  # ... các cấu hình khác ...
  deploy:
    replicas: 2  # Cấu hình 2 replicas
```

#### Minh họa kết quả:

```
ID                  NAME                   IMAGE                                      NODE                DESIRED STATE       CURRENT STATE        ERROR               PORTS
elk_stack_logstash  docker.elastic.co/logstash/logstash:8.17.4   vps0                Running             Running 10 minutes ago                       
```

### Yêu cầu 2: Test log của logstash đọc logs từ 4 services

#### Tạo logs từ các service:

```bash
# Khởi động lại các service để tạo logs
docker service scale elk_stack_rng=0 && docker service scale elk_stack_rng=1
docker service scale elk_stack_hasher=0 && docker service scale elk_stack_hasher=1
docker service scale elk_stack_worker=0 && docker service scale elk_stack_worker=1
docker service scale elk_stack_webui=0 && docker service scale elk_stack_webui=1
```

#### Kiểm tra logs trong Logstash:

```bash
# Kiểm tra logs của các service
docker service logs elk_stack_logstash | grep -E "rng|hasher|worker|webui"
```

### Yêu cầu 3: Logstash xử lý dữ liệu log theo filter

#### Kiểm tra cấu hình filter trong `logstash.conf`:

```
filter {
  # Xử lý log từ service rng
  if [container][image][name] =~ /.*rng.*/ {
    mutate {
      add_field => { "service_name" => "rng" }
      add_tag => [ "rng_service" ]
    }
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log_level} %{GREEDYDATA:log_message}" }
    }
  }
  
  # Tương tự cho các service khác...
  
  # Lọc log cấp độ ERROR
  if [log_level] == "ERROR" or [log_level] == "FATAL" {
    mutate {
      add_tag => [ "error_log" ]
    }
  }
  
  # Lọc thông tin quan trọng
  if [message] =~ /exception|error|failed|timeout|critical|warning|warn/ {
    mutate {
      add_tag => [ "important" ]
    }
  }
}
```

#### Kiểm tra logs đã được filter:

```bash
# Kiểm tra logs đã có tags
docker service logs elk_stack_logstash | grep "tags"
```

### Yêu cầu 4: Elasticsearch chạy trên swarm

#### Kiểm tra:

```bash
# Kiểm tra trạng thái Elasticsearch
docker service ps elk_stack_elasticsearch

# Kiểm tra sức khỏe cluster
curl -X GET "http://192.168.186.101:9200/_cluster/health?pretty"
```

Kết quả mong đợi:
```json
{
  "cluster_name" : "elk-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  ...
}
```

### Yêu cầu 5: Truy vấn dữ liệu trong Elasticsearch

#### Kiểm tra các indices:

```bash
curl -X GET "http://192.168.186.101:9200/_cat/indices?v"
```

#### Truy vấn dữ liệu:

```bash
# 1. Tìm logs từ service rng
curl -X GET "http://192.168.186.101:9200/logs-rng-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {"match_all": {}},
  "size": 10,
  "sort": [{"@timestamp": {"order": "desc"}}]
}'

# 2. Tìm logs cấp độ ERROR
curl -X GET "http://192.168.186.101:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {"match": {"tags": "error_log"}},
  "size": 10,
  "sort": [{"@timestamp": {"order": "desc"}}]
}'

# 3. Thống kê số lượng log theo service
curl -X GET "http://192.168.186.101:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "service_count": {
      "terms": {"field": "service_name.keyword", "size": 10}
    }
  }
}'
```

### Yêu cầu 6: Kibana chạy trên swarm

#### Kiểm tra:

```bash
# Kiểm tra trạng thái Kibana
docker service ps elk_stack_kibana

# Kiểm tra logs
docker service logs elk_stack_kibana
```

#### Truy cập Kibana:

Mở trình duyệt và truy cập: `http://<IP_máy_ảo>:5601`

### Yêu cầu 7: Kibana kết nối thành công với Elasticsearch

#### Tạo Index Pattern:

1. Truy cập Kibana UI tại `http://<IP_máy_ảo>:5601`
2. Vào "Stack Management" > "Index Patterns"
3. Chọn "Create index pattern"
4. Nhập `logs-*` vào ô "Index pattern name"
5. Nhấn "Next step"
6. Chọn `@timestamp` làm trường thời gian (Time field)
7. Nhấn "Create index pattern"

#### Kiểm tra kết nối:

1. Vào "Dev Tools" trong Kibana
2. Nhập và chạy lệnh sau:
   ```
   GET _cluster/health
   ```
3. Kiểm tra phản hồi để xác nhận kết nối thành công

### Yêu cầu 8: Thao tác Kibana tìm kiếm logs, trực quan, phân tích dữ liệu logs

#### 8.1. Tìm kiếm logs:

1. Vào "Discover" trong menu chính của Kibana
2. Chọn index pattern `logs-*`
3. Sử dụng các truy vấn:
   - `service_name: "rng"` - Tìm logs từ service rng
   - `log_level: "ERROR"` - Tìm logs có cấp độ ERROR
   - `tags: "error_log"` - Tìm logs được đánh dấu là error
   - `message: *exception*` - Tìm logs có chứa từ "exception"

#### 8.2. Tạo Visualizations:

**Biểu đồ đường hiển thị số lượng logs theo thời gian:**

1. Vào "Visualize Library" > "Create visualization" > "Line"
2. Chọn index pattern `logs-*`
3. Cấu hình:
   - Y-axis: Count (metric)
   - X-axis: Date Histogram, field @timestamp
4. Nhấn "Update" và lưu với tên "Logs Distribution Over Time"

**Biểu đồ tròn hiển thị tỷ lệ logs theo service:**

1. Vào "Visualize Library" > "Create visualization" > "Pie"
2. Chọn index pattern `logs-*`
3. Cấu hình:
   - Metric: Count
   - Buckets: Split slices > Terms > service_name.keyword
4. Nhấn "Update" và lưu với tên "Logs by Service"

**Bảng hiển thị các lỗi:**

1. Vào "Visualize Library" > "Create visualization" > "Data Table"
2. Chọn index pattern `logs-*`
3. Thêm filter: `tags: "error_log"`
4. Cấu hình:
   - Metric: Count
   - Buckets: Split rows > Terms > service_name.keyword
5. Nhấn "Update" và lưu với tên "Error Logs by Service"

#### 8.3. Tạo Dashboard:

1. Vào "Dashboard" > "Create dashboard"
2. Nhấn "Add" để thêm các visualization đã tạo:
   - "Logs Distribution Over Time"
   - "Logs by Service"
   - "Error Logs by Service"
3. Sắp xếp và điều chỉnh kích thước phù hợp
4. Thêm bộ lọc thời gian (Last 15 minutes, Last 1 hour, etc.)
5. Nhấn "Save" và đặt tên "ELK Services Monitoring"

## Khắc phục sự cố thường gặp

### Filebeat không thu thập logs

Điều chỉnh cấu hình trong `docker-stack.yml`:

```yaml
filebeat:
  image: docker.elastic.co/beats/filebeat:8.17.4
  user: root  # Bắt buộc phải có
  command: ["filebeat", "-e", "--strict.perms=false"]  # Sử dụng đúng cú pháp
  # ... các cấu hình khác ...
```

### Elasticsearch không nhận logs

1. Kiểm tra filebeat và logstash đang chạy:
   ```bash
   docker service ps elk_stack_filebeat
   docker service ps elk_stack_logstash
   ```

2. Kiểm tra logs của filebeat và logstash:
   ```bash
   docker service logs elk_stack_filebeat
   docker service logs elk_stack_logstash
   ```

### Kibana không kết nối với Elasticsearch

1. Kiểm tra cấu hình trong `docker-stack.yml`:
   ```yaml
   kibana:
     environment:
       - "ELASTICSEARCH_HOSTS=http://elasticsearch:9200"
   ```

2. Kiểm tra logs của Kibana:
   ```bash
   docker service logs elk_stack_kibana
   ``` 