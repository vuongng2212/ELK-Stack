# Hướng dẫn triển khai ELK Stack trên Docker Swarm để thu thập logs từ các service

## Chuẩn bị

Các image cần được chuẩn bị trước:
- docker.elastic.co/elasticsearch/elasticsearch:8.17.4
- docker.elastic.co/kibana/kibana:8.17.4
- docker.elastic.co/logstash/logstash:8.17.4
- docker.elastic.co/beats/filebeat:8.17.4

## Kiểm tra và chuẩn bị môi trường Docker Swarm

1. Kiểm tra các node trong Docker Swarm:
```
docker node ls
```

2. Kiểm tra network overlay `coinswarm` đã tồn tại chưa:
```
docker network ls
```

3. Nếu chưa có network overlay, tạo mới:
```
docker network create --driver overlay --attachable coinswarm
```

## Build và push images cho các service dockercoins

1. Di chuyển vào thư mục orchestration-workshop-mrnam:
```
cd orchestration-workshop-mrnam/dockercoins
```

2. Build các image:
```
docker-compose build
```

3. Tag và push các image lên registry (thay thế USERNAME bằng tên đăng nhập của bạn):
```
export USERNAME=mrnam
docker tag dockercoins_rng $USERNAME/rng
docker tag dockercoins_hasher $USERNAME/hasher
docker tag dockercoins_webui $USERNAME/webui
docker tag dockercoins_worker $USERNAME/worker
docker push $USERNAME/rng
docker push $USERNAME/hasher
docker push $USERNAME/webui
docker push $USERNAME/worker
```

## Triển khai stack

1. Đặt các file cấu hình đã tạo vào cùng thư mục:
   - docker-stack.yml
   - filebeat.yml
   - logstash.conf

2. Triển khai stack:
```
docker stack deploy -c docker-stack.yml dockercoins
```

3. Kiểm tra các service đã được triển khai:
```
docker service ls
```

## Kiểm tra logs

1. Đợi vài phút để các service khởi động hoàn tất

2. Mở Kibana qua trình duyệt web:
   http://vps0:5601

3. Thiết lập index pattern trong Kibana:
   - Truy cập Stack Management > Index Patterns
   - Tạo index pattern mới: `dockercoins-*`
   - Chọn trường thời gian mặc định: `@timestamp`

4. Xem logs trong Discover tab

## Xử lý sự cố

1. Kiểm tra trạng thái các service:
```
docker service ps dockercoins_elasticsearch
docker service ps dockercoins_kibana
docker service ps dockercoins_logstash
docker service ps dockercoins_filebeat
```

2. Xem logs của service có vấn đề:
```
docker service logs dockercoins_logstash
docker service logs dockercoins_filebeat
```

3. Nếu có vấn đề với Elasticsearch:
```
curl -X GET "vps0:9200/_cat/health?v"
```