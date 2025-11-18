# Find My Rizz - 보안 및 프라이버시 정책

## 문서 정보

- **버전**: 1.0
- **작성일**: 2024-01-01
- **적용 범위**: 전체 서비스

---

## 목차

1. [개인정보 처리 방침](#1-개인정보-처리-방침)
2. [데이터 보안](#2-데이터-보안)
3. [24시간 자동 파기 정책](#3-24시간-자동-파기-정책)
4. [전화번호 인증 보안](#4-전화번호-인증-보안)
5. [결제 보안](#5-결제-보안)
6. [이미지 보안](#6-이미지-보안)
7. [API 보안](#7-api-보안)
8. [인프라 보안](#8-인프라-보안)
9. [컴플라이언스](#9-컴플라이언스)

---

## 1. 개인정보 처리 방침

### 1.1 수집하는 개인정보

**필수 정보:**

- 전화번호 (SMS 발송 및 결과 조회 인증용)
- 얼굴 사진 (AI 분석용)

**선택 정보:**

- 만족도 조사 응답 (익명 처리)

**자동 수집 정보:**

- IP 주소 (보안 및 부정 사용 방지)
- 접속 로그 (서비스 개선)

### 1.2 개인정보 이용 목적

- 서비스 제공 (AI 이미지 생성, 리포트 생성)
- 결과 전송 (SMS 발송)
- 결과 조회 인증
- 서비스 개선 및 통계 분석

### 1.3 개인정보 보유 기간

**원칙: 24시간 자동 파기**

- 얼굴 사진: 업로드 후 24시간
- 생성된 이미지: 생성 후 24시간
- 전화번호: Job 생성 후 24시간
- 리포트 데이터: 생성 후 24시간

**예외: 법적 의무**

- 결제 정보: 전자상거래법에 따라 5년 보관
- 부정 사용 로그: 1년 보관

### 1.4 개인정보 제3자 제공

**제공 대상 및 목적:**
| 제공 대상 | 제공 목적 | 제공 항목 | 보유 기간 |
|-----------|-----------|-----------|-----------|
| PortOne (웰컴페이먼츠) | 결제 처리 | 전화번호, 결제 정보 | 거래 완료 후 5년 |
| CoolSMS | SMS 발송 | 전화번호, 메시지 내용 | 발송 즉시 삭제 |
| Replicate | AI 이미지 생성 | 얼굴 사진 | 생성 완료 즉시 삭제 |
| OpenAI | 리포트 생성 | 이미지 분석 데이터 | 분석 완료 즉시 삭제 |
| AWS | 데이터 저장 | 모든 데이터 | 24시간 |

**제3자 제공 동의:**

- 서비스 이용 시 자동 동의
- 동의 거부 시 서비스 이용 불가

---

## 2. 데이터 보안

### 2.1 데이터 암호화

**전송 중 암호화:**

- HTTPS 사용 (기본)
- 모든 API 통신 암호화

**저장 시 암호화:**

- S3: Server-Side Encryption (SSE-S3) 기본 설정

### 2.2 접근 제어

**데이터베이스:**

- 애플리케이션 전용 계정 사용
- 최소 권한 원칙

**S3:**

- Public Read Bucket (UUID 기반 난독화)
- IAM Role 기반 업로드 권한
- 애플리케이션 레벨 접근 제어
- CORS 정책 설정

**API:**

- Rate Limiting (IP 기반)
- JWT 토큰 인증 (결과 조회)
- CORS 정책 설정

### 2.3 로깅

**기본 로그:**

- 인증 시도 로그
- 결제 트랜잭션 로그
- 에러 로그

---

## 3. 24시간 자동 파기 정책 (Soft Delete)

### 3.1 파기 방식

**Soft Delete 원칙:**

- 물리적 삭제가 아닌 논리적 삭제 (Soft Delete) 방식 사용
- 데이터 복구 및 감사 추적 가능
- 법적 분쟁 대비 및 서비스 개선 분석 목적

### 3.2 파기 대상

**Soft Delete 방식:**

- **Job 레코드의 deleted_at만 설정**
- 전화번호, 파일, 리포트 등 모든 데이터는 그대로 유지
- API에서 `WHERE deleted_at IS NULL` 체크로 접근 차단

**동작 원리:**

```python
# 모든 API에서 deleted_at 체크
job = db.query(Job).filter(
    Job.job_id == job_id,
    Job.deleted_at.is_(None)  # 이것만으로 접근 차단
).first()

if not job:
    raise HTTPException(404, "Not found")
```

**장점:**

- 간단한 구현 (Job 테이블만 UPDATE)
- 완전한 복구 가능 (모든 데이터 보존)
- 법적 분쟁 대비 (원본 데이터 유지)

**영구 보존 대상:**

- 결제 정보 (법적 의무 - 5년)
- 익명화된 통계 데이터
- 만족도 조사 응답 (익명)
- 파기 로그 (감사 추적용)
- Soft Delete된 모든 데이터 (영구 보관)

### 3.3 Soft Delete 프로세스

**매 시간 실행 (Scheduled Task):**

```sql
-- Job 테이블의 deleted_at만 설정
UPDATE jobs
SET
  deleted_at = NOW(),
  status = 'expired'
WHERE expires_at < NOW()
  AND deleted_at IS NULL;
```

**그게 끝입니다!**

- 전화번호, 파일, 리포트 등 모든 데이터는 그대로 유지
- S3 파일도 그대로 유지
- API에서 deleted_at 체크만으로 접근 완전 차단

**데이터 보관 이유:**

- 법적 분쟁 대비
- 서비스 품질 개선 분석
- 사용자 복구 요청 대응
- 완전한 감사 추적

### 3.4 파기 확인 및 복구

**복구 정책:**

- Soft Delete 데이터는 영구 보관되어 언제든 복구 가능
- 관리자 대시보드에서 deleted_at 필드 확인
- 법적 분쟁 시 데이터 제공 가능
- S3 파일도 영구 보관되어 완전한 복구 지원

**복구 방법:**

```sql
-- Job의 deleted_at만 NULL로 변경
UPDATE jobs
SET deleted_at = NULL
WHERE job_id = ?;
```

**그게 끝입니다!**

- 모든 데이터가 그대로 보존되어 있으므로 즉시 복구됨

### 3.5 사용자 안내

**서비스 전반:**

- 랜딩 페이지에 명시: "24시간 후 자동 파기"
- 결제 전 동의 필수
- 결과 페이지에 카운트다운 표시
- SMS에 만료 시간 안내

**개인정보 처리 방침 명시:**

- Soft Delete 방식 사용 안내
- 데이터 영구 보관 명시 (접근만 차단)
- 법적 분쟁 대비 및 서비스 개선 목적 설명
- 복구 요청 가능 (관리자 승인 필요)

**만료 전 알림 (선택사항):**

- 만료 2시간 전 SMS 발송
- "곧 삭제됩니다. 지금 다운로드하세요!"

---

## 4. 전화번호 인증 보안

### 4.1 인증 프로세스

**1단계: 전화번호 입력**

- 하이픈 자동 삽입 (UX)
- 형식 검증 (010-XXXX-XXXX)
- 프론트엔드 + 백엔드 이중 검증

**2단계: 일치 확인**

- Job의 user_phone_number와 비교
- 대소문자 구분 없음
- 공백 제거 후 비교

**3단계: JWT 토큰 발급**

- 유효기간: 1시간
- Payload: job_id, phone_number
- 서명: HS256

### 4.2 보안 조치

**실패 제한:**

- IP 기반: 3회 실패 시 1시간 차단
- Job 기반: 10회 실패 시 영구 차단
- phone_verification_attempts 테이블에 로그

**차단 해제:**

- 시간 경과 후 자동 해제
- 관리자 수동 해제 가능

**로그 저장:**

```sql
INSERT INTO phone_verification_attempts (
    job_id,
    phone_number,
    ip_address,
    success
) VALUES (?, ?, ?, ?);
```

### 4.3 JWT 토큰 관리

**토큰 구조:**

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "phone_number": "010-1234-5678",
  "exp": 1704070800,
  "iat": 1704067200
}
```

**검증:**

- 서명 검증
- 만료 시간 확인
- job_id 존재 확인
- Job 상태 확인 (completed)

---

## 5. 결제 보안

### 5.1 PortOne 연동 보안

**결제 검증:**

- Webhook 서명 검증 (PortOne 제공)
- 금액 일치 확인 (9,900원)
- 중복 결제 방지 (merchant_uid)

**민감 정보 처리:**

- 카드 정보: PortOne에서 처리 (PCI-DSS 준수)
- 결제 정보: 암호화 저장
- 로그: 마스킹 처리 (카드번호 뒷 4자리만)

### 5.2 중복 결제 방지

**merchant_uid 생성:**

```python
merchant_uid = f"FMR_{datetime.now().strftime('%Y%m%d_%H%M%S')}_{random_string(6)}"
```

**중복 체크:**

```sql
SELECT COUNT(*)
FROM jobs
WHERE merchant_uid = ?;
```

**동일 전화번호 체크:**

```sql
SELECT COUNT(*)
FROM jobs
WHERE user_phone_number = ?
  AND created_at > NOW() - INTERVAL '24 hours';
```

### 5.3 환불 정책

**환불 가능 조건:**

- AI 생성 시작 전 (status: pending_upload)
- 기술적 오류로 생성 실패 (status: failed)

**환불 불가 조건:**

- AI 생성 완료 후
- 사용자 귀책 사유 (잘못된 이미지 업로드)

---

## 6. 이미지 보안

### 6.1 업로드 보안

**파일 검증:**

- MIME 타입 검증 (image/\*)
- 파일 확장자 검증

**크기 제한:**

- 최대 10MB
- 최소 512x512

### 6.2 S3 보안

**Bucket 설정: 두 가지 방식 지원**

환경변수 `S3_BUCKET_POLICY`로 선택 가능:

- `private`: Private Bucket + Presigned URL (권장)
- `public`: Public Read Bucket + UUID 난독화

---

#### 방식 1: Private Bucket + Presigned URL (권장)

**보안 전략:**

- S3 Bucket을 완전 Private으로 설정
- 인증된 사용자에게만 Presigned URL 발급
- URL 만료 시간 제어 (1시간)
- 24시간 자동 파기 정책과 일관성

**장점:**

- 높은 보안성: URL 만료 시간 제어
- 접근 제어: 인증된 사용자만 URL 발급
- URL 유출 시에도 1시간 후 자동 만료
- 감사 추적 용이

**단점:**

- Presigned URL 생성 오버헤드
- CDN 캐싱 제한적

**보안 계층:**

1. **Private S3 Bucket**: 모든 객체 기본 비공개
2. **Presigned URL**: 전화번호 인증 성공 시에만 발급 (1시간 유효)
3. **애플리케이션 인증**: JWT 토큰 검증
4. **DB 접근 제어**: 삭제/만료된 파일 URL 제공 안 함

**Bucket Policy (Private):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::findmyrizz-files/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::ACCOUNT_ID:role/FindMyRizzAppRole"
        }
      }
    }
  ]
}
```

**Presigned URL 생성:**

```python
def generate_presigned_url(file_key: str) -> str:
    return s3_client.generate_presigned_url(
        'get_object',
        Params={'Bucket': settings.AWS_S3_BUCKET, 'Key': file_key},
        ExpiresIn=settings.S3_PRESIGNED_URL_EXPIRATION  # 3600초 (1시간)
    )
```

---

#### 방식 2: Public Read Bucket + UUID 난독화

**보안 전략:**

- Public Read 허용
- 파일명을 UUID로 난독화하여 추측 불가능하게 설정
- 전화번호 인증을 통한 접근 제어 (애플리케이션 레벨)
- URL을 모르면 접근 불가능 (UUID 기반 난독화)

**장점:**

- Presigned URL 생성 오버헤드 제거
- CDN 캐싱 최적화 가능
- 이미지 로딩 속도 향상
- 인프라 복잡도 감소

**단점:**

- URL 유출 시 영구 접근 가능 (24시간 동안)
- 보안성 상대적으로 낮음

**보안 계층:**

1. **파일명 난독화**: UUID 기반으로 추측 불가능

   - 예: `550e8400-e29b-41d4-a716-446655440000/original/a1b2c3d4-e5f6-7890-abcd-ef1234567890.png`
   - 브루트포스 공격 불가능 (UUID 공간: 2^122)

2. **애플리케이션 레벨 인증**:

   - 결과 페이지 접근 시 전화번호 인증 필수
   - JWT 토큰으로 인증된 사용자만 S3 URL 제공
   - 3회 실패 시 IP 차단

3. **DB 접근 제어**:
   - deleted_at이 설정된 파일은 URL 제공 안 함
   - 만료된 Job의 파일 URL 제공 안 함

**Bucket Policy (Public Read):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::findmyrizz-files/*"
    }
  ]
}
```

**CORS 설정:**

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET"],
    "AllowedOrigins": ["https://findmyrizz.com"],
    "ExposeHeaders": []
  }
]
```

**파일명 구조:**

- UUID 기반 (추측 불가)
- 경로: `{job_id}/{type}/{uuid}.png`
- 예: `550e8400-e29b-41d4-a716-446655440000/generated/1_a1b2c3d4.png`

---

## 7. API 보안

### 7.1 인증 및 인가

**Public API:**

- 결제 초기화
- 이미지 업로드 가이드 조회
- Health Check

**인증 필요 API:**

- 결과 조회 (JWT)
- 이미지 다운로드 (JWT)

### 7.2 Rate Limiting

**IP 기반:**

- 일반 API: 100 requests/minute
- 결제 API: 10 requests/minute
- 인증 API: 5 requests/minute

**구현:**

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/v1/results/{job_id}/verify")
@limiter.limit("5/minute")
async def verify_phone(job_id: str, phone: str):
    ...
```

### 7.3 입력 검증

**모든 입력 검증:**

- 타입 검증 (Pydantic)
- 길이 제한
- 정규식 검증
- SQL Injection 방어 (ORM 사용)
- XSS 방어 (입력 이스케이프)

**예시:**

```python
from pydantic import BaseModel, validator

class PhoneVerifyRequest(BaseModel):
    phone_number: str

    @validator('phone_number')
    def validate_phone(cls, v):
        if not re.match(r'^010-?\d{4}-?\d{4}$', v):
            raise ValueError('Invalid phone number')
        return v.replace('-', '')
```

---

## 8. 인프라 보안

### 8.1 기본 보안

**Security Group:**

- 필요한 포트만 개방 (443, 5432)
- 기본 방화벽 설정

**애플리케이션 보안:**

- 환경변수로 민감 정보 관리
- AWS Secrets Manager 사용

### 8.2 백업

**데이터베이스:**

- 일일 자동 백업 (RDS 기본 기능)

**S3:**

- 버전 관리 활성화

---

## 9. 컴플라이언스

### 9.1 법적 준수

**개인정보보호법:**

- 개인정보 처리 방침 공개
- 수집 동의 획득
- 24시간 파기 정책

**전자상거래법:**

- 결제 정보 5년 보관
- 환불 정책 명시

**정보통신망법:**

- 개인정보 암호화
- 접근 로그 보관

---

## 10. 사고 대응

### 10.1 보안 사고 대응 절차

**1단계: 탐지**

- 자동 모니터링 알림
- 사용자 신고

**2단계: 격리**

- 영향 범위 파악
- 서비스 일시 중단 (필요 시)

**3단계: 조사**

- 로그 분석
- 원인 파악

**4단계: 복구**

- 취약점 패치
- 데이터 복구

**5단계: 사후 조치**

- 사용자 공지
- 재발 방지 대책

### 10.2 개인정보 유출 대응

**즉시 조치:**

- 유출 범위 파악
- 관련 기관 신고 (개인정보보호위원회)
- 사용자 통지 (24시간 내)

**사후 조치:**

- 피해 보상
- 보안 강화
- 재발 방지

---

## 11. 사용자 권리

### 11.1 개인정보 열람/정정/삭제

**요청 방법:**

- 이메일: privacy@findmyrizz.com
- 처리 기간: 3일 이내

**즉시 삭제:**

- 24시간 경과 전 삭제 요청 시 즉시 처리

### 11.2 동의 철회

**서비스 이용 중:**

- 동의 철회 시 서비스 이용 불가
- 환불 정책에 따라 처리

**서비스 완료 후:**

- 24시간 후 자동 삭제

---

## 12. 문의

**개인정보 보호 책임자:**

- 이름: [담당자명]
- 이메일: privacy@findmyrizz.com
- 전화: 02-XXXX-XXXX

**보안 취약점 신고:**

- 이메일: security@findmyrizz.com
- Bug Bounty 프로그램 (향후 운영)
