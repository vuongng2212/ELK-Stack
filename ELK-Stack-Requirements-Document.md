# Tài liệu triển khai ELK Stack cho ứng dụng CoinSwarm

## Mục lục
1. [Tổng quan](#tổng-quan)
2. [Chuẩn bị môi trường](#chuẩn-bị-môi-trường)
3. [Yêu cầu 1: Logstash chạy trên swarm với 2 replicas](#yêu-cầu-1-logstash-chạy-trên-swarm-với-2-replicas)
4. [Yêu cầu 2: Test log của logstash đọc logs từ 4 services](#yêu-cầu-2-test-log-của-logstash-đọc-logs-từ-4-services)
5. [Yêu cầu 3: Logstash xử lý lấy dữ liệu log theo filter](#yêu-cầu-3-logstash-xử-lý-lấy-dữ-liệu-log-theo-filter)
6. [Yêu cầu 4: Elasticsearch chạy trên swarm](#yêu-cầu-4-elasticsearch-chạy-trên-swarm)
7. [Yêu cầu 5: Truy vấn dữ liệu trong elasticsearch](#yêu-cầu-5-truy-vấn-dữ-liệu-trong-elasticsearch)
8. [Yêu cầu 6: Kibana chạy trên swarm](#yêu-cầu-6-kibana-chạy-trên-swarm)
9. [Yêu cầu 7: Kibana kết nối lấy dữ liệu thành công từ ElasticSearch](#yêu-cầu-7-kibana-kết-nối-lấy-dữ-liệu-thành-công-từ-elasticsearch)
10. [Yêu cầu 8: Thao tác Kibana tìm kiếm logs, trực quan, phân tích dữ liệu logs](#yêu-cầu-8-thao-tác-kibana-tìm-kiếm-logs-trực-quan-phân-tích-dữ-liệu-logs)
11. [Xử lý sự cố thường gặp](#xử-lý-sự-cố-thường-gặp)

## Tổng quan

Tài liệu này mô tả chi tiết 8 yêu cầu cần thiết để triển khai hệ thống ELK Stack (Elasticsearch, Logstash, Kibana) trên Docker Swarm để thu thập, phân tích và trực quan hóa logs từ các service trong ứng dụng CoinSwarm. Mỗi yêu cầu được trình bày riêng biệt với chi tiết triển khai và kiểm tra.

## Chuẩn bị môi trường

### Kiểm tra Docker Swarm

```bash
# Kiểm tra các node trong Docker Swarm
docker node ls
```

### Kiểm tra và tạo network

```bash
# Kiểm tra network coinswarmnet đã tồn tại
docker network ls | grep coinswarmnet

# Nếu chưa có, tạo mới
docker network create --driver overlay --attachable coinswarmnet
```

### Chuẩn bị và lưu file cấu hình

Tạo thư mục làm việc và lưu các file cấu hình:

```bash
mkdir -p ~/elk-stack
cd ~/elk-stack

# Lưu file docker-stack-elk.yml và logstash.conf
# (Xem nội dung chi tiết ở phần yêu cầu tương ứng)
```

## Yêu cầu 1: Logstash chạy trên swarm với 2 replicas

### Cấu hình trong `docker-stack-elk.yml`

```yaml
logstash:
  image: docker.elastic.co/logstash/logstash:8.17.4
  volumes:
    - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
  networks:
    - elk
    - coinswarmnet
  ports:
    - "5044:5044"
    - "5000:5000"
    - "5000:5000/udp"
    - "9600:9600"
    - "8080:8080"
    - "12201:12201/udp"
  deploy:
    replicas: 2  # Cấu hình 2 replicas
    placement:
      constraints:
        - node.hostname == 192.168.186.101
    restart_policy:
      condition: on-failure
  depends_on:
    - elasticsearch
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
```

### Triển khai

```bash
docker stack deploy -c docker-stack-elk.yml elk
```

### Kiểm tra

```bash
# Kiểm tra số lượng replicas của logstash
docker service ls | grep elk_logstash
```

Kết quả mong đợi: REPLICAS hiển thị "2/2"

## Yêu cầu 2: Test log của logstash đọc logs từ 4 services

### Cấu hình log driver cho các service

```bash
# Cập nhật log driver để gửi logs đến Logstash qua GELF
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 redis
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 rng
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 hasher
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 worker
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 webui
```

### Cấu hình GELF input trong logstash.conf

```
input {
  # Các input khác...
  
  # GELF input để thu thập logs từ Docker
  gelf {
    port => 12201
    type => docker
  }
}
```

### Kiểm tra

```bash
# Kiểm tra logs từ Logstash
docker service logs elk_logstash

# Xem logs để tìm thông tin về 4 service cần theo dõi
docker service logs elk_logstash | grep -E "worker|webui|hasher|rng"
```

Ngoài ra, có thể kiểm tra trên Kibana sau khi đã thiết lập index pattern.

## Yêu cầu 3: Logstash xử lý lấy dữ liệu log theo filter với các thông tin quan trọng

### Cấu hình filter trong logstash.conf

```
filter {
  # Thêm trường service_name để dễ truy vấn
  mutate {
    add_field => { "service_name" => "%{container_id}" }
  }
  
  # Filter cho worker
  if [message] =~ "worker" or [image_name] =~ "worker" or [container_name] =~ "worker" {
    mutate { 
      add_field => { "[@metadata][app]" => "worker" }
      add_field => { "app_type" => "worker" }
    }
    grok {
      match => { 
        "message" => [
          "\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:log_level}: %{GREEDYDATA:log_message}",
          "%{GREEDYDATA:message}"
        ]
      }
      overwrite => [ "message" ]
    }
    
    # Trích xuất thông tin liên quan đến errors và warnings
    if [log_level] =~ "ERROR" {
      mutate { add_tag => ["error_log", "important"] }
    }
    else if [log_level] =~ "WARN" {
      mutate { add_tag => ["warning_log"] }
    }
    
    # Trích xuất thông tin về request từ RNG và Hasher
    if [log_message] =~ ".*Computing.*" {
      grok {
        match => { "log_message" => "Computing (?<operation_type>\w+)" }
      }
    }
  }
  
  # Filter cho các service khác (webui, hasher, rng, redis)
  # (Chi tiết theo tài liệu ELK-Stack-Config-Files.md)
  
  # Phát hiện lỗi hoặc exception trong message
  if [message] =~ ".*(error|exception|fail|timeout).*" or [log_message] =~ ".*(error|exception|fail|timeout).*" {
    mutate { add_tag => ["contains_error_keywords", "important"] }
  }
}
```

### Kiểm tra

Trong Kibana, tìm kiếm các tag đã được thêm vào logs:
- Tìm `tags:error_log`
- Tìm `tags:warning_log`
- Tìm `tags:important`
- Tìm `tags:contains_error_keywords`

Lưu ý: Nếu không tìm thấy kết quả bằng cách tìm theo tags, hãy thử mở rộng khung thời gian tìm kiếm và tìm kiếm theo các trường khác.

## Yêu cầu 4: Elasticsearch chạy trên swarm

### Cấu hình trong docker-stack-elk.yml

```yaml
elasticsearch:
  image: docker.elastic.co/elasticsearch/elasticsearch:8.17.4
  environment:
    - discovery.type=single-node
    - xpack.security.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  networks:
    - coinswarmnet
    - elk
  ports:
    - "9200:9200"
    - "9300:9300"
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == 192.168.186.101
    restart_policy:
      condition: on-failure
  volumes:
    - esdata:/usr/share/elasticsearch/data
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
```

### Kiểm tra

```bash
# Kiểm tra trạng thái service elasticsearch
docker service ls | grep elk_elasticsearch

# Kiểm tra xem service có chạy không
curl http://192.168.186.101:9200/_cat/health
```

Kết quả mong đợi: Trạng thái của service là "running" và curl trả về thông tin sức khỏe của cluster.

## Yêu cầu 5: Truy vấn dữ liệu trong elasticsearch

### Truy vấn thông qua API

```bash
# Kiểm tra các indices
curl http://192.168.186.101:9200/_cat/indices?v

# Tìm kiếm tất cả logs
curl -X GET "http://192.168.186.101:9200/dockercoins-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  },
  "size": 10
}
'

# Tìm logs từ service worker
curl -X GET "http://192.168.186.101:9200/dockercoins-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "app_type": "worker"
    }
  },
  "size": 10
}
'

# Tìm logs có chứa lỗi
curl -X GET "http://192.168.186.101:9200/dockercoins-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "tags": "error_log"
    }
  },
  "size": 10
}
'
```

### Truy vấn thông qua Kibana

Xem phần Yêu cầu 8 về cách tìm kiếm trong Kibana.

## Yêu cầu 6: Kibana chạy trên swarm

### Cấu hình trong docker-stack-elk.yml

```yaml
kibana:
  image: docker.elastic.co/kibana/kibana:8.17.4
  environment:
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    - XPACK_SECURITY_ENABLED=false
  networks:
    - elk
    - coinswarmnet
  ports:
    - "5601:5601"
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == 192.168.186.101
    restart_policy:
      condition: on-failure
  depends_on:
    - elasticsearch
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
```

### Kiểm tra

```bash
# Kiểm tra trạng thái service kibana
docker service ls | grep elk_kibana

# Kiểm tra logs của kibana
docker service logs elk_kibana
```

Ngoài ra, truy cập Kibana qua trình duyệt: http://192.168.186.101:5601

## Yêu cầu 7: Kibana kết nối lấy dữ liệu thành công từ ElasticSearch

### Tạo index pattern trong Kibana

1. Truy cập Kibana: http://192.168.186.101:5601
2. Đi đến "Stack Management" > "Data Views" > "Create data view"
3. Nhập:
   - Index pattern: `dockercoins-*`
   - Time field: `@timestamp`
4. Nhấp vào "Save data view to Kibana"

### Kiểm tra

1. Đi đến "Discover" tab
2. Chọn index pattern vừa tạo
3. Mở rộng khung thời gian tìm kiếm (chọn "Last 24 hours" hoặc "Last 7 days")
4. Nếu dữ liệu hiển thị, việc kết nối đã thành công

## Yêu cầu 8: Thao tác Kibana tìm kiếm logs, trực quan, phân tích dữ liệu logs

### Tìm kiếm logs

1. Đi đến "Discover" tab trong Kibana
2. **Điều chỉnh khung thời gian**:
   - Ở góc trên bên phải, nhấp vào bộ chọn thời gian (thường hiển thị "Last 15 minutes")
   - Chọn khoảng thời gian lớn hơn, ví dụ: "Last 24 hours", "Last 7 days", hoặc "Last 30 days"
   - Nhấp "Apply" để áp dụng

3. Sử dụng các truy vấn:
   - Tìm logs từ worker: `app_type:worker` hoặc `*worker*` hoặc `image_name:*worker*` hoặc `container_name:*worker*`
   - Tìm logs từ webui: `app_type:webui` hoặc `*webui*` hoặc `image_name:*webui*` hoặc `container_name:*webui*`
   - Tìm logs từ hasher: `app_type:hasher` hoặc `*hasher*` hoặc `image_name:*hasher*` hoặc `container_name:*hasher*`
   - Tìm logs từ rng: `app_type:rng` hoặc `*rng*` hoặc `image_name:*rng*` hoặc `container_name:*rng*`
   - Tìm logs lỗi: `tags:error_log OR tags:important`

### Trực quan hóa dữ liệu

#### Biểu đồ phân bố logs theo service

1. Đi đến "Visualize Library" > "Create visualization"
2. Chọn "Pie"
3. Chọn index pattern `dockercoins-*`
4. Thêm "Bucket" > "Split slices" > "Terms"
5. Chọn field "app_type" hoặc "image_name" (tùy thuộc vào trường nào có dữ liệu)
6. Mở rộng khung thời gian tìm kiếm nếu cần
7. Nhấn "Update"

#### Biểu đồ số lượng logs theo thời gian

1. Đi đến "Visualize Library" > "Create visualization"
2. Chọn "Line"
3. Chọn index pattern `dockercoins-*`
4. Thêm "Bucket" > "X-axis" > "Date Histogram"
5. Chọn field "@timestamp"
6. Thêm "Bucket" > "Split series" > "Terms"
7. Chọn field "app_type" hoặc "image_name" (tùy thuộc vào trường nào có dữ liệu)
8. Mở rộng khung thời gian tìm kiếm nếu cần
9. Nhấn "Update"

### Tạo Dashboard

1. Đi đến "Dashboard" > "Create dashboard"
2. Nhấn "Add" để thêm các visualization đã tạo
3. Sắp xếp các visualization theo ý muốn
4. Nhấn "Save" > Đặt tên "CoinSwarm Monitoring Dashboard"

## Xử lý sự cố thường gặp

### 1. Không thấy kết quả trong Kibana khi tìm kiếm

**Nguyên nhân có thể:**
- Khung thời gian tìm kiếm quá nhỏ (mặc định chỉ 15 phút)
- Trường tìm kiếm không tồn tại trong dữ liệu
- Filter trong Logstash không gán đúng giá trị cho các trường

**Giải pháp:**
1. **Mở rộng khung thời gian tìm kiếm**:
   - Chọn "Last 24 hours", "Last 7 days", hoặc "Last 30 days" thay vì mặc định 15 phút
   - Hoặc sử dụng "Absolute" để chọn khoảng thời gian cụ thể (ví dụ: từ 1 tuần trước)

2. **Thử tìm kiếm với các truy vấn khác**:
   - Thay vì `app_type:worker`, hãy thử `*worker*` (tìm worker ở bất kỳ trường nào)
   - Thử tìm theo trường `image_name` hoặc `container_name`: `image_name:*worker*`
   - Sử dụng truy vấn KQL thay vì Lucene: `message : *worker*`

3. **Kiểm tra các trường có sẵn trong dữ liệu**:
   - Trong Discover, mở rộng một document bất kỳ để xem các trường thực tế
   - Xem danh sách trường ở sidebar bên trái trong tab Discover
   - Tìm kiếm theo các trường có sẵn

4. **Cập nhật cấu hình Logstash và buộc cập nhật service**:
   ```bash
   sudo nano ~/elk-stack/logstash.conf
   # Chỉnh sửa file theo hướng dẫn
   docker service update --force elk_logstash
   ```

5. **Tạo lại index pattern trong Kibana**:
   - Đi đến Stack Management > Data Views
   - Nhấp "Create data view"
   - Nhập pattern `dockercoins-*`, chọn trường thời gian `@timestamp`

### 2. Logstash không nhận được logs

**Nguyên nhân có thể:**
- Port 12201/udp không được mở
- Log driver không đúng
- Network không đúng
- Địa chỉ IP không chính xác

**Giải pháp:**
- Kiểm tra port: `docker service inspect elk_logstash --format "{{.Endpoint.Ports}}"`
- Kiểm tra log driver: `docker service inspect redis --pretty | grep LogDriver`
- Kiểm tra networks: `docker service inspect redis --format "{{.Spec.TaskTemplate.Networks}}"`
- Kiểm tra và cập nhật lại địa chỉ IP đúng:
  ```bash
  IP_ADDRESS=$(hostname -I | awk '{print $1}')
  echo "Cập nhật log driver với địa chỉ: $IP_ADDRESS"
  docker service update --log-driver gelf --log-opt gelf-address=udp://$IP_ADDRESS:12201 redis
  # Tương tự cho các service khác
  ```

### 3. Elasticsearch không khởi động

**Nguyên nhân có thể:**
- Thiếu bộ nhớ
- Cấu hình không đúng

**Giải pháp:**
- Kiểm tra logs: `docker service logs elk_elasticsearch`
- Tăng giới hạn bộ nhớ: Chỉnh sửa `ES_JAVA_OPTS=-Xms512m -Xmx512m`

### 4. Kibana không kết nối được với Elasticsearch

**Nguyên nhân có thể:**
- Elasticsearch chưa khởi động
- Cấu hình ELASTICSEARCH_HOSTS không đúng

**Giải pháp:**
- Kiểm tra Elasticsearch: `curl http://192.168.186.101:9200`
- Kiểm tra logs của Kibana: `docker service logs elk_kibana` 