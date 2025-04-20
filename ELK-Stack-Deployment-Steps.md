# Hướng dẫn triển khai ELK Stack

Tài liệu này cung cấp các bước triển khai tuần tự để thiết lập hệ thống ELK Stack (Elasticsearch, Logstash, Kibana) trên Docker Swarm. Mỗi bước được mô tả chi tiết và tuần tự để đảm bảo triển khai thành công.

## Bước 1: Chuẩn bị môi trường

### 1.1. Kiểm tra Docker Swarm

Đảm bảo Docker Swarm đã được khởi tạo:

```bash
# Kiểm tra các node trong Docker Swarm
docker node ls
```

Nếu Swarm chưa được khởi tạo, sử dụng lệnh:

```bash
docker swarm init --advertise-addr <IP_ADDRESS>
```

### 1.2. Kiểm tra và tạo network

```bash
# Kiểm tra network coinswarmnet đã tồn tại
docker network ls | grep coinswarmnet

# Nếu chưa có, tạo mới
docker network create --driver overlay --attachable coinswarmnet
```

### 1.3. Tạo thư mục làm việc

```bash
mkdir -p ~/elk-stack
cd ~/elk-stack
```

## Bước 2: Tạo các file cấu hình

### 2.1. Tạo file docker-stack-elk.yml

Tạo file `docker-stack-elk.yml` với nội dung sau:

```bash
# Sử dụng trình soạn thảo ưa thích để tạo file
nano docker-stack-elk.yml
```

Nội dung file (xem chi tiết trong tài liệu "ELK-Stack-Config-Files.md")

### 2.2. Tạo file logstash.conf

```bash
nano logstash.conf
```

Nội dung file (xem chi tiết trong tài liệu "ELK-Stack-Config-Files.md")

## Bước 3: Triển khai ELK Stack

### 3.1. Deploy stack

```bash
docker stack deploy -c docker-stack-elk.yml elk
```

### 3.2. Kiểm tra services đã được triển khai

```bash
docker stack services elk
```

Kết quả mong đợi:
```
ID             NAME              MODE         REPLICAS   IMAGE                                                 PORTS
abc123def456   elk_elasticsearch replicated   1/1        docker.elastic.co/elasticsearch/elasticsearch:8.17.4  *:9200->9200/tcp, *:9300->9300/tcp
ghi789jkl101   elk_kibana        replicated   1/1        docker.elastic.co/kibana/kibana:8.17.4               *:5601->5601/tcp
mno202pqr303   elk_logstash      replicated   2/2        docker.elastic.co/logstash/logstash:8.17.4           *:5000->5000/tcp, *:5044->5044/tcp, *:8080->8080/tcp, *:9600->9600/tcp, *:5000->5000/udp, *:12201->12201/udp
```

## Bước 4: Cấu hình log driver cho các service CoinSwarm

```bash
# Cập nhật log driver cho service redis
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 redis

# Cập nhật log driver cho service rng
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 rng

# Cập nhật log driver cho service hasher
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 hasher

# Cập nhật log driver cho service worker
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 worker

# Cập nhật log driver cho service webui
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 webui
```

## Bước 5: Kiểm tra logs và kết nối

### 5.1. Kiểm tra logs của Logstash

```bash
docker service logs elk_logstash
```

Xác nhận Logstash đang nhận logs từ các service:

```bash
docker service logs elk_logstash | grep -E "worker|webui|hasher|rng"
```

### 5.2. Kiểm tra sức khỏe của Elasticsearch

```bash
curl http://192.168.186.101:9200/_cat/health
```

### 5.3. Kiểm tra các indices của Elasticsearch

```bash
curl http://192.168.186.101:9200/_cat/indices?v
```

## Bước 6: Thiết lập Kibana

### 6.1. Truy cập Kibana UI

Mở trình duyệt và truy cập URL: `http://192.168.186.101:5601`

### 6.2. Tạo index pattern

1. Đi đến "Stack Management" > "Data Views" > "Create data view"
2. Nhập:
   - Index pattern: `dockercoins-*`
   - Time field: `@timestamp`
3. Nhấp vào "Save data view to Kibana"

## Bước 7: Khám phá dữ liệu trong Kibana

### 7.1. Sử dụng Discover

1. Đi đến "Discover" tab
2. Chọn index pattern `dockercoins-*`
3. Sử dụng các truy vấn:
   - Tìm logs từ worker: `app_type:worker`
   - Tìm logs lỗi: `tags:error_log OR tags:important`

### 7.2. Tạo Visualizations

1. Đi đến "Visualize Library" > "Create visualization"
2. Chọn loại biểu đồ (Pie, Line, ...)
3. Chọn index pattern `dockercoins-*`
4. Thiết lập các thông số theo yêu cầu

### 7.3. Tạo Dashboard

1. Đi đến "Dashboard" > "Create dashboard"
2. Thêm các visualization đã tạo
3. Sắp xếp và lưu dashboard

## Bước 8: Kiểm tra 8 yêu cầu đã hoàn thành

### 8.1. Yêu cầu 1: Logstash chạy với 2 replicas

```bash
docker service ls | grep elk_logstash
```

### 8.2. Yêu cầu 2: Logstash đọc logs từ 4 services

Kiểm tra trong Discover của Kibana bằng cách tìm kiếm:
- `app_type:worker`
- `app_type:webui`
- `app_type:hasher`
- `app_type:rng`

### 8.3 đến 8.8: Kiểm tra các yêu cầu còn lại

Xem chi tiết trong tài liệu "ELK-Stack-Requirements-Document.md"

## Xử lý sự cố nếu có

Chi tiết về xử lý sự cố, xem tài liệu "ELK-Stack-Requirements-Document.md" phần cuối. 