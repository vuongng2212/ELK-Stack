# Hướng dẫn triển khai ELK Stack cho CoinSwarm

## File cấu hình hoàn chỉnh

### 1. docker-stack-elk.yml

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
      - coinswarmnet
      - elk
    ports:
      - "9200:9200"
      - "9300:9300"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname == vps0
      restart_policy:
        condition: on-failure
    volumes:
      - esdata:/usr/share/elasticsearch/data
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

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
          - node.hostname == vps0
      restart_policy:
        condition: on-failure
    depends_on:
      - elasticsearch
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

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
      replicas: 2
      placement:
        constraints:
          - node.hostname == vps0
      restart_policy:
        condition: on-failure
    depends_on:
      - elasticsearch
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

networks:
  coinswarmnet:
    external: true
  elk:
    driver: overlay

volumes:
  esdata:
    driver: local
```

### 2. logstash.conf

```
input {
  beats {
    port => 5044
  }
  tcp {
    port => 5000
    codec => json
  }
  udp {
    port => 5000
    codec => json
  }
  http {
    port => 8080
    codec => json
  }
  # Thêm gelf input để hỗ trợ log driver gelf từ Docker
  gelf {
    port => 12201
    type => docker
  }
}

filter {
  # Thêm trường service_name để dễ truy vấn
  mutate {
    add_field => { "service_name" => "%{[container][image][name]}" }
  }
  
  # Phát hiện loại dịch vụ dựa trên nội dung message hoặc metadata có sẵn
  if [message] =~ "worker" or [container][image] =~ "worker" or [container][name] =~ "worker" {
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
  else if [message] =~ "webui" or [container][image] =~ "webui" or [container][name] =~ "webui" {
    mutate { 
      add_field => { "[@metadata][app]" => "webui" }
      add_field => { "app_type" => "webui" }
    }
    grok {
      match => { "message" => "%{COMMONAPACHELOG}" }
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
  else if [message] =~ "hasher" or [container][image] =~ "hasher" or [container][name] =~ "hasher" {
    mutate { 
      add_field => { "[@metadata][app]" => "hasher" }
      add_field => { "app_type" => "hasher" }
    }
    grok {
      match => { "message" => ".*\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:log_level}: %{GREEDYDATA:log_message}" }
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
  else if [message] =~ "rng" or [container][image] =~ "rng" or [container][name] =~ "rng" {
    mutate { 
      add_field => { "[@metadata][app]" => "rng" }
      add_field => { "app_type" => "rng" }
    }
    grok {
      match => { "message" => ".*\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:log_level}: %{GREEDYDATA:log_message}" }
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
  else if [message] =~ "redis" or [container][image] =~ "redis" or [container][name] =~ "redis" {
    mutate { 
      add_field => { "[@metadata][app]" => "redis" }
      add_field => { "app_type" => "redis" }
    }
  }
  
  # Thêm thông tin thời gian và phân loại
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
    remove_field => [ "timestamp" ]
  }
  
  # Phát hiện lỗi hoặc exception trong message
  if [message] =~ ".*(error|exception|fail|timeout).*" or [log_message] =~ ".*(error|exception|fail|timeout).*" {
    mutate { add_tag => ["contains_error_keywords", "important"] }
  }
  
  # Thêm thông tin về host và container
  if [container][name] {
    mutate {
      add_field => { "container_name" => "%{[container][name]}" }
    }
  }
  
  if [host][name] {
    mutate {
      add_field => { "host_name" => "%{[host][name]}" }
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
    index => "dockercoins-%{+YYYY.MM.dd}"
  }
  
  # Bật đầu ra stdout để debug trong giai đoạn phát triển
  stdout { codec => rubydebug }
}
```

## Các bước triển khai

1. **Kiểm tra network coinswarmnet**:
   ```
   docker network ls
   ```
   Nếu chưa có, tạo mới:
   ```
   docker network create --driver overlay --attachable coinswarmnet
   ```

2. **Lưu các file cấu hình**:
   - Lưu `docker-stack-elk.yml` vào thư mục làm việc
   - Lưu `logstash.conf` vào cùng thư mục

3. **Triển khai ELK Stack**:
   ```
   docker stack deploy -c docker-stack-elk.yml elk
   ```

4. **Kiểm tra các service đã triển khai**:
   ```
   docker stack services elk
   ```

5. **Cấu hình log driver cho các service CoinSwarm**:
   ```
   docker service update --log-driver gelf --log-opt gelf-address=udp://vps0:12201 redis
   docker service update --log-driver gelf --log-opt gelf-address=udp://vps0:12201 rng
   docker service update --log-driver gelf --log-opt gelf-address=udp://vps0:12201 hasher
   docker service update --log-driver gelf --log-opt gelf-address=udp://vps0:12201 worker
   docker service update --log-driver gelf --log-opt gelf-address=udp://vps0:12201 webui
   ```

6. **Kiểm tra logs tại logstash**:
   ```
   docker service logs elk_logstash
   ```

7. **Truy cập Kibana**:
   - URL: http://vps0:5601

8. **Tạo index pattern trong Kibana**:
   - Stack Management > Data Views > Create data view
   - Pattern: `dockercoins-*`
   - Time field: `@timestamp`
   - Nhấn "Save"

9. **Thao tác Kibana tìm kiếm logs**:
   - Đi đến "Discover"
   - Tìm kiếm với queries, ví dụ: `app_type: "worker"` hoặc `tags: "error_log"`

10. **Tạo dashboard trong Kibana**:
    - Đi đến "Dashboard" > "Create dashboard"
    - Thêm các visualization cần thiết

## Kiểm tra yêu cầu

1. **Logstash chạy với 2 replicas**: Đã cấu hình trong file docker-stack-elk.yml
2. **Đọc logs từ 4 services**: Kiểm tra bằng filter trên Kibana
3. **Lọc thông tin quan trọng**: Xem các tags như "important", "error_log"
4. **Elasticsearch chạy trên swarm**: Kiểm tra với `docker stack services elk`
5. **Truy vấn Elasticsearch**: Sử dụng Kibana Discover hoặc API:
   ```
   curl http://vps0:9200/dockercoins-*/_search?pretty
   ```
6. **Kibana chạy trên swarm**: Truy cập http://vps0:5601
7. **Kibana kết nối Elasticsearch**: Tạo và xem được index pattern
8. **Thao tác Kibana phân tích logs**: Sử dụng Discover, Visualize, Dashboard 