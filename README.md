## 1. Thông Tin Máy Ảo

- **Tên máy**: `vps0`
- **Cấu hình**:
  - **CPU**: 2 core
  - **RAM**: 4096 MB (4GB)
- **Yêu cầu**: Đảm bảo đã cài Docker và Docker Swarm.

--- 

## 2. Docker image
**Các image sẽ được tải tự động từ Docker Hub khi triển khai:**
- `elastic/elasticsearch:8.17.4`
- `elastic/kibana:8.17.4`
- `elastic/logstash:8.17.4`
- `elastic/filebeat:8.17.4`

**Hoặc dùng lệnh docker pull để tải các image trước khi deploy**
- `sudo docker pull elastic/elasticsearch:8.17.4`
- `sudo docker pull elastic/kibana:8.17.4`
- `sudo docker pull elastic/logstash:8.17.4`
- `sudo docker pull elastic/filebeat:8.17.4`

---

## 3. Clone Repository
- Tất cả file cấu hình (`docker-stack.yml`, `logstash.conf`, `filebeat.yml`) đã được đẩy lên GitHub. Chỉ cần clone về:
  - `git clone https://github.com/vuongng2212/ELK-Stack.git`
  - `cd ELK-Stack`

---

## Các Bước Triển Khai
### 1. Đăng Nhập vào vps0
### 2. Clone repository
### 3. Triển Khai ELK Stack
- Triển khai stack bằng file `docker-stack.yml` trong repository:
  - `sudo docker stack deploy -c docker-stack.yml elk_stack`
  - **Ghi chú**: `elk_stack` là tên stack, có thể thay đổi nếu muốn.
### 4. Kiểm Tra Trạng Thái
- `sudo docker stack ps elk_stack`
- Kiểm tra log của từng dịch vụ:
  - `sudo docker service logs elk_stack_elasticsearch`
  - `sudo docker service logs elk_stack_kibana`
  - `sudo docker service logs elk_stack_logstash`
  - `sudo docker service logs elk_stack_filebeat`
### 5. Truy Cập Dịch Vụ
  - **Elasticsearch**: `http://<vps0_ip>:9200`
  - **Kibana**: `http://<vps0_ip>:5601`
  - **Logstash**: Nhận dữ liệu từ Filebeat qua cổng `5044`
### 5. Gỡ ELK Stack
  - `sudo docker stack rm elk_stack`