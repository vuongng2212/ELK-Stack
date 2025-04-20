# Triển khai ELK Stack trên Docker Swarm
## Thu thập và phân tích logs từ các service DockerCoins

---

## Tổng quan giải pháp

- **Yêu cầu**: Thu thập và phân tích logs từ 4 service (rng, hasher, worker, webui) trên Docker Swarm
- **Giải pháp**: Triển khai ELK Stack (Elasticsearch, Logstash, Kibana) cùng với Filebeat
- **Mục tiêu**: Giám sát tập trung, phân tích logs và trực quan hóa dữ liệu
- **Giới hạn**: Chỉ vps0 có đủ tài nguyên để chạy ELK Stack

---

## Kiến trúc hệ thống

![ELK Stack Architecture](https://i.imgur.com/XHcvN5v.png)

- **Các service đã triển khai**: rng, hasher, worker, webui (chạy trên network coinswarm)
- **Filebeat**: Thu thập logs từ các container (chạy ở mode global)
- **Logstash**: Xử lý logs với 2 replicas (chạy trên vps0)
- **Elasticsearch**: Lưu trữ và index dữ liệu (chạy trên vps0)
- **Kibana**: Trực quan hóa và phân tích logs (chạy trên vps0)

---

## Thực hiện triển khai

1. **Tạo các file cấu hình**:
   - docker-stack.yml: Định nghĩa các service ELK với placement constraints
   - filebeat.yml: Cấu hình thu thập logs
   - logstash.conf: Cấu hình xử lý và filter logs

2. **Chỉ định triển khai trên vps0**:
   ```yaml
   elasticsearch:
     deploy:
       placement:
         constraints:
           - node.hostname == vps0
   ```

3. **Kiểm tra hoạt động**:
   ```bash
   docker service ls
   docker service ps elk_logstash
   ```

---

## Chức năng 1: Logstash chạy trên swarm với 2 replicas

- Cấu hình trong docker-stack.yml:
  ```yaml
  logstash:
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.hostname == vps0
  ```

- Kiểm tra số lượng replicas:
  ```bash
  docker service ps elk_logstash
  ```

- **Lợi ích**: Tăng khả năng xử lý logs, đảm bảo tính sẵn sàng cao

---

## Chức năng 2: Thu thập logs từ 4 services

- Filebeat cấu hình để theo dõi container logs:
  ```yaml
  filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
  ```

- Filter chỉ thu thập logs từ các service cần thiết:
  ```yaml
  - drop_event:
      when:
        not:
          or:
            - equals:
                container.image.name: "*/worker"
            - equals:
                container.image.name: "*/webui"
            # và các service khác...
  ```

---

## Chức năng 3: Logstash xử lý logs với filter

- Phân loại logs theo service:
  ```
  if [container][image][name] =~ "worker" {
    mutate { 
      add_field => { "[@metadata][app]" => "worker" }
      add_field => { "app_type" => "worker" }
    }
  }
  ```

- Trích xuất thông tin quan trọng:
  ```
  if [log_level] =~ "ERROR" {
    mutate { add_tag => ["error_log", "important"] }
  }
  ```

- Tối ưu dữ liệu để phân tích:
  ```
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }
  ```

---

## Chức năng 4: Elasticsearch lưu trữ dữ liệu

- Cấu hình lưu trữ và tối ưu hóa tài nguyên:
  ```yaml
  elasticsearch:
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - esdata:/usr/share/elasticsearch/data
  ```

- Tạo index theo service và ngày:
  ```
  output {
    elasticsearch {
      index => "dockercoins-%{[@metadata][app]}-%{+YYYY.MM.dd}"
    }
  }
  ```

- **Truy vấn dữ liệu**: REST API để tìm kiếm logs

---

## Chức năng 5: Kibana trực quan hóa dữ liệu logs

![Kibana Dashboard Example](https://i.imgur.com/BLV3eMa.png)

- **Index Pattern**: dockercoins-*
- **Visualization**: Biểu đồ phân bố logs theo service, thời gian
- **Dashboard**: Tổng hợp các visualization quan trọng
- **Discover**: Tìm kiếm và lọc logs theo nhiều tiêu chí

---

## Tối ưu hiệu suất cho vps0

- **Giới hạn bộ nhớ Elasticsearch**:
  ```yaml
  environment:
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  ```

- **Cấu hình Logstash hiệu quả**:
  ```
  filter {
    # Sử dụng mutate và remove_field để giảm kích thước dữ liệu
    mutate {
      remove_field => ["agent", "ecs", "input", "log", "@version"]
    }
  }
  ```

- **Giám sát hiệu năng node vps0**:
  ```bash
  ssh vps0 "top -b -n 1"
  ```

---

## Demo & Kiểm thử

- **Truy cập Kibana**: http://vps0:5601
- **Kiểm tra logs thu thập**: Kibana Discover
- **Hiển thị dashboard**: Biểu đồ phân tích logs theo thời gian
- **Tìm kiếm lỗi**: Tìm logs ERROR và HTTP error codes

---

## Kết luận

- **Đã đạt yêu cầu**:
  1. Logstash chạy với 2 replicas trên vps0 ✓
  2. Thu thập logs từ 4 services ✓
  3. Filter và xử lý logs với thông tin quan trọng ✓
  4. Elasticsearch lưu trữ dữ liệu ✓
  5. Truy vấn logs trong Elasticsearch ✓
  6. Kibana hiển thị và phân tích logs ✓

- **Cải tiến tương lai**:
  - Nâng cấp tài nguyên cho vps0
  - Triển khai Elasticsearch cluster nếu có thêm node mạnh
  - Tối ưu thêm việc lưu trữ logs với Index Lifecycle Management 