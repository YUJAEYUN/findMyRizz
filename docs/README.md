# Find My Rizz - 프로젝트 스펙 문서

## 📋 문서 개요

이 문서는 "Find My Rizz" 서비스의 **완전한 기술 스펙**이자 **슈도코드 기반 구현 가이드**입니다.

### 문서의 목적

1. **기술 스펙 정의**: 모든 기능, API, 데이터베이스 스키마의 명확한 정의
2. **슈도코드 기반 구현**: 코드 작성 전 완전한 구현 명세 제공
3. **일관성 유지**: 모든 개발자가 동일한 이해를 바탕으로 작업
4. **즉시 구현 가능**: 문서만 보고 바로 코딩 시작 가능

---

## 📁 문서 구조

```
docs/
├── README.md                          # 이 문서 (프로젝트 문서 총람)
├── PDD_METHODOLOGY.md                 # PDD 방법론 가이드 ⭐
├── planning/                          # 기획 문서
│   ├── project_plan.md               # 서비스 기획서
│   └── prd_detailed.md               # 상세 PRD
├── technical/                         # 기술 문서
│   ├── technical_architecture.md     # 기술 아키텍처
│   ├── api_specification.md          # API 명세서
│   ├── schema.md                     # 데이터베이스 스키마 (KB 포함)
│   ├── environment_variables.md      # 환경변수 설정
│   ├── portone_v2_integration.md     # PortOne V2 연동
│   ├── phone_verification.md         # 전화번호 인증
│   ├── ai_image_generation_prompt.md # AI 이미지 생성 프롬프트
│   ├── langchain_architecture.md     # LangGraph 아키텍처 (리포트 생성)
│   ├── openai_report_generation.md   # OpenAI 리포트 생성
│   └── library_selection.md          # 라이브러리 선택 가이드
├── implementation/                    # 구현 가이드 (슈도코드 주도)
│   ├── README-PSEUDOCODE-DRIVEN.md   # 슈도코드 주도 개발 가이드
│   ├── 01-setup/                     # 프로젝트 초기 설정
│   ├── 02-database/                  # 데이터베이스 모델 및 마이그레이션
│   ├── 03-payment/                   # 결제 시스템 구현
│   ├── 04-image-upload/              # 이미지 업로드 구현
│   ├── 05-ai-generation/             # AI 이미지 생성 구현
│   ├── 06-report-generation/         # 리포트 생성 구현
│   ├── 07-sms/                       # SMS 발송 구현
│   ├── 08-result-view/               # 결과 조회 구현
│   ├── 09-security/                  # 보안 기능 구현
│   └── 10-deployment/                # 배포 및 운영
├── security/                          # 보안 및 정책
│   ├── security_privacy_policy.md    # 보안 및 프라이버시 정책
│   └── error_handling.md             # 에러 핸들링 전략
└── review/                            # 검토 문서
    └── review_and_corrections.md     # 문서 검토 및 수정 사항
```

---

## 📚 문서 목록

### 0. PDD 방법론 가이드 ⭐

#### [`PDD_METHODOLOGY.md`](./PDD_METHODOLOGY.md)

**Pseudocode-Driven Development 방법론**

이 프로젝트가 따르는 **PDD (슈도코드 주도 개발)** 방법론의 완전한 가이드입니다.

**주요 내용:**

- PDD 개요 및 핵심 원칙
- 5계층 문서 구조 (Layer 0~4)
- 슈도코드 작성 7원칙
- AI 할루시네이션 방지 전략
- 실전 예시 및 체크리스트

**이 문서를 먼저 읽으면 전체 문서 구조를 이해하는 데 도움이 됩니다.**

---

### 1. 기획 문서 (`planning/`)

#### [`planning/project_plan.md`](./planning/project_plan.md)

**서비스 기획서**

- 서비스 컨셉 및 목표
- 타겟 고객
- 핵심 고객 경험 (CX)
- 사용자 시나리오
- 향후 고도화 방향

**주요 내용:**

