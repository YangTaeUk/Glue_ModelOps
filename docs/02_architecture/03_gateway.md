# 3. Gateway

> Nginx Gateway 상세 설계

## 역할

| 기능 | 설명 |
|------|------|
| SSL Termination | TLS 1.3 암호화/복호화 |
| Rate Limiting | 요청 속도 제한 |
| Load Balancing | 백엔드 분산 |
| Routing | URL 기반 라우팅 |

## 구성

```
┌─────────────────────────────────────┐
│ Nginx Gateway                       │
│                                     │
│  ┌───────────┐  ┌───────────────┐  │
│  │ SSL/TLS   │  │ Rate Limiter  │  │
│  └─────┬─────┘  └───────┬───────┘  │
│        │                │          │
│        ▼                ▼          │
│  ┌─────────────────────────────┐   │
│  │     Load Balancer           │   │
│  │  (Round Robin / Least Conn) │   │
│  └─────────────────────────────┘   │
│        │                           │
│        ▼                           │
│  ┌─────────────────────────────┐   │
│  │   Upstream: FastAPI Pods    │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

## Nginx 설정

```nginx
# Rate Limiting Zone
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
limit_req_zone $http_authorization zone=user:10m rate=10r/s;

upstream api_backend {
    least_conn;
    server fastapi-1:8000 weight=1;
    server fastapi-2:8000 weight=1;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL
    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
    ssl_protocols TLSv1.3;

    # Rate Limit
    limit_req zone=api burst=50 nodelay;

    # API Routes
    location /v1/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 60s;
        proxy_read_timeout 300s;
    }

    # Streaming (긴 timeout)
    location /v1/stream/ {
        proxy_pass http://api_backend;
        proxy_buffering off;
        proxy_read_timeout 3600s;
    }
}
```

## Rate Limit 정책

| Zone | Rate | Burst | 대상 |
|------|------|-------|------|
| api | 100/s | 50 | IP 기반 |
| user | 10/s | 20 | 사용자 기반 |

## Health Check

```nginx
location /health {
    access_log off;
    return 200 "OK";
}

upstream api_backend {
    server fastapi-1:8000;
    health_check interval=5s fails=3 passes=2;
}
```

## Related

- [02_network.md](./02_network.md) - 통신 프로토콜
- [../01_overview/02_srs.md](../01_overview/02_srs.md) - REQ-SEC-01
