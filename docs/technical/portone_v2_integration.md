# PortOne V2 웰컴페이먼츠 연동 스펙

## 1. 개요

- **PG사**: 웰컴페이먼츠 (Welcome Payments)
- **PortOne 버전**: V2
- **결제 금액**: 9,900원 (고정)
- **지원 결제수단**: 카드 (CARD)

---

## 2. 환경 설정

### 2.1 필수 환경 변수

```bash
# PortOne V2
PORTONE_STORE_ID=store-00000000000000000000
PORTONE_CHANNEL_KEY=channel-key-00000000000000
PORTONE_API_SECRET=your_v2_api_secret

# 웰컴페이먼츠 설정
PORTONE_PG_PROVIDER=welcome
```

### 2.2 SDK 설치

```bash
# Frontend
npm install @portone/browser-sdk

# Backend (Python)
pip install httpx
```

---

## 3. Frontend 결제 요청

### 3.1 결제 초기화 API 호출

```javascript
// Step 1: Backend에서 Job 생성
const initResponse = await fetch("/api/v1/payments/initialize", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ amount: 9900 }),
});

const { job_id, payment_id } = await initResponse.json();
```

### 3.2 PortOne SDK 결제 요청

```javascript
import * as PortOne from "@portone/browser-sdk/v2";

const response = await PortOne.requestPayment({
  storeId: process.env.PORTONE_STORE_ID,
  channelKey: process.env.PORTONE_CHANNEL_KEY,
  paymentId: payment_id,
  orderName: "Find My Rizz - 리즈 사진 생성",
  totalAmount: 9900,
  currency: "CURRENCY_KRW",
  payMethod: "CARD",
  customer: {
    fullName: "홍길동", // 필수 (PC)
    phoneNumber: "010-1234-5678", // 필수 (PC)
    email: "test@example.com", // 필수 (PC)
  },
  redirectUrl: `${window.location.origin}/payment/callback`, // 모바일용
});

// 결제 성공 처리
if (response.code === undefined) {
  await fetch("/api/v1/payments/complete", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ payment_id: response.paymentId }),
  });

  // 전화번호 입력 페이지로 이동
  window.location.href = `/phone-input?job_id=${job_id}`;
} else {
  alert(`결제 실패: ${response.message}`);
}
```

---

## 4. Backend 결제 검증

### 4.1 결제 완료 검증 엔드포인트

```python
# app/api/payments.py

@router.post("/complete")
async def complete_payment(
    payment_id: str,
    db: AsyncSession = Depends(get_db),
    settings: Settings = Depends(get_settings)
):
    # 1. PortOne API로 결제 정보 조회
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.portone.io/payments/{payment_id}",
            headers={"Authorization": f"PortOne {settings.PORTONE_API_SECRET}"}
        )

        if response.status_code != 200:
            raise HTTPException(status_code=400, detail="결제 조회 실패")

        payment_data = response.json()

    # 2. 결제 금액 검증
    if payment_data["amount"]["total"] != 9900:
        raise HTTPException(status_code=400, detail="결제 금액 불일치")

    # 3. 결제 상태 확인
    if payment_data["status"] != "PAID":
        raise HTTPException(status_code=400, detail="결제 미완료")

    # 4. Job 및 Payment 업데이트
    # ... (구현 생략)

    return {"success": True, "job_id": job.id}
```

---

## 5. Webhook 처리 (선택사항)

### 5.1 Webhook 엔드포인트

```python
@router.post("/webhook")
async def payment_webhook(
    request: Request,
    db: AsyncSession = Depends(get_db)
):
    # Webhook 데이터 파싱
    data = await request.json()

    # 서명 검증 (PortOne V2)
    # ... 구현 필요

    # 결제 상태 업데이트
    # ... 구현

    return {"success": True}
```

---

## 6. 에러 처리

### 6.1 주요 에러 코드

| 에러 코드           | 설명           | 처리 방법                 |
| ------------------- | -------------- | ------------------------- |
| `PAYMENT_NOT_FOUND` | 결제 정보 없음 | 재시도 안내               |
| `AMOUNT_MISMATCH`   | 금액 불일치    | 관리자 알림               |
| `PAYMENT_FAILED`    | 결제 실패      | 사용자에게 실패 사유 표시 |

---

## 7. 테스트

### 7.1 테스트 카드 번호

- **카드번호**: 1111-1111-1111-1111
- **유효기간**: 임의 (미래 날짜)
- **CVC**: 123
- **비밀번호**: 00

---

## 8. 참고 문서

- [PortOne V2 웰컴페이먼츠 가이드](https://developers.portone.io/opi/ko/integration/pg/v2/welcome)
- [PortOne V2 결제 연동](https://developers.portone.io/opi/ko/integration/start/v2/checkout)
- [PortOne V2 API Reference](https://developers.portone.io/api/rest-v2)
