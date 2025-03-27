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

**Hoặc dùng lệnh docker pull để tải về các image trước**
- `sudo docker pull elastic/elasticsearch:8.17.4`
- `sudo docker pull elastic/kibana:8.17.4`
- `sudo docker pull elastic/logstash:8.17.4`
- `sudo docker pull elastic/filebeat:8.17.4`