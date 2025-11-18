# Find My Rizz - API 명세서

## 목차

1. [인증 및 보안](#1-인증-및-보안)
2. [결제 API](#2-결제-api)
3. [Job 관리 API](#3-job-관리-api)
4. [이미지 업로드 API](#4-이미지-업로드-api)
5. [결과 조회 API](#5-결과-조회-api)
6. [만족도 조사 API](#6-만족도-조사-api)
7. [에러 코드](#7-에러-코드)

---

## 1. 인증 및 보안

### 1.1 API 인증

- MVP에서는 별도 인증 없음 (Public API)
- 전화번호 기반 결과 조회 인증만 사용

### 1.2 Rate Limiting

- IP 기반: 100 requests/minute
- 전화번호 인증 실패: 3회/hour

---

## 2. 결제 API

### 2.1 결제 초기화

**Endpoint:** `POST /api/v1/payments/initialize`

**Description:** PortOne 결제를 위한 초기 Job 생성 및 결제 정보 반환

**Request Body:**

```json
{
  "amount": 9900
}
```

**Request Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| amount | integer | Yes | 결제 금액 (9900 고정) |

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "payment_id": "660e8400-e29b-41d4-a716-446655440001",
    "merchant_uid": "FMR_20240101_123456",
    "amount": 9900,
    "portone_config": {
      "pg": "welcome",
      "pay_method": "card",
      "merchant_uid": "FMR_20240101_123456",
      "name": "Find My Rizz - 나의 리즈 찾기",
      "amount": 9900
    }
  }
}
```

**Error Responses:**

- `500 Internal Server Error`: 서버 오류

---

### 2.2 결제 검증 (Webhook)

**Endpoint:** `POST /api/v1/payments/webhook`

**Description:** PortOne에서 결제 완료 후 호출하는 Webhook

**Request Body (PortOne):**

```json
{
  "imp_uid": "imp_123456789",
  "merchant_uid": "FMR_20240101_123456",
  "status": "paid",
  "amount": 9900,
  "paid_at": 1704067200
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Payment verified successfully"
}
```

**Internal Process:**

1. PortOne API로 결제 정보 재검증
2. 금액 일치 확인
3. Payment 상태 업데이트 (`paid`)
4. Job 상태 업데이트 (`pending_phone`)

---

### 2.3 결제 실패 처리

**Endpoint:** `POST /api/v1/payments/failure`

**Description:** 결제 실패 시 프론트엔드에서 호출

**Request Body:**

```json
{
  "payment_id": "660e8400-e29b-41d4-a716-446655440001",
  "error_code": "F001",
  "error_message": "사용자 결제 취소"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Payment failure recorded"
}
```

---

## 3. Job 관리 API

### 3.1 Job 상태 조회

**Endpoint:** `GET /api/v1/jobs/{job_id}/status`

**Description:** Job의 현재 상태 조회

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| job_id | UUID | Job ID |

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "processing",
    "created_at": "2024-01-01T12:00:00Z",
    "expires_at": "2024-01-02T12:00:00Z",
    "progress": {
      "current_step": "generating_images",
      "total_steps": 3,
      "percentage": 66
    }
  }
}
```

**Job Status Values:**

- `pending_payment`: 결제 대기
- `pending_phone`: 전화번호 입력 대기
- `pending_upload`: 이미지 업로드 대기
- `processing`: AI 생성 중
- `completed`: 완료
- `failed`: 실패
- `expired`: 만료 (24시간 경과)

**Job Status 전이 다이어그램:**

```
pending_payment
    ↓ (결제 성공 - Webhook)
pending_phone
    ↓ (전화번호 입력 - PATCH /jobs/{job_id}/phone)
pending_upload
    ↓ (이미지 업로드 - POST /jobs/{job_id}/upload)
processing
    ↓ (AI 생성 완료)        ↓ (AI 생성 실패)
completed                  failed

* 모든 상태에서 24시간 경과 시 → expired
* failed 상태에서는 재시도 불가 (새로운 Job 생성 필요)
* completed 상태는 최종 상태 (변경 불가)
```

**허용되는 상태 전이:**

| 현재 상태       | 다음 가능 상태 | 전이 조건           |
| --------------- | -------------- | ------------------- |
| pending_payment | pending_phone  | 결제 성공 (Webhook) |
| pending_payment | expired        | 24시간 경과         |
| pending_phone   | pending_upload | 전화번호 입력       |
| pending_phone   | expired        | 24시간 경과         |
| pending_upload  | processing     | 이미지 업로드       |
| pending_upload  | expired        | 24시간 경과         |
| processing      | completed      | AI 생성 완료        |
| processing      | failed         | AI 생성 실패        |
| processing      | expired        | 24시간 경과         |
| completed       | expired        | 24시간 경과         |
| failed          | expired        | 24시간 경과         |

---

### 3.2 Job 전화번호 업데이트

**Endpoint:** `PATCH /api/v1/jobs/{job_id}/phone`

**Description:** 결제 성공 후 전화번호 등록 (필수)

**Request Body:**

```json
{
  "user_phone_number": "010-1234-5678"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Phone number updated successfully"
}
```

**Internal Process:**

1. Job 상태 확인 (`pending_phone`인지 검증)
2. 전화번호 형식 검증
3. Job의 `user_phone_number` 필드 업데이트
4. Job 상태 업데이트 (`pending_upload`)

---

## 4. 이미지 업로드 API

### 4.1 이미지 업로드

**Endpoint:** `POST /api/v1/jobs/{job_id}/upload`

**Description:** 사용자 이미지 업로드 및 AI 생성 시작

**Content-Type:** `multipart/form-data`

**Request Body:**

```
image: File (JPEG/PNG, max 10MB)
crop_data: JSON string
```

**crop_data JSON:**

```json
{
  "x": 100,
  "y": 50,
  "width": 600,
  "height": 800,
  "aspect_ratio": "3:4"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "file_id": "770e8400-e29b-41d4-a716-446655440002",
    "status": "processing",
    "message": "이미지 업로드 완료. AI 생성이 시작되었습니다."
  }
}
```

**Error Responses:**

- `400 Bad Request`: 잘못된 이미지 형식 또는 크기
- `404 Not Found`: Job을 찾을 수 없음
- `409 Conflict`: 이미 이미지가 업로드됨

**Validation Rules:**

- 파일 형식: JPEG, PNG, WEBP, HEIC 등 모든 이미지 형식 허용
- 최대 크기: 10MB (변환 전 기준)
- 최소 해상도: 512x512
- 크롭 비율: 3:4

**이미지 처리 파이프라인:**

```
1. 업로드
   - 모든 이미지 형식 허용 (JPEG, PNG, WEBP, HEIC 등)
   - multipart/form-data로 전송

2. 검증
   - 파일 크기 확인 (최대 10MB)
   - 이미지 해상도 확인 (최소 512x512)
   - 크롭 데이터 검증 (3:4 비율)

3. 자동 PNG 변환
   - Pillow 라이브러리 사용
   - 원본 품질 유지
   - 변환 후 파일명: {uuid}.png
   - 원본 형식은 메타데이터에 기록

4. S3 저장
   - 경로: {job_id}/original/{uuid}.png
   - 메타데이터: 파일 크기, MIME 타입 (image/png), 크롭 데이터, 원본 형식

5. DB 저장
   - job_files 테이블에 레코드 생성
   - file_type: 'original'
   - original_format: 원본 파일 형식 (예: 'HEIC', 'JPEG')
```

**Internal Process:**

1. Job 상태 확인 (`pending_upload`인지 검증)
2. 이미지 파일 검증 (형식, 크기, 해상도)
3. PNG로 자동 변환 (필요 시)
4. S3 업로드 (`{job_id}/original/{uuid}.png`)
5. job_files 테이블에 레코드 생성
6. Job 상태 업데이트 (`processing`)
7. Background Task로 AI 이미지 생성 시작

---

### 4.2 이미지 업로드 가이드 조회

**Endpoint:** `GET /api/v1/upload-guide`

**Description:** 이미지 업로드 가이드 정보 조회

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "good_examples": [
      {
        "title": "정면 얼굴",
        "description": "얼굴이 정면을 향하고 있어야 합니다",
        "image_url": "https://cdn.findmyrizz.com/guide/good_1.jpg"
      },
      {
        "title": "밝은 조명",
        "description": "자연광 또는 밝은 조명에서 촬영",
        "image_url": "https://cdn.findmyrizz.com/guide/good_2.jpg"
      },
      {
        "title": "선명한 화질",
        "description": "흔들림 없이 선명하게 촬영",
        "image_url": "https://cdn.findmyrizz.com/guide/good_3.jpg"
      }
    ],
    "bad_examples": [
      {
        "title": "측면 얼굴",
        "description": "얼굴이 옆을 향하면 안 됩니다",
        "image_url": "https://cdn.findmyrizz.com/guide/bad_1.jpg"
      },
      {
        "title": "어두운 조명",
        "description": "너무 어두우면 분석이 어렵습니다",
        "image_url": "https://cdn.findmyrizz.com/guide/bad_2.jpg"
      },
      {
        "title": "선글라스/마스크",
        "description": "얼굴을 가리는 액세서리 착용 금지",
        "image_url": "https://cdn.findmyrizz.com/guide/bad_3.jpg"
      }
    ],
    "requirements": {
      "format": ["JPEG", "PNG"],
      "max_size_mb": 10,
      "min_resolution": "512x512",
      "aspect_ratio": "3:4"
    }
  }
}
```

---

## 5. 결과 조회 API

### 5.1 전화번호 인증

**Endpoint:** `POST /api/v1/results/{job_id}/verify`

**Description:** 결과 조회를 위한 전화번호 인증

**Request Body:**

```json
{
  "phone_number": "010-1234-5678"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "verified": true,
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 3600
  }
}
```

**Error Responses:**

- `401 Unauthorized`: 전화번호 불일치
- `404 Not Found`: Job을 찾을 수 없음
- `410 Gone`: Job이 만료됨 (24시간 경과)
- `429 Too Many Requests`: 인증 시도 횟수 초과

---

### 5.2 결과 조회

**Endpoint:** `GET /api/v1/results/{job_id}`

**Description:** AI 생성 결과 및 리포트 조회

**Headers:**

```
Authorization: Bearer {access_token}
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "job_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "completed",
    "created_at": "2024-01-01T12:00:00Z",
    "expires_at": "2024-01-02T12:00:00Z",
    "images": [
      {
        "file_id": "880e8400-e29b-41d4-a716-446655440003",
        "url": "https://s3.amazonaws.com/findmyrizz/...",
        "type": "generated",
        "order": 1
      },
      {
        "file_id": "880e8400-e29b-41d4-a716-446655440004",
        "url": "https://s3.amazonaws.com/findmyrizz/...",
        "type": "generated",
        "order": 2
      },
      {
        "file_id": "880e8400-e29b-41d4-a716-446655440005",
        "url": "https://s3.amazonaws.com/findmyrizz/...",
        "type": "generated",
        "order": 3
      }
    ],
    "report": {
      "summary": "체계적인 관리를 통해 피부 톤 개선, 얼굴 윤곽 정의, 전반적인 인상 개선이 가능합니다.",
      "improvements": [
        {
          "area": "피부 관리",
          "description": "피부톤 균일화 및 광채 개선",
          "recommendations": [
            {
              "id": "SC003",
              "type": "self_care",
              "name": "기초 화장 - 썬크림",
              "category": "기초 피부 관리",
              "cost_range": "$10-40/병 (월 2-4병 소비)",
              "duration": "2-3시간 (재도포 필요)"
            },
            {
              "id": "P007",
              "type": "procedure",
              "name": "래이저 토닝",
              "category": "레이저 시술",
              "cost_range": "$200-400",
              "duration": "2-4주"
            }
          ]
        },
        {
          "area": "얼굴 윤곽",
          "description": "턱선 정의 및 V라인 개선",
          "recommendations": [
            {
              "id": "SC005",
              "type": "self_care",
              "name": "다이어트 (5kg 감량)",
              "category": "신체 자기관리 - 전신 개선",
              "cost_range": "$0 (홈기반) ~ $1,000+ (전문 프로그램)",
              "duration": "지속 가능 (유지식으로 안정)"
            }
          ]
        }
      ],
      "timeline": "8-12주 예상",
      "estimated_cost": "300,000 - 800,000원"
    }
  }
}
```

---

### 5.3 이미지 다운로드

**Endpoint:** `GET /api/v1/results/{job_id}/images/{image_id}/download`

**Description:** 개별 이미지 다운로드 (3장 각각 다운로드)

**Headers:**

```
Authorization: Bearer {access_token}
```

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| job_id | UUID | Job ID |
| image_id | UUID | 다운로드할 이미지의 file_id |

**Response (200 OK):**

- Content-Type: `image/jpeg`
- Content-Disposition: `attachment; filename="my_rizz_image_{order}.jpg"`

**사용 예시:**

```
GET /api/v1/results/{job_id}/images/{file_id_1}/download  # 첫 번째 이미지
GET /api/v1/results/{job_id}/images/{file_id_2}/download  # 두 번째 이미지
GET /api/v1/results/{job_id}/images/{file_id_3}/download  # 세 번째 이미지
```

---

## 6. 만족도 조사 API

### 6.1 만족도 조사 제출

**Endpoint:** `POST /api/v1/surveys`

**Description:** 서비스 만족도 조사 제출

**Request Body:**

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "survey_data": {
    "purpose": "자기관리 동기부여",
    "source": "인스타그램 광고",
    "satisfaction": 5,
    "would_recommend": true,
    "would_use_again": true
  },
  "feedback_text": "결과가 매우 만족스러웠습니다. 관리 목표가 생겼어요!",
  "review": "정말 좋은 서비스입니다!"
}
```

