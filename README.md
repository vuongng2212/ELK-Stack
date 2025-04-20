# Tài liệu triển khai ELK Stack cho CoinSwarm

Dự án này chứa tất cả các tài liệu cần thiết để triển khai ELK Stack (Elasticsearch, Logstash, Kibana) trên Docker Swarm để thu thập và phân tích logs từ các service CoinSwarm. Hệ thống đáp ứng đầy đủ 8 yêu cầu chức năng.

## Cấu trúc tài liệu

Tài liệu được chia thành 3 phần chính để dễ dàng tham khảo:

1. **[ELK-Stack-Requirements-Document.md](ELK-Stack-Requirements-Document.md)**: Chi tiết từng yêu cầu cụ thể
   - Mô tả 8 yêu cầu chức năng
   - Cách triển khai và kiểm tra từng yêu cầu
   - Hướng dẫn xử lý sự cố

2. **[ELK-Stack-Config-Files.md](ELK-Stack-Config-Files.md)**: Các file cấu hình hoàn chỉnh
   - docker-stack-elk.yml
   - logstash.conf
   - Cấu hình log driver
   - Lệnh triển khai

3. **[ELK-Stack-Deployment-Steps.md](ELK-Stack-Deployment-Steps.md)**: Các bước triển khai tuần tự
   - Hướng dẫn từng bước một
   - Các lệnh cụ thể
   - Quy trình kiểm tra

## Yêu cầu hệ thống

- Docker Swarm đã được thiết lập
- Ít nhất 1 node với:
  - RAM: ≥ 4GB
  - CPU: ≥ 2 cores
  - Disk: ≥ 10GB
- Các port cần mở:
  - 9200, 9300 (Elasticsearch)
  - 5601 (Kibana)
  - 5044, 5000, 8080, 9600, 12201 (Logstash)

## 8 yêu cầu chức năng

1. Logstash chạy trên swarm với 2 replicas
2. Test log của logstash đọc logs từ 4 services: rng, hasher, worker, webui
3. Logstash xử lý lấy dữ liệu log theo filter với các thông tin quan trọng
4. Elasticsearch chạy trên swarm
5. Truy vấn dữ liệu trong elasticsearch
6. Kibana chạy trên swarm
7. Kibana kết nối lấy dữ liệu thành công từ ElasticSearch
8. Thao tác Kibana tìm kiếm logs, trực quan, phân tích dữ liệu logs

## Triển khai nhanh

```bash
# Chuẩn bị thư mục làm việc
mkdir -p ~/elk-stack
cd ~/elk-stack

# Tạo các file cấu hình (sao chép từ ELK-Stack-Config-Files.md)

# Triển khai ELK Stack
docker stack deploy -c docker-stack-elk.yml elk

# Cấu hình log driver cho các service
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 redis
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 rng
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 hasher
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 worker
docker service update --log-driver gelf --log-opt gelf-address=udp://192.168.186.101:12201 webui
```

Chi tiết hơn, vui lòng tham khảo [ELK-Stack-Deployment-Steps.md](ELK-Stack-Deployment-Steps.md)