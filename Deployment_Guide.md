# Hướng dẫn Triển khai ELK Stack trên Docker Swarm

Tài liệu này cung cấp hướng dẫn chi tiết để triển khai ELK Stack (Elasticsearch, Logstash, Kibana) trên Docker Swarm để thu thập và phân tích logs từ các service đã được triển khai sẵn trong ứng dụng DockerCoins.

## 1. Chuẩn bị môi trường

### 1.1. Kiểm tra môi trường Docker Swarm

Đảm bảo Docker Swarm đã được khởi tạo và các node đang hoạt động:

```bash
# Kiểm tra các node trong Docker Swarm
docker node ls
```

### 1.2. Kiểm tra network overlay coinswarm

Kiểm tra xem network overlay `coinswarm` đã tồn tại:

```bash
docker network ls | grep coinswarm
```

### 1.3. Kiểm tra các service đã triển khai

Xác nhận các service worker, hasher, webui, rng đã được triển khai:

```bash
docker service ls
```

### 1.4. Pull images ELK Stack

Pull các images Elastic cần thiết:

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.17.4
docker pull docker.elastic.co/kibana/kibana:8.17.4
docker pull docker.elastic.co/logstash/logstash:8.17.4
docker pull docker.elastic.co/beats/filebeat:8.17.4
```

## 2. Chuẩn bị các file cấu hình

### 2.1. Tạo thư mục làm việc

```bash
mkdir -p ~/elk-stack
cd ~/elk-stack
```

### 2.2. Tạo file docker-stack.yml

Tạo file `docker-stack.yml` với nội dung sau (chỉ bao gồm các service ELK Stack):

```yaml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.4
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    networks:
      - coinswarm
      - elk
    ports:
      - "9200:9200"
      - "9300:9300"
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
    volumes:
      - esdata:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.17.4
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - XPACK_SECURITY_ENABLED=false
    networks:
      - elk
      - coinswarm
    ports:
      - "5601:5601"
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:8.17.4
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    networks:
      - elk
      - coinswarm
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.17.4
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elk
      - coinswarm
    user: root
    deploy:
      mode: global
      restart_policy:
        condition: on-failure
    depends_on:
      - logstash

networks:
  coinswarm:
    external: true
  elk:
    driver: overlay

volumes:
  esdata:
    driver: local
```

### 2.3. Tạo file filebeat.yml

Tạo file `filebeat.yml` với nội dung sau:

```yaml
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'
  json.keys_under_root: true
  json.overwrite_keys: true
  json.add_error_key: true
  json.message_key: log
  processors:
    - add_docker_metadata:
        host: "unix:///var/run/docker.sock"
    - add_fields:
        target: docker
        fields:
          swarm_stack: ${SWARM_STACK_NAME:dockercoins}
    - drop_event:
        when:
          not:
            or:
              - equals:
                  container.image.name: "*/worker"
              - equals:
                  container.image.name: "*/webui"
              - equals:
                  container.image.name: "*/hasher"
              - equals:
                  container.image.name: "*/rng"

filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

setup.ilm.enabled: false
setup.template.enabled: false

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0640
```

### 2.4. Tạo file logstash.conf

Tạo file `logstash.conf` với nội dung sau:

```
input {
  beats {
    port => 5044
  }
}

