# Triển khai ELK Stack trên Docker Swarm
## Thu thập và phân tích logs từ các service DockerCoins

---

## Tổng quan giải pháp

- **Yêu cầu**: Thu thập và phân tích logs từ 4 service (rng, hasher, worker, webui) trên Docker Swarm
- **Giải pháp**: Triển khai ELK Stack (Elasticsearch, Logstash, Kibana) cùng với Filebeat
- **Mục tiêu**: Giám sát tập trung, phân tích logs và trực quan hóa dữ liệu

---

## Kiến trúc hệ thống

![ELK Stack Architecture](https://i.imgur.com/XHcvN5v.png)

- **Các service đã triển khai**: rng, hasher, worker, webui (chạy trên network coinswarm)
- **Filebeat**: Thu thập logs từ các container 
- **Logstash**: Xử lý logs (2 replicas)
- **Elasticsearch**: Lưu trữ và index dữ liệu
- **Kibana**: Trực quan hóa và phân tích logs

---

## Thực hiện triển khai

1. **Tạo các file cấu hình**:
   - docker-stack.yml: Định nghĩa các service ELK
   - filebeat.yml: Cấu hình thu thập logs
   - logstash.conf: Cấu hình xử lý và filter logs

2. **Triển khai trên Docker Swarm**:
   ```bash
   docker stack deploy -c docker-stack.yml elk
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

- Cấu hình lưu trữ:
  ```yaml
  elasticsearch:
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

## Demo & Kiểm thử

- **Tạo logs test**: Thực hiện các request tới các service
- **Kiểm tra logs thu thập**: Kibana Discover
- **Hiển thị dashboard**: Biểu đồ phân tích logs theo thời gian
- **Tìm kiếm lỗi**: Tìm logs ERROR và HTTP error codes

---

## Kết luận

- **Đã đạt yêu cầu**:
  1. Logstash chạy với 2 replicas ✓
  2. Thu thập logs từ 4 services ✓
  3. Filter và xử lý logs với thông tin quan trọng ✓
  4. Elasticsearch lưu trữ dữ liệu ✓
  5. Truy vấn logs trong Elasticsearch ✓
  6. Kibana hiển thị và phân tích logs ✓

- **Cải tiến tương lai**:
  - Bổ sung alerts khi phát hiện lỗi
  - Tích hợp APM để theo dõi hiệu năng
  - Mở rộng Elasticsearch cluster 