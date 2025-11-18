# 폴더 구조

## 📋 목표

FastAPI 프로젝트의 전체 폴더 구조를 생성합니다.

---

## 🔧 전체 폴더 구조

```
fmr-api/
├── main.py                     # FastAPI 앱 진입점
├── .env                        # 환경변수
├── .env.example                # 환경변수 템플릿
├── .gitignore                  # Git 무시 파일
├── requirements.txt            # 의존성
├── README.md                   # 프로젝트 설명
│
├── app/                        # 메인 애플리케이션
│   ├── __init__.py
│   ├── config.py              # 설정
│   │
│   ├── api/                   # API 라우터
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── payments.py    # 결제 API
│   │       ├── uploads.py     # 업로드 API
│   │       ├── results.py     # 결과 조회 API
│   │       └── webhooks.py    # Webhook API
│   │
│   ├── core/                  # 핵심 기능
│   │   ├── __init__.py
│   │   ├── database.py        # DB 연결
│   │   ├── exceptions.py      # 커스텀 예외
│   │   ├── security.py        # 보안 유틸
│   │   └── logging.py         # 로깅 설정
│   │
│   ├── models/                # SQLAlchemy 모델
│   │   ├── __init__.py
│   │   ├── job.py
│   │   ├── payment.py
│   │   ├── job_file.py
│   │   ├── sms_log.py
│   │   ├── phone_verification.py
│   │   ├── knowledge.py
│   │   └── report.py
│   │
│   ├── schemas/               # Pydantic 스키마
│   │   ├── __init__.py
│   │   ├── payment.py
│   │   ├── upload.py
│   │   ├── result.py
│   │   └── common.py
│   │
│   ├── services/              # 비즈니스 로직
│   │   ├── __init__.py
│   │   ├── payment_service.py
│   │   ├── image_service.py
│   │   ├── ai_service.py
│   │   ├── report_service.py
│   │   ├── sms_service.py
│   │   ├── s3_service.py
│   │   └── verification_service.py
│   │
│   ├── tasks/                 # Background Tasks
│   │   ├── __init__.py
│   │   ├── ai_generation.py
│   │   └── auto_deletion.py
│   │
│   └── utils/                 # 유틸리티
│       ├── __init__.py
│       ├── image.py           # 이미지 검증
│       ├── phone.py           # 전화번호 검증
│       └── helpers.py         # 기타 헬퍼
│
├── alembic/                   # DB 마이그레이션
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
│
├── tests/                     # 테스트
│   ├── __init__.py
│   ├── conftest.py           # pytest 설정
│   ├── test_payment.py
│   ├── test_upload.py
│   ├── test_ai.py
│   └── test_result.py
│
├── logs/                      # 로그 파일
│   └── .gitkeep
│
└── docs/                      # 문서
    ├── planning/
    ├── technical/
    └── implementation/        # 이 문서들
```

---

## 🔧 폴더 생성 스크립트

### 파일: `create_structure.sh`

```bash
#!/bin/bash

# 메인 폴더
mkdir -p app/api/v1
mkdir -p app/core
mkdir -p app/models
mkdir -p app/schemas
mkdir -p app/services
mkdir -p app/tasks
mkdir -p app/utils
mkdir -p alembic/versions
mkdir -p tests
mkdir -p logs

# __init__.py 생성
touch app/__init__.py
touch app/api/__init__.py
touch app/api/v1/__init__.py
touch app/core/__init__.py
touch app/models/__init__.py
touch app/schemas/__init__.py
touch app/services/__init__.py
touch app/tasks/__init__.py
touch app/utils/__init__.py
touch tests/__init__.py

# .gitkeep 생성
touch logs/.gitkeep

echo "폴더 구조 생성 완료!"
```

### 실행

```bash
chmod +x create_structure.sh
./create_structure.sh
```

---

## 🔧 각 폴더 설명

### `app/api/v1/`
- API 엔드포인트 라우터
- 버전별로 분리 (v1, v2, ...)

### `app/core/`
- 핵심 기능 (DB, 예외, 보안, 로깅)
- 전역적으로 사용되는 유틸리티

### `app/models/`
- SQLAlchemy ORM 모델
- 각 테이블당 1개 파일

### `app/schemas/`
- Pydantic 스키마 (요청/응답)
- API 입출력 검증

### `app/services/`
- 비즈니스 로직
- 각 도메인당 1개 서비스

### `app/tasks/`
- Background Tasks
- 비동기 작업 (AI 생성, 자동 삭제)

### `app/utils/`
- 유틸리티 함수
- 재사용 가능한 헬퍼

### `alembic/`
- DB 마이그레이션 파일
- Alembic 설정

### `tests/`
- 단위 테스트 및 통합 테스트
- pytest 사용

### `logs/`
- 애플리케이션 로그 파일
- .gitignore에 추가

---

## ✅ 확인

```bash
# 폴더 구조 확인
tree -L 3 -I 'venv|__pycache__|*.pyc'

# 또는
find . -type d -not -path '*/venv/*' -not -path '*/__pycache__/*'
```

---

## 📝 체크리스트

- [ ] create_structure.sh 생성
- [ ] 스크립트 실행
- [ ] 모든 폴더 생성 확인
- [ ] __init__.py 파일 확인
- [ ] .gitkeep 파일 확인

---

## 🚀 다음 단계

폴더 구조 완료 → **Phase 2: Database** (02-database/README.md)