**Request Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| job_id | UUID | Yes | Job ID |
| survey_data | object | Yes | 설문 데이터 (구조화) |
| feedback_text | string | No | 자유 피드백 |
| review | string | No | 리뷰 (최대 250자) |

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "response_id": "990e8400-e29b-41d4-a716-446655440006",
    "submitted_at": "2024-01-01T13:00:00Z"
  }
}
```

---

## 7. 에러 코드

### 7.1 HTTP 상태 코드

| Code | Description           |
| ---- | --------------------- |
| 200  | 성공                  |
| 400  | 잘못된 요청           |
| 401  | 인증 실패             |
| 404  | 리소스를 찾을 수 없음 |
| 409  | 충돌 (중복 요청 등)   |
| 410  | 리소스 만료           |
| 429  | 요청 횟수 초과        |
| 500  | 서버 오류             |

### 7.2 커스텀 에러 코드

**에러 응답 형식:**

```json
{
  "success": false,
  "error": {
    "code": "E001",
    "message": "결제 금액이 일치하지 않습니다",
    "details": {
      "expected": 9900,
      "received": 10000
    }
  }
}
```

**에러 코드 목록:**
| Code | Description |
|------|-------------|
| E001 | 결제 금액 불일치 |
| E002 | 결제 검증 실패 |
| E003 | 잘못된 전화번호 형식 |
| E004 | 이미지 형식 오류 |
| E005 | 이미지 크기 초과 |
| E006 | 얼굴 감지 실패 |
| E007 | AI 생성 실패 |
| E008 | 전화번호 인증 실패 |
| E009 | Job 만료 |
| E010 | 인증 시도 횟수 초과 |
| E011 | S3 업로드 실패 |
| E012 | SMS 발송 실패 |

---

## 8. Webhook 엔드포인트

### 8.1 Replicate Webhook

**Endpoint:** `POST /api/v1/webhooks/replicate`

**Description:** Replicate에서 이미지 생성 완료 시 호출

**Request Body:**

```json
{
  "id": "replicate_prediction_id",
  "status": "succeeded",
  "output": [
    "https://replicate.delivery/pbxt/image1.jpg",
    "https://replicate.delivery/pbxt/image2.jpg",
    "https://replicate.delivery/pbxt/image3.jpg"
  ],
  "metrics": {
    "predict_time": 12.5
  }
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Webhook processed successfully"
}
```

---

## 9. Health Check

### 9.1 서버 상태 확인

**Endpoint:** `GET /health`

**Response (200 OK):**

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T12:00:00Z",
  "services": {
    "database": "connected",
    "s3": "accessible",
    "replicate": "reachable",
    "openai": "reachable",
    "coolsms": "reachable"
  }
}
```