- 9,900원 1회 결제 모델
- 회원가입 없는 간편한 UX
- 24시간 자동 데이터 파기
- AI 기반 이미지 3장 생성
- 맞춤형 관리법 리포트

#### [`planning/prd_detailed.md`](./planning/prd_detailed.md)

**상세 PRD (Product Requirements Document)**

- 제품 개요 및 비전
- 핵심 기능 상세 (6개 주요 기능)
- 사용자 스토리
- 기술 요구사항
- 비기능 요구사항
- 성공 지표

**핵심 기능:**

1. 결제 시스템 (PortOne)
2. 이미지 업로드 및 처리 (3:4 크롭)
3. AI 이미지 생성 (Replicate)
4. 리포트 생성 (OpenAI GPT-4o)
5. SMS 발송 (CoolSMS)
6. 결과 조회 (전화번호 인증)

---

### 2. 구현 가이드 (`implementation/`)

#### [`implementation/README-PSEUDOCODE-DRIVEN.md`](./implementation/README-PSEUDOCODE-DRIVEN.md)

**슈도코드 주도 개발 가이드**

- 슈도코드 주도 개발 방법론
- 기존 문서 vs 슈도코드 주도 문서 비교
- 완전한 함수 시그니처 및 구현 명세
- 즉시 코딩 가능한 상세 가이드

**핵심 원칙:**

- 코드 작성 전 완전한 구현 명세 작성
- 라인 바이 라인 구현 가이드
- 실제 사용할 변수명 및 타입 명시
- 모든 에러 케이스 처리 방법 포함

#### 구현 단계별 문서

**Phase 1: 프로젝트 초기 설정 (`01-setup/`)**

- `01-project-init.md`: FastAPI 프로젝트 초기화
- `02-environment.md`: 환경변수 설정
- `03-dependencies.md`: 의존성 관리
- `04-folder-structure.md`: 폴더 구조 설계

**Phase 2: 데이터베이스 (`02-database/`)**

- `connection.md`: PostgreSQL 연결 설정
- `migrations.md`: Alembic 마이그레이션
- `seed-data.md`: 초기 데이터 삽입
- `models/`: SQLAlchemy 모델 정의 (12개 모델)
  - `job.md`, `payment.md`, `job-file.md`
  - `kb-category.md`, `kb-procedure.md`, `kb-self-care-item.md`
  - `report.md`, `report-knowledge-item.md`
  - `sms-log.md`, `payment-failure.md`, `phone-verification.md`
  - `satisfaction-survey.md`

**Phase 3: 결제 시스템 (`03-payment/`)**

- `api.md`: 결제 API 엔드포인트
- `schemas.md`: Pydantic 스키마
- `services/`: 결제 서비스 로직
- `02-payment-initialize-DETAILED.md`: 결제 초기화 상세 구현

**Phase 4: 이미지 업로드 (`04-image-upload/`)**

- `api.md`: 이미지 업로드 API
- `services/`: S3 업로드 및 검증 로직
- `03-upload-api-DETAILED.md`: 업로드 API 상세 구현

**Phase 5: AI 이미지 생성 (`05-ai-generation/`)**

- `tasks.md`: Background Task 구현
- `services/`: Replicate API 연동

**Phase 6: 리포트 생성 (`06-report-generation/`)**

- `api.md`: 리포트 생성 API
- `services/`: OpenAI API 연동 및 매칭 로직

**Phase 7: SMS 발송 (`07-sms/`)**

- `services/`: CoolSMS API 연동

**Phase 8: 결과 조회 (`08-result-view/`)**

- `api.md`: 결과 조회 API
- `schemas.md`: 응답 스키마
- `services/`: 전화번호 인증 및 결과 조회

**Phase 9: 보안 기능 (`09-security/`)**

- `auto-deletion.md`: 24시간 자동 파기
- `rate-limiting.md`: API Rate Limiting
- `error-handling.md`: 에러 처리
- `logging.md`: 로깅 및 모니터링

