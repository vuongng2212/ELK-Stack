# File cấu hình hoàn chỉnh cho ELK Stack

Tài liệu này chứa các file cấu hình hoàn chỉnh để triển khai ELK Stack (Elasticsearch, Logstash, Kibana) trên Docker Swarm.

## 1. docker-stack-elk.yml

File này định nghĩa các service Elasticsearch, Kibana và Logstash để triển khai trên Docker Swarm.

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

networks:
  coinswarmnet:
    external: true
  elk:
    driver: overlay

volumes:
  esdata:
    driver: local
```

## 2. logstash.conf

File này cấu hình các input, filter và output cho Logstash.

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
    add_field => { "service_name" => "%{container_id}" }
  }
  
  # Phát hiện loại dịch vụ dựa trên nội dung message hoặc metadata có sẵn
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
  else if [message] =~ "webui" or [image_name] =~ "webui" or [container_name] =~ "webui" {
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
  else if [message] =~ "hasher" or [image_name] =~ "hasher" or [container_name] =~ "hasher" {
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
  else if [message] =~ "rng" or [image_name] =~ "rng" or [container_name] =~ "rng" {
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
  else if [message] =~ "redis" or [image_name] =~ "redis" or [container_name] =~ "redis" {
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
  if [container_name] {
    mutate {
      add_field => { "container_name" => "%{[container_name]}" }
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

## 3. Cấu hình log driver cho các service CoinSwarm

Đây là các lệnh để cấu hình log driver cho các service CoinSwarm để gửi logs đến Logstash:

```bash
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 redis
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 rng
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 hasher
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 worker
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 webui
```

## 4. Lệnh triển khai

```bash
# Kiểm tra và tạo network nếu cần
docker network create --driver overlay --attachable coinswarmnet

# Triển khai ELK Stack
docker stack deploy -c docker-stack-elk.yml elk
```

## 5. Cập nhật Logstash khi có thay đổi cấu hình

Nếu bạn đã thay đổi file logstash.conf và muốn áp dụng thay đổi, hãy sử dụng lệnh:

```bash
# Buộc cập nhật service logstash để áp dụng cấu hình mới
docker service update --force elk_logstash
``` 