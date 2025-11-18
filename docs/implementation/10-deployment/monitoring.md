# 모니터링 및 알림 (온프레미스)

## 📋 개요

**온프레미스 서버**에서 Prometheus + Grafana로 모니터링

**모니터링 스택:**
- ✅ **Prometheus** - 메트릭 수집 (온프레미스)
- ✅ **Grafana** - 시각화 대시보드 (온프레미스)
- ✅ **Sentry** - 에러 추적 (SaaS)
- ✅ **Alertmanager** - 알림 (온프레미스)

---

## 🎯 목적

1. **실시간 모니터링** - 시스템 상태 실시간 추적
2. **에러 추적** - Sentry로 에러 자동 수집
3. **알림** - 임계값 초과 시 Slack/Email 알림
4. **대시보드** - Grafana로 시각화

---

## 🔧 docker-compose 모니터링 스택

### 파일: `docker-compose.monitoring.yml`

```yaml
version: '3.8'

services:
  # Prometheus (메트릭 수집)
  prometheus:
    image: prom/prometheus:latest
    container_name: fmr-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped
    networks:
      - fmr-network

  # Grafana (시각화)
  grafana:
    image: grafana/grafana:latest
    container_name: fmr-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
    depends_on:
      - prometheus
    restart: unless-stopped
    networks:
      - fmr-network

  # Alertmanager (알림)
  alertmanager:
    image: prom/alertmanager:latest
    container_name: fmr-alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/config.yml:/etc/alertmanager/config.yml:ro
    restart: unless-stopped
    networks:
      - fmr-network

  # Node Exporter (시스템 메트릭)
  node-exporter:
    image: prom/node-exporter:latest
    container_name: fmr-node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    restart: unless-stopped
    networks:
      - fmr-network

volumes:
  prometheus_data:
  grafana_data:

networks:
  fmr-network:
    external: true
```

### 실행

```bash
# 모니터링 스택 실행
docker-compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d

# 확인
docker-compose ps
```

---

## 🔧 Prometheus 설정

### 파일: `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alertmanager 설정
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Alert 규칙
rule_files:
  - 'alerts.yml'

# Scrape 설정
scrape_configs:
  # FastAPI 애플리케이션
  - job_name: 'fmr-api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: '/metrics'

  # PostgreSQL
  - job_name: 'postgres'
    static_configs:
      - targets: ['db:5432']

  # Redis
  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']

  # Node Exporter (시스템 메트릭)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # Prometheus 자체
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### 파일: `prometheus/alerts.yml`

```yaml
groups:
  - name: fmr_alerts
    interval: 30s
    rules:
      # API 응답 시간
      - alert: HighResponseTime
        expr: http_request_duration_seconds{quantile="0.95"} > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API response time"
          description: "95th percentile response time is {{ $value }}s"

      # 에러율
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate"
          description: "Error rate is {{ $value }}"

      # DB 연결 실패
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database is down"
          description: "PostgreSQL is not responding"

      # 디스크 사용량
      - alert: HighDiskUsage
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space"
          description: "Disk usage is above 90%"

      # 메모리 사용량
      - alert: HighMemoryUsage
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is above 90%"
```

---

## 🔧 Grafana 설정

### 파일: `grafana/datasources/prometheus.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### 파일: `grafana/dashboards/dashboard.yml`

```yaml
apiVersion: 1

providers:
  - name: 'FMR Dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

### Grafana 접속

```
URL: http://your-server:3000
Username: admin
Password: (GRAFANA_PASSWORD 환경 변수)
```

---

## 🔧 Alertmanager 설정

### 파일: `alertmanager/config.yml`

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack'

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'email'
    email_configs:
      - to: 'admin@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: '${SMTP_USERNAME}'
        auth_password: '${SMTP_PASSWORD}'
```

---

## 🔧 FastAPI 메트릭 통합

### 파일: `app/core/metrics.py`

```python
from prometheus_client import Counter, Histogram, Gauge

# HTTP 요청
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# 응답 시간
http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

# 결제
payments_total = Counter(
    'payments_total',
    'Total payments',
    ['status']
)

# AI 생성
ai_generations_total = Counter(
    'ai_generations_total',
    'Total AI generations',
    ['status']
)

# 활성 Job
active_jobs = Gauge(
    'active_jobs',
    'Number of active jobs'
)
```

### 파일: `app/middleware/metrics_middleware.py`

```python
from fastapi import Request
import time
from app.core.metrics import http_requests_total, http_request_duration_seconds

async def metrics_middleware(request: Request, call_next):
    start_time = time.time()
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    
    http_requests_total.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    http_request_duration_seconds.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    return response
```

### 파일: `app/main.py`

```python
from prometheus_client import make_asgi_app
from fastapi import FastAPI

app = FastAPI()

# Prometheus 메트릭 엔드포인트
metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)

# 미들웨어 추가
from app.middleware.metrics_middleware import metrics_middleware
app.middleware("http")(metrics_middleware)
```

---

## 🔧 Sentry 에러 추적

### 설치

```bash
pip install sentry-sdk[fastapi]
```

### 파일: `app/core/sentry.py`

```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

def init_sentry(dsn: str, environment: str):
    sentry_sdk.init(
        dsn=dsn,
        environment=environment,
        integrations=[
            FastApiIntegration(),
            SqlalchemyIntegration(),
        ],
        traces_sample_rate=0.1,  # 10% 트랜잭션 샘플링
        profiles_sample_rate=0.1,
    )
```

### 파일: `app/main.py`

```python
from app.core.sentry import init_sentry
from app.config import settings

# Sentry 초기화
if settings.SENTRY_DSN:
    init_sentry(
        dsn=settings.SENTRY_DSN,
        environment=settings.ENVIRONMENT
    )
```

---

## ✅ 모니터링 체크리스트

- [ ] docker-compose.monitoring.yml 생성
- [ ] Prometheus 설정 (prometheus.yml, alerts.yml)
- [ ] Grafana 설정 (datasources, dashboards)
- [ ] Alertmanager 설정 (Slack/Email)
- [ ] FastAPI 메트릭 통합
- [ ] Sentry 설정
- [ ] 대시보드 구성
- [ ] 알림 테스트

---

## 📚 참고

- **이전 문서**: `ci-cd.md` - CI/CD 파이프라인
- **Prometheus**: https://prometheus.io/docs/
- **Grafana**: https://grafana.com/docs/
- **Sentry**: https://docs.sentry.io/
