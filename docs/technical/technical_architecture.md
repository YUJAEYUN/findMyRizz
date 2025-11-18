# Find My Rizz - 기술 아키텍처 문서

## 1. 시스템 개요

### 1.1 아키텍처 다이어그램

```
┌─────────────┐
│   사용자    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│                    Frontend (Web)                        │
│  - 랜딩 페이지                                           │
│  - 결제 페이지                                           │
│  - 이미지 업로드 & 크롭                                  │
│  - 결과 조회 페이지 (전화번호 인증)                      │
│  - 리포트 페이지                                         │
└──────┬──────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│              FastAPI Backend (Python)                    │
│  ┌─────────────────────────────────────────────┐        │
│  │  API Layer                                  │        │
│  │  - 결제 API                                 │        │
│  │  - 이미지 업로드 API                        │        │
│  │  - Job 상태 조회 API                        │        │
│  │  - 결과 조회 API (전화번호 인증)            │        │
│  │  - 만족도 조사 API                          │        │
│  └─────────────────────────────────────────────┘        │
│  ┌─────────────────────────────────────────────┐        │
│  │  Business Logic Layer                       │        │
│  │  - 결제 처리 로직                           │        │
│  │  - 이미지 처리 로직                         │        │
│  │  - AI 생성 오케스트레이션                   │        │
│  │  - 리포트 생성 로직                         │        │
│  │  - SMS 발송 로직                            │        │
│  └─────────────────────────────────────────────┘        │
│  ┌─────────────────────────────────────────────┐        │
│  │  Background Tasks (Celery + Redis)          │        │
│  │  - AI 이미지 생성 작업                      │        │
│  │  - 리포트 생성 작업                         │        │
│  │  - SMS 발송 작업                            │        │
│  │  - 24시간 후 데이터 파기 작업               │        │
│  └─────────────────────────────────────────────┘        │
└──────┬──────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│                  External Services                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  PortOne     │  │  Replicate   │  │  OpenAI      │   │
│  │  (결제)      │  │  (AI 이미지 생성) │  │  (GPT-4o)    │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │  CoolSMS     │  │  AWS S3      │                     │
│  │  (SMS 발송)  │  │  (파일저장)  │                     │
│  └──────────────┘  └──────────────┘                     │
└──────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│              PostgreSQL Database                          │
│  - jobs (작업 정보)                                       │
│  - payments (결제 정보)                                   │
│  - job_files (파일 정보)                                  │
│  - job_recommendations (추천 관리법)                      │
│  - survey_responses (만족도 조사)                         │
│  - knowledge_base (관리법 지식베이스)                     │
│  - sms_logs (SMS 발송 이력)                               │
│  - payment_failures (결제 실패 로그)                      │
└──────────────────────────────────────────────────────────┘
```

## 2. 기술 스택

### 2.1 Backend

- **Framework**: FastAPI 0.104+
- **Language**: Python 3.11+
- **ASGI Server**: Uvicorn
- **Background Tasks**: Celery + Redis
- **Task Queue**: Redis
- **API Documentation**: Swagger (OpenAPI 3.0)

### 2.2 Database

- **Primary DB**: PostgreSQL 15+
- **ORM**: SQLAlchemy 2.0+
- **Migration**: Alembic

### 2.3 External APIs

- **Payment**: PortOne (웰컴페이먼츠)
- **Image Generation**: Replicate API
- **Report Generation**: OpenAI GPT-4o API
- **SMS**: CoolSMS API
- **Storage**: AWS S3

### 2.4 Infrastructure

- **Hosting**: On-Premise Server
- **Storage**: AWS S3 (파일 저장 전용)
- **Monitoring**: Sentry / Prometheus + Grafana
- **Timezone**: KST (Asia/Seoul)

## 3. 데이터 플로우

### 3.1 전체 사용자 여정 데이터 플로우

```
1. 랜딩 페이지 방문
   ↓
2. "지금 바로 나의 리즈 알아보기" 클릭
   ↓
3. 결제 초기화 → Job 생성 (status: pending_payment)
   ↓
4. PortOne 결제창 호출
   ↓
5. 결제 성공 → Job 상태 업데이트 (status: pending_phone)
   ↓
6. 전화번호 입력 페이지
   ↓
7. 전화번호 등록 → Job 상태 업데이트 (status: pending_upload)
   ↓
8. 사진 가이드 확인
   ↓
9. 이미지 업로드 & 3:4 크롭
   ↓
10. 원본 이미지 S3 업로드 → Job 상태 업데이트 (status: processing)
   ↓
11. Celery Task 시작:
   - Replicate AI 이미지 생성 API 호출 (3장 생성)
   - 생성된 이미지 S3 저장
   - OpenAI GPT-4o로 리포트 생성
   - Job 상태 업데이트 (status: completed)
   ↓
12. CoolSMS로 결과 URL 발송
   ↓
13. 사용자가 SMS 링크 클릭
   ↓
14. 전화번호 인증 페이지
   ↓
15. 인증 성공 → 결과 페이지 표시
   ↓
16. 이미지 3장 + 리포트 조회
   ↓
17. 다운로드 (선택사항)
   ↓
18. 만족도 조사 제출 (선택사항)
   ↓
19. 24시간 후 자동 파기 (Celery Beat Scheduled Task)
```

**시간대 설정:**

- 모든 시간은 KST (Asia/Seoul) 기준으로 저장 및 표시
- Database: `TIMEZONE='Asia/Seoul'` 설정
- Python: `pytz` 또는 `zoneinfo` 사용
- API 응답: ISO 8601 형식 with timezone (예: `2024-01-01T12:00:00+09:00`)

## 4. 핵심 컴포넌트 설계

### 4.1 결제 처리 (PortOne)

**플로우:**

```
Frontend → PortOne SDK 초기화
         → 결제창 호출
         → 결제 성공/실패 콜백
         → Backend Webhook 수신
         → 결제 검증 (PortOne API)
         → Job 생성 또는 상태 업데이트
```

**주요 고려사항:**

- 결제 금액: 9,900원 (부가세 포함)
- Webhook 검증: PortOne 서명 검증 필수
- 중복 결제 방지: idempotency key 사용
- 결제 실패 시 재시도 로직

### 4.2 이미지 처리

**업로드 플로우:**

```
Frontend → 이미지 선택
         → 3:4 비율 크롭 (Canvas API)
         → Base64 또는 FormData로 전송
         → Backend 이미지 검증 (크기, 형식, 얼굴 감지)
         → S3 업로드 (원본)
         → job_files 테이블 저장
```

**이미지 검증:**

- 파일 형식: JPEG, PNG
- 최대 크기: 10MB
- 최소 해상도: 512x512
- 얼굴 감지: OpenCV 또는 Face Detection API (선택사항)

### 4.3 AI 이미지 생성 (Replicate)

**생성 플로우:**

```
Celery Task 시작
  ↓
S3에서 원본 이미지 다운로드
  ↓
Replicate API 호출 (AI 이미지 생성 프롬프트 사용)
  ↓
3장 생성 (동일 프롬프트, 다른 seed)
  ↓
생성된 이미지 S3 업로드
  ↓
job_files 테이블 업데이트
  ↓
Job 상태 업데이트
```

**재시도 정책:**

- 최대 재시도: 3회
- 재시도 간격: 지수 백오프 (5초, 10초, 20초)
- 실패 시 error_message 저장 및 사용자 알림

### 4.4 리포트 생성 (OpenAI GPT-4o)

**생성 플로우:**

```
AI 이미지 생성 완료 후
  ↓
프롬프트 구성:
  - 원본 이미지 분석
  - 생성된 이미지 3장 분석
  - AI 이미지 생성 프롬프트 내용 참조
  ↓
OpenAI GPT-4o API 호출
  ↓
리포트 텍스트 생성 (JSON 형식)
  ↓
knowledge_base에서 관련 관리법 매칭
  ↓
job_recommendations 테이블 저장
  ↓
ai_analysis_text 필드 저장
```

**리포트 구조:**

```json
{
  "summary": "전반적인 개선 방향 요약",
  "improvements": [
    {
      "area": "피부",
      "description": "피부톤 균일화 및 광채 개선",
      "recommendations": ["SC003", "P007"]
    },
    {
      "area": "얼굴 윤곽",
      "description": "턱선 정의 및 V라인 개선",
      "recommendations": ["SC005", "SC007"]
    }
  ],
  "timeline": "8-12주 예상",
  "estimated_cost": "300,000 - 800,000원"
}
```

### 4.5 SMS 발송 (CoolSMS)

**발송 플로우:**

```
Job 완료 후
  ↓
결과 URL 생성: https://findmyrizz.com/result/{job_id}
  ↓
CoolSMS API 호출
  ↓
SMS 발송 이력 저장 (sms_logs 테이블)
  ↓
발송 실패 시 재시도 (최대 3회)
```

**SMS 템플릿:**

```
[Find My Rizz]
나의 리즈 사진이 완성되었습니다!
아래 링크에서 24시간 내에 확인하세요.
{result_url}

※ 24시간 후 자동 삭제됩니다.
```

## 5. 보안 및 프라이버시

### 5.1 전화번호 인증

- 결과 조회 시 전화번호 입력 필수
- 입력된 전화번호와 Job의 user_phone_number 일치 확인
- 3회 실패 시 1시간 차단 (IP 기반)

### 5.2 데이터 파기

- Job 생성 시 expires_at = created_at + 24시간
- Scheduled Task (매 시간 실행):
  - expires_at < NOW() 인 Job 조회
  - S3 파일 삭제
  - DB 레코드 삭제 (CASCADE)

### 5.3 S3 보안

- Bucket Policy: Private
- Presigned URL 사용 (유효기간: 1시간)
- 파일명: UUID 기반 (추측 불가)

## 6. 성능 최적화

### 6.1 이미지 최적화

