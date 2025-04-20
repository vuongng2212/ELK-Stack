# Hướng dẫn truy vấn dữ liệu trong Elasticsearch

## 1. Giới thiệu về API của Elasticsearch

Elasticsearch cung cấp REST API dễ sử dụng để tương tác với dữ liệu. Hầu hết các thao tác được thực hiện qua HTTP request với dữ liệu được truyền dưới dạng JSON.

## 2. Kiểm tra trạng thái Cluster

```bash
# Kiểm tra sức khỏe của cluster
curl -X GET "http://localhost:9200/_cluster/health?pretty"

# Xem thông tin node
curl -X GET "http://localhost:9200/_cat/nodes?v"

# Xem thông tin các index
curl -X GET "http://localhost:9200/_cat/indices?v"
```

## 3. Tìm kiếm cơ bản

### 3.1. Tìm kiếm tất cả dữ liệu trong một index

```bash
curl -X GET "http://localhost:9200/logs-rng-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  },
  "size": 20
}'
```

### 3.2. Tìm kiếm theo trường cụ thể

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "service_name": "rng"
    }
  }
}'
```

## 4. Truy vấn nâng cao

### 4.1. Truy vấn Boolean

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "service_name": "worker" } }
      ],
      "should": [
        { "match": { "log_level": "ERROR" } },
        { "match": { "log_level": "WARN" } }
      ],
      "minimum_should_match": 1,
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1d" } } }
      ]
    }
  }
}'
```

### 4.2. Truy vấn phạm vi (Range)

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2023-05-01T00:00:00",
        "lt": "2023-05-31T23:59:59"
      }
    }
  }
}'
```

### 4.3. Truy vấn text đầy đủ

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "query_string": {
      "query": "log_message:exception OR log_message:error",
      "default_field": "log_message"
    }
  }
}'
```

### 4.4. Tìm kiếm theo prefix hoặc wildcard

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "wildcard": {
      "log_message": "connect*"
    }
  }
}'
```

## 5. Aggregation (Tổng hợp dữ liệu)

### 5.1. Đếm số lượng log theo service

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "service_count": {
      "terms": {
        "field": "service_name.keyword",
        "size": 10
      }
    }
  }
}'
```

### 5.2. Đếm số lượng log theo mức độ (ERROR, INFO...)

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "log_levels": {
      "terms": {
        "field": "log_level.keyword",
        "size": 10
      }
    }
  }
}'
```

### 5.3. Phân tích log theo thời gian

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "logs_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "hour"
      }
    }
  }
}'
```

### 5.4. Kết hợp nhiều aggregation

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "services": {
      "terms": {
        "field": "service_name.keyword",
        "size": 10
      },
      "aggs": {
        "log_levels": {
          "terms": {
            "field": "log_level.keyword",
            "size": 5
          }
        }
      }
    }
  }
}'
```

## 6. Sắp xếp kết quả

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  },
  "sort": [
    { "@timestamp": { "order": "desc" } }
  ]
}'
```

## 7. Phân trang kết quả

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "from": 10,
  "size": 20,
  "query": {
    "match_all": {}
  }
}'
```

## 8. Lọc trường kết quả

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "_source": ["service_name", "log_level", "log_message", "@timestamp"],
  "query": {
    "match": {
      "service_name": "rng"
    }
  }
}'
```

## 9. Highlight (Highlight kết quả tìm kiếm)

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "log_message": "error"
    }
  },
  "highlight": {
    "fields": {
      "log_message": {}
    }
  }
}'
```

## 10. Truy vấn với script

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "filter": {
        "script": {
          "script": "doc[\"service_name.keyword\"].value == \"rng\" && doc[\"log_level.keyword\"].value == \"ERROR\""
        }
      }
    }
  }
}'
```

## 11. Truy vấn theo tag

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "terms": {
      "tags.keyword": ["error_log", "important"]
    }
  }
}'
```

## 12. Truy vấn kết hợp nhiều điều kiện

```bash
curl -X GET "http://localhost:9200/logs-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "service_name": "rng" } }
      ],
      "must_not": [
        { "match": { "log_level": "INFO" } }
      ],
      "should": [
        { "match": { "log_message": "error" } },
        { "match": { "log_message": "exception" } }
      ],
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1d", "lte": "now" } } }
      ]
    }
  }
}'
```

## 13. Cập nhật và xóa dữ liệu

### 13.1. Xóa một document theo ID

```bash
curl -X DELETE "http://localhost:9200/logs-rng-2023.06.01/_doc/document_id_here"
```

### 13.2. Xóa theo truy vấn

```bash
curl -X POST "http://localhost:9200/logs-*/_delete_by_query?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "log_level": "DEBUG"
    }
  }
}'
```

## 14. Quản lý Index

### 14.1. Tạo index với các mapping tùy chỉnh

```bash
curl -X PUT "http://localhost:9200/custom-logs?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "service_name": { "type": "keyword" },
      "log_level": { "type": "keyword" },
      "log_message": { "type": "text" },
      "@timestamp": { "type": "date" }
    }
  }
}'
```

### 14.2. Xem mapping của một index

```bash
curl -X GET "http://localhost:9200/logs-rng-2023.06.01/_mapping?pretty"
```

## 15. Templates và Index Lifecycle Management

### 15.1. Tạo index template

```bash
curl -X PUT "http://localhost:9200/_index_template/logs-template?pretty" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "service_name": { "type": "keyword" },
        "log_level": { "type": "keyword" },
        "log_message": { "type": "text" }
      }
    }
  }
}'
```

### 15.2. Tạo Index Lifecycle Policy

```bash
curl -X PUT "http://localhost:9200/_ilm/policy/logs-policy?pretty" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "5gb"
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}'
```

## 16. Mẹo và best practices

1. **Luôn sử dụng filter thay vì query khi có thể**: Filter được cache và nhanh hơn.
2. **Sử dụng tham số size=0 khi chỉ cần aggregation**: Tăng hiệu suất khi không cần document.
3. **Sử dụng paging hoặc scroll API cho tập kết quả lớn**: Tránh OutOfMemory khi truy vấn.
4. **Sử dụng các tham số như preference để đảm bảo kết quả nhất quán**.
5. **Sử dụng _source filtering để giảm kích thước response**. 