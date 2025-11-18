# Phase 3: Payment - 결제 시스템

## 📋 개요

PortOne을 사용한 결제 시스템을 구현합니다.

---

## 🎯 목표

1. 결제 초기화 API
2. PortOne Webhook 처리
3. 결제 검증 로직
4. 에러 처리

---

## 📁 문서 구조

```
03-payment/
├── README.md              # 이 파일
├── schemas.md             # Pydantic 스키마
├── services/
│   ├── initialize.md     # 결제 초기화
│   ├── webhook.md        # Webhook 처리
│   └── verification.md   # 결제 검증
├── phone-registration.md  # 전화번호 등록 API
├── api.md                 # API 엔드포인트
└── tests.md              # 테스트
```

---

## 🔄 결제 흐름

```
1. POST /api/v1/payments/initialize
   → Job 생성 (status=pending_payment)
   → merchant_uid 생성
   → 응답: {job_id, merchant_uid, amount}
   ↓
2. 프론트엔드: PortOne SDK로 결제 진행
   ↓
3. PortOne → POST /api/v1/webhooks/portone
   → 결제 검증
   → Job 상태 업데이트 (pending_phone)
   → Payment 레코드 업데이트
   ↓
4. 프론트엔드: 전화번호 입력 페이지 표시
   ↓
5. PATCH /api/v1/jobs/{job_id}/phone
   → 전화번호 저장
   → Job 상태 업데이트 (pending_upload)
   ↓
6. 프론트엔드: 이미지 업로드 페이지로 이동
```

---

## 💰 결제 정보

- **금액:** 9,900원 (고정)
- **결제 수단:** 카드 (CARD)
- **PG사:** PortOne V2 (웰컴페이먼츠)
- **참고 문서:** `docs/technical/portone_v2_integration.md`

---

## 🔄 실행 순서

1. **스키마 정의** (`schemas.md`)
2. **결제 초기화** (`services/initialize.md`)
3. **Webhook 처리** (`services/webhook.md`)
4. **결제 검증** (`services/verification.md`)
5. **전화번호 등록 API** (`phone-registration.md`)
6. **API 구현** (`api.md`)
7. **테스트** (`tests.md`)

---

## ✅ 완료 기준

- [ ] 결제 초기화 API 동작
- [ ] Webhook 수신 및 처리
- [ ] 결제 검증 성공
- [ ] 전화번호 등록 API 동작
- [ ] Job 상태 자동 업데이트 (pending_payment → pending_phone → pending_upload)
- [ ] 모든 테스트 통과

---

## 🚀 다음 단계

Payment 완료 → **Phase 4: Image Upload**