filter {
  # Thêm trường service_name để dễ truy vấn
  mutate {
    add_field => { "service_name" => "%{[container][image][name]}" }
  }
  
  if [container][image][name] =~ "worker" {
    mutate { 
      add_field => { "[@metadata][app]" => "worker" }
      add_field => { "app_type" => "worker" }
    }
    grok {
      match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:log_level}: %{GREEDYDATA:log_message}" }
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
  else if [container][image][name] =~ "webui" {
    mutate { 
      add_field => { "[@metadata][app]" => "webui" }
      add_field => { "app_type" => "webui" }
    }
    grok {
      match => { "message" => "%{COMMONAPACHELOG}" }
      overwrite => [ "message" ]
    }
    
    # Phân tích request HTTP
    if [request] {
      grok {
        match => { "request" => "%{WORD:http_method} %{URIPATHPARAM:request_path}" }
      }
      
      # Thêm tags cho HTTP errors
      if [status] =~ "^[45]" {
        mutate { add_tag => ["http_error", "important"] }
      }
    }
  }
  else if [container][image][name] =~ "hasher" {
    mutate { 
      add_field => { "[@metadata][app]" => "hasher" }
      add_field => { "app_type" => "hasher" }
    }
    grok {
      match => { "message" => ".*\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:log_level}: %{GREEDYDATA:log_message}" }
      overwrite => [ "message" ]
    }
    
    # Trích xuất thông tin về hasher requests
    if [log_message] =~ ".*hash.*" {
      grok {
        match => { "log_message" => "Computing hash for (?<input_data>[0-9a-f]+)" }
      }
    }
    
    # Tag errors và warnings
    if [log_level] =~ "ERROR" {
      mutate { add_tag => ["error_log", "important"] }
    }
    else if [log_level] =~ "WARN" {
      mutate { add_tag => ["warning_log"] }
    }
  }
  else if [container][image][name] =~ "rng" {
    mutate { 
      add_field => { "[@metadata][app]" => "rng" }
      add_field => { "app_type" => "rng" }
    }
    grok {
      match => { "message" => ".*\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:log_level}: %{GREEDYDATA:log_message}" }
      overwrite => [ "message" ]
    }
    
    # Phát hiện dữ liệu quan trọng trong RNG
    if [log_message] =~ ".*Generated.*" {
      grok {
        match => { "log_message" => "Generated (?<bytes_generated>\d+) bytes" }
      }
    }
    
    # Tag errors và warnings
    if [log_level] =~ "ERROR" {
      mutate { add_tag => ["error_log", "important"] }
    }
    else if [log_level] =~ "WARN" {
      mutate { add_tag => ["warning_log"] }
    }
  }
  
  # Thêm thông tin thời gian và phân loại
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
    remove_field => [ "timestamp" ]
  }
  
  # Phát hiện lỗi hoặc exception trong message
  if [log_message] =~ ".*(error|exception|fail|timeout).*" {
    mutate { add_tag => ["contains_error_keywords", "important"] }
  }
  
  # Thêm thông tin về host và container
  mutate {
    add_field => { 
      "container_name" => "%{[container][name]}"
      "host_name" => "%{[host][name]}"
    }
  }
  
  # Loại bỏ các trường không cần thiết
  mutate {
    remove_field => ["agent", "ecs", "input", "log", "@version"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "dockercoins-%{[@metadata][app]}-%{+YYYY.MM.dd}"
  }
  
  # Trong môi trường phát triển, có thể bật dòng này để debug
  # stdout { codec => rubydebug }
}
```

## 3. Triển khai ELK Stack

Với các file cấu hình đã chuẩn bị, triển khai ELK Stack trên hệ thống Docker Swarm:

```bash
docker stack deploy -c docker-stack.yml elk
```

## 4. Kiểm tra trạng thái các service ELK

```bash
# Kiểm tra tất cả các service đã triển khai
docker service ls

# Kiểm tra chi tiết từng service
docker service ps elk_elasticsearch
docker service ps elk_logstash
docker service ps elk_kibana
docker service ps elk_filebeat
```

## 5. Kiểm tra logs

```bash
# Kiểm tra logs của Filebeat
docker service logs elk_filebeat

# Kiểm tra logs của Logstash
docker service logs elk_logstash

# Kiểm tra logs của Elasticsearch
docker service logs elk_elasticsearch

# Kiểm tra logs của Kibana
docker service logs elk_kibana
```

## 6. Truy cập Kibana

Truy cập giao diện Kibana thông qua trình duyệt web:

```
http://vps0:5601
```

### 6.1. Tạo Index Pattern

1. Đi tới **Stack Management** > **Index Patterns**
2. Chọn **Create index pattern**
3. Nhập pattern: `dockercoins-*`
4. Chọn **Next step**
5. Chọn `@timestamp` làm trường thời gian mặc định
6. Chọn **Create index pattern**

### 6.2. Khám phá dữ liệu

1. Đi tới tab **Discover**
2. Chọn index pattern `dockercoins-*`
3. Thử các tìm kiếm:
   - `app_type: "worker"`
   - `tags: "error_log"`
   - `log_level: "ERROR"`

### 6.3. Tạo Dashboard

1. Đi tới **Dashboard** > **Create dashboard**
2. Chọn **Create visualization**
3. Tạo các visualization để hiển thị dữ liệu logs

## 7. Xác minh các yêu cầu

### 7.1. Kiểm tra replicas của Logstash

```bash
docker service ps elk_logstash
```

Xác nhận có 2 replica đang chạy.

### 7.2. Kiểm tra logs từ 4 services

Trong Kibana, hãy tìm kiếm logs từ mỗi service:

- `app_type: "worker"`
- `app_type: "webui"`
- `app_type: "hasher"`
- `app_type: "rng"`

Bạn có thể thấy Filebeat đang thu thập logs từ tất cả 4 service này.

### 7.3. Kiểm tra Elasticsearch

```bash
# Kiểm tra sức khỏe của Elasticsearch
curl -X GET "http://vps0:9200/_cluster/health?pretty"

# Xem các indices
curl -X GET "http://vps0:9200/_cat/indices?v"
```

### 7.4. Truy vấn dữ liệu trong Elasticsearch

```bash
# Truy vấn logs từ worker
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

## 8. Tạo visualization trong Kibana

### 8.1. Log Count by Service

1. Tạo visualization mới (Bar vertical)
2. Cấu hình:
   - Metrics: Y-axis = Count
   - Buckets: X-axis = Terms, Field = app_type
   - Lưu với tên "Log Count by Service"

### 8.2. Log Trends Over Time

1. Tạo visualization mới (Line)
2. Cấu hình:
   - Metrics: Y-axis = Count
   - Buckets: X-axis = Date Histogram, Field = @timestamp
   - Buckets: Split Series = Terms, Field = log_level
   - Lưu với tên "Log Trends Over Time"

### 8.3. Error Log Table

1. Tạo visualization mới (Data Table)
2. Cấu hình:
   - Filter: tags:error_log OR tags:http_error
   - Columns: @timestamp, service_name, log_message
   - Sort: @timestamp descending
   - Lưu với tên "Recent Errors"

### 8.4. Dashboard

1. Tạo dashboard mới
2. Thêm tất cả visualization đã tạo
3. Lưu với tên "DockerCoins Monitoring"

## 9. Xử lý sự cố

### 9.1. Elasticsearch không khởi động

Kiểm tra logs:

```bash
docker service logs elk_elasticsearch
```

Vấn đề phổ biến và giải pháp:
- Thiếu vm.max_map_count: `sudo sysctl -w vm.max_map_count=262144`
- Kiểm tra quyền truy cập volume: `ls -la /var/lib/docker/volumes/`

### 9.2. Filebeat không nhận được logs

Kiểm tra logs:

```bash
docker service logs elk_filebeat
```

Đảm bảo:
- Filebeat chạy với quyền root
- Có quyền truy cập vào /var/lib/docker/containers
- Cấu hình filter container.image.name phù hợp với tên image thực tế

### 9.3. Logstash không xử lý logs

```bash
docker service logs elk_logstash
```

Đảm bảo:
- Cổng 5044 được mở và Logstash đang lắng nghe
- Grok patterns phù hợp với format logs của các service
- Kết nối được với Elasticsearch

## 10. Dọn dẹp

Khi không cần thiết, bạn có thể xóa stack ELK:

```bash
docker stack rm elk
```

---

## Phụ lục: Các câu lệnh Elasticsearch hữu ích

### Kiểm tra trạng thái Elasticsearch

```bash
curl -X GET "http://vps0:9200/_cluster/health?pretty"
```

### Liệt kê các indices

```bash
curl -X GET "http://vps0:9200/_cat/indices?v"
```

### Truy vấn logs theo tags

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

### Truy vấn logs có từ khóa cụ thể trong log_message

```bash
curl -X GET "http://vps0:9200/dockercoins-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "query_string": {
      "default_field": "log_message",
      "query": "*exception* OR *error*"
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