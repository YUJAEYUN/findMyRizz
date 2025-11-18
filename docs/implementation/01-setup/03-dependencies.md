# 의존성 패키지

## 📋 목표

프로젝트에 필요한 모든 Python 패키지를 정의합니다.

---

## 🔧 requirements.txt

### 파일: `requirements.txt`

```txt
# ===== FastAPI 및 서버 =====
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6

# ===== 데이터베이스 =====
sqlalchemy==2.0.23
alembic==1.12.1
psycopg2-binary==2.9.9

# ===== Background Tasks =====
celery==5.3.4
redis==5.0.1

# ===== Pydantic 설정 =====
pydantic==2.5.0
pydantic-settings==2.1.0

# ===== AWS S3 =====
boto3==1.29.7

# ===== 외부 API 클라이언트 =====
httpx==0.25.2
replicate==0.22.0
openai==1.3.7

# ===== 이미지 처리 =====
Pillow==10.1.0

# ===== 보안 =====
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
slowapi==0.1.9

# ===== 유틸리티 =====
python-dotenv==1.0.0
python-dateutil==2.8.2

# ===== 로깅 =====
structlog==23.2.0

# ===== 테스트 =====
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0
httpx==0.25.2  # 테스트용 HTTP 클라이언트

# ===== 개발 도구 =====
black==23.11.0
isort==5.12.0
flake8==6.1.0
mypy==1.7.1
```

---

## 🔧 설치

```bash
# 가상환경 활성화 후
pip install -r requirements.txt
```

---

## 📦 주요 패키지 설명

### FastAPI 관련

- `fastapi`: 웹 프레임워크
- `uvicorn`: ASGI 서버
- `python-multipart`: 파일 업로드 지원

### 데이터베이스

- `sqlalchemy`: ORM
- `alembic`: 마이그레이션 도구
- `psycopg2-binary`: PostgreSQL 드라이버

### Background Tasks

- `celery`: 분산 작업 큐
- `redis`: 메시지 브로커 및 결과 백엔드

### 외부 API

- `boto3`: AWS S3
- `replicate`: Replicate AI API
- `openai`: OpenAI GPT-4o API
- `httpx`: 비동기 HTTP 클라이언트

### 이미지 처리

- `Pillow`: 이미지 검증 및 처리

### 보안

- `python-jose`: JWT 토큰
- `passlib`: 비밀번호 해싱
- `slowapi`: Rate Limiting

### 테스트

- `pytest`: 테스트 프레임워크
- `pytest-asyncio`: 비동기 테스트
- `pytest-cov`: 코드 커버리지

### 개발 도구

- `black`: 코드 포맷터
- `isort`: import 정렬
- `flake8`: 코드 스타일 검사
- `mypy`: 정적 타입 체크

---

## ✅ 설치 확인

```bash
# 설치된 패키지 확인
pip list

# 특정 패키지 버전 확인
pip show fastapi
pip show sqlalchemy
```

---

## 📝 체크리스트

- [ ] requirements.txt 생성
- [ ] pip install -r requirements.txt 실행
- [ ] 모든 패키지 설치 성공
- [ ] import 테스트 성공

---

## 🚀 다음 단계

의존성 설치 완료 → **04-folder-structure.md** (폴더 구조)