**Phase 10: 배포 및 운영 (`10-deployment/`)**

- `docker.md`: Docker 컨테이너화
- `ci-cd.md`: CI/CD 파이프라인
- `monitoring.md`: 모니터링 및 알림

---

### 3. 기술 문서 (`technical/`)

#### [`technical/technical_architecture.md`](./technical/technical_architecture.md)

**기술 아키텍처 설계**

- 시스템 아키텍처 다이어그램
- 기술 스택 정의
- 데이터 플로우
- 핵심 컴포넌트 설계
- 성능 최적화 전략
- 확장성 고려사항

**기술 스택:**

- Backend: FastAPI (Python 3.11+)
- Database: PostgreSQL 15+
- Image Generation: Replicate AI
- Report Generation: OpenAI GPT-4o
- Payment: PortOne (웰컴페이먼츠)
- SMS: CoolSMS
- Storage: AWS S3

#### [`technical/api_specification.md`](./technical/api_specification.md)

**API 명세서**

- 모든 엔드포인트 정의
- Request/Response 스펙
- 에러 코드 정의
- Webhook 명세

**주요 API:**

- 결제 API (초기화, 검증, 실패 처리)
- Job 관리 API (상태 조회, 전화번호 업데이트)
- 이미지 업로드 API
- 결과 조회 API (전화번호 인증, 결과 조회, 다운로드)
- 만족도 조사 API

#### [`technical/schema.md`](./technical/schema.md)

**데이터베이스 스키마**

- 전체 테이블 정의 (DDL)
- 관계 설정
- 인덱스 전략
- 초기 데이터 (DML)

**주요 테이블:**

- `jobs`: 작업 정보
- `payments`: 결제 정보
- `job_files`: 파일 정보 (원본, 생성 이미지)
- `job_recommendations`: AI 추천 관리법
- `knowledge_base`: 관리법 지식베이스 (시술, 홈케어)
- `sms_logs`: SMS 발송 이력
- `payment_failures`: 결제 실패 로그
- `phone_verification_attempts`: 전화번호 인증 시도 로그

#### [`technical/ai_image_generation_prompt.md`](./technical/ai_image_generation_prompt.md)

**AI 이미지 생성 프롬프트**

- Replicate AI 이미지 생성 API용 프롬프트
- 3장 생성 방식 (다른 seed 사용)
- API 호출 예시 코드

**핵심 원칙:**

- 자연스럽고 현실적인 개선
- 원본 얼굴 특징 보존
- 과장 없는 뷰티 향상

---

### 4. 보안 및 정책 (`security/`)

#### [`security/security_privacy_policy.md`](./security/security_privacy_policy.md)

**보안 및 프라이버시 정책**

- 개인정보 처리 방침
- 데이터 보안 (암호화, 접근 제어)
- 24시간 자동 파기 정책
- 전화번호 인증 보안
- 결제 보안
- 이미지 보안 (S3 Presigned URL)
- API 보안 (Rate Limiting)
- 컴플라이언스 (개인정보보호법, 전자상거래법)

**핵심 보안 원칙:**

- HTTPS/TLS 1.3 필수
- 전송 중/저장 시 암호화
- 최소 권한 원칙
- 24시간 후 완전 삭제

#### [`security/error_handling.md`](./security/error_handling.md)

**에러 핸들링 및 예외 처리**

- 에러 처리 원칙
- 단계별 에러 시나리오 (8개 카테고리)
- 재시도 정책
- 모니터링 및 알림

**에러 카테고리:**

- 결제 에러 (E-PAY-XXX)
- 이미지 에러 (E-IMG-XXX)
- AI 생성 에러 (E-AI-XXX)
- 리포트 에러 (E-RPT-XXX)
- SMS 에러 (E-SMS-XXX)
- 인증 에러 (E-AUTH-XXX)
- 파일 에러 (E-FILE-XXX)
- 시스템 에러 (E-SYS-XXX)

---

### 5. 검토 문서 (`review/`)