- 원본 이미지: S3 Standard
- 생성된 이미지: WebP 변환 (선택사항)
- Thumbnail 생성: 리스트 조회용 (선택사항)

### 6.2 데이터베이스 최적화

- 인덱스:
  - jobs(status)
  - jobs(expires_at)
  - jobs(user_phone_number) - 해시 인덱스
  - payments(job_id)
  - job_files(job_id)

### 6.3 캐싱

- knowledge_base: 메모리 캐시 (변경 빈도 낮음)
- API 응답: Redis 캐시 (선택사항)

## 7. 모니터링 및 로깅

### 7.1 로깅

- 구조화된 로깅 (JSON 형식)
- 로그 레벨: DEBUG, INFO, WARNING, ERROR, CRITICAL
- 주요 로깅 포인트:
  - 결제 성공/실패
  - AI 생성 시작/완료/실패
  - SMS 발송 성공/실패
  - 데이터 파기 실행

### 7.2 모니터링

- API 응답 시간
- AI 생성 소요 시간
- 결제 성공률
- SMS 발송 성공률
- 에러율

### 7.3 알림

- 결제 실패율 > 10%
- AI 생성 실패율 > 5%
- SMS 발송 실패율 > 5%
- API 응답 시간 > 3초

## 8. 확장성 고려사항

### 8.1 트래픽 증가 대응

- Application Scaling: Uvicorn 워커 수 증가 또는 서버 추가
- Database: Read Replica 추가
- Background Tasks: Celery 워커 수 증가
- Load Balancing: Nginx 또는 HAProxy 사용

### 8.2 기능 확장

- 스타일 선택 기능: 프롬프트 템플릿 확장
- 맞춤형 리포트: LangChain/LangGraph 도입
- 다국어 지원: i18n 라이브러리 도입

## 9. API 문서화

### 9.1 Swagger (OpenAPI 3.0)

FastAPI는 자동으로 Swagger UI를 생성합니다.

**접근 경로:**

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- OpenAPI JSON: `http://localhost:8000/openapi.json`

**문서화 가이드:**

- 모든 API 엔드포인트에 docstring 작성
- Request/Response 모델에 example 추가
- 에러 응답 명시
- 태그를 사용한 API 그룹화

**예시:**

```python
@router.post(
    "/payments/initialize",
    response_model=PaymentInitResponse,
    tags=["payments"],
    summary="결제 초기화",
    description="PortOne 결제를 위한 초기 Job 생성 및 결제 정보 반환"
)
async def initialize_payment(
    request: PaymentInitRequest
) -> PaymentInitResponse:
    """
    결제 초기화 API

    - **amount**: 결제 금액 (9900 고정)
    """
    pass
```

## 10. 개발 환경 설정

### 10.1 로컬 개발

```bash
# Python 가상환경
python -m venv venv
source venv/bin/activate

# 의존성 설치
pip install -r requirements.txt

# 환경변수 설정
cp .env.example .env

# 데이터베이스 마이그레이션
alembic upgrade head

# 서버 실행
uvicorn main:app --reload
```

### 9.2 개발 워크플로우

**각 태스크별 개발 완료 시 플로우:**

1. **코드 작성 완료**

   ```bash
   # 코드 구현
   ```

2. **린트 실행 (후처리)**

   ```bash
   # Black (코드 포맷팅)
   black .

   # isort (import 정렬)
   isort .

   # Flake8 (코드 스타일 검사)
   flake8 .

   # mypy (타입 체크)
   mypy .
   ```

3. **테스트 코드 작성**

   ```bash
   # pytest를 사용한 단위 테스트 작성
   # tests/ 디렉토리에 테스트 파일 생성
   ```

4. **테스트 실행 및 통과**

   ```bash
   # 전체 테스트 실행
   pytest

   # 커버리지 포함 테스트
   pytest --cov=app --cov-report=html

   # 특정 테스트만 실행
   pytest tests/test_payments.py
   ```

5. **통과 확인 후 다음 태스크 진행**

**테스트 작성 가이드:**

- 각 API 엔드포인트에 대한 단위 테스트 필수
- 성공 케이스 및 실패 케이스 모두 테스트
- Mock을 사용한 외부 API 테스트 (PortOne, Replicate, OpenAI, CoolSMS)
- 데이터베이스 트랜잭션 테스트
- 최소 80% 이상의 코드 커버리지 목표

### 9.3 환경변수

```
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/findmyrizz

# PortOne
PORTONE_API_KEY=your_api_key
PORTONE_API_SECRET=your_api_secret

# Replicate
REPLICATE_API_TOKEN=your_token

# OpenAI
OPENAI_API_KEY=your_api_key

# CoolSMS
COOLSMS_API_KEY=your_api_key
COOLSMS_API_SECRET=your_api_secret

# AWS S3
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_S3_BUCKET=findmyrizz-files
AWS_REGION=ap-northeast-2

# App
SECRET_KEY=your_secret_key
ENVIRONMENT=development
TIMEZONE=Asia/Seoul

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
```