#### [`review/review_and_corrections.md`](./review/review_and_corrections.md)

**문서 검토 및 수정 사항**

- 발견된 논리적 모순 및 해결 방안
- 데이터베이스 스키마 개선 사항
- API 명세서 개선 사항
- 보안 정책 개선 사항
- 권장 개발 순서 (8주 계획)
- 최종 체크리스트

**주요 수정 사항:**

1. 전화번호 입력 시점: 결제 **후**로 변경 (개인정보 최소 수집 및 결제 장벽 최소화)
2. 리포트 형식: 웹페이지 (PDF 제거)
3. 이미지 생성: 3회 API 호출 (다른 seed)

---

## 🎯 빠른 시작 가이드

### 개발자를 위한 문서 읽기 순서

**1단계: 기획 및 설계 이해**

1. `planning/project_plan.md` - 서비스 전체 개요
2. `planning/prd_detailed.md` - 상세 기능 요구사항
3. `technical/technical_architecture.md` - 기술 아키텍처
4. `technical/api_specification.md` - API 명세서
5. `technical/schema.md` - 데이터베이스 스키마

**2단계: 구현 준비**

6. `implementation/README-PSEUDOCODE-DRIVEN.md` - 슈도코드 주도 개발 방법론
7. `implementation/01-setup/` - 프로젝트 초기 설정
8. `implementation/02-database/` - 데이터베이스 모델 및 마이그레이션

**3단계: 기능별 구현**

9. `implementation/03-payment/` - 결제 시스템
10. `implementation/04-image-upload/` - 이미지 업로드
11. `implementation/05-ai-generation/` - AI 이미지 생성
12. `implementation/06-report-generation/` - 리포트 생성
13. `implementation/07-sms/` - SMS 발송
14. `implementation/08-result-view/` - 결과 조회

**4단계: 보안 및 배포**

15. `security/security_privacy_policy.md` - 보안 정책
16. `security/error_handling.md` - 에러 처리
17. `implementation/09-security/` - 보안 기능 구현
18. `implementation/10-deployment/` - 배포 및 운영

**5단계: 최종 검토**

19. `review/review_and_corrections.md` - 문서 검토 및 수정 사항

### PM/기획자를 위한 문서 읽기 순서

1. `planning/project_plan.md` - 서비스 전체 개요
2. `planning/prd_detailed.md` - 상세 기능 요구사항
3. `review/review_and_corrections.md` - 최종 검토 사항

---

## 🔒 보안 및 컴플라이언스

### 개인정보 보호

- 24시간 자동 데이터 파기
- 전화번호 인증 (3회 실패 시 차단)
- S3 Presigned URL (1시간 유효)
- HTTPS/TLS 1.3 필수

### 법적 준수

- 개인정보보호법
- 전자상거래법 (결제 정보 5년 보관)
- 정보통신망법

---

## 📖 문서 활용 가이드

### 이 문서를 활용하는 방법

1. **스펙 확인**: 구현하려는 기능의 요구사항과 스펙을 정확히 파악
2. **슈도코드 참조**: `implementation/` 디렉토리의 상세 구현 가이드 확인
3. **코드 작성**: 슈도코드를 실제 코드로 변환
4. **검증**: API 명세서와 스키마 문서로 구현 검증

### 문서 업데이트 원칙

- 모든 변경사항은 관련된 모든 문서에 반영
- 슈도코드와 실제 구현의 일관성 유지
- 변경 이유를 `review/review_and_corrections.md`에 기록

---

## 🎯 핵심 원칙

### 1. 스펙 우선 (Spec-First)

코드 작성 전 스펙을 먼저 정의하고 합의

### 2. 슈도코드 주도 (Pseudocode-Driven)

구현 전 슈도코드로 로직을 완전히 설계

### 3. 문서와 코드의 일치 (Documentation-Code Parity)

문서와 실제 코드가 항상 동기화

### 4. 일관성 유지 (Consistency)

모든 문서 간 논리적 일관성 보장
