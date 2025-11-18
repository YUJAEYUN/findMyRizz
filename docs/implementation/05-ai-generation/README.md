# Phase 5: AI Generation - AI 이미지 생성

## 📋 개요

Replicate SeeDream4 API를 사용하여 AI 이미지 3장을 생성합니다.

---

## 🎯 목표

1. Replicate API 연동
2. 3장의 이미지 생성 (다른 seed)
3. Webhook으로 결과 수신
4. S3에 저장

---

## 📁 문서 구조

```
05-ai-generation/
├── README.md              # 이 파일
├── services/
│   ├── replicate-client.md   # Replicate API
│   ├── image-generator.md    # 생성 로직
│   └── webhook-handler.md    # Webhook 처리
├── tasks.md               # Background Task
└── tests.md              # 테스트
```

---

## 🔄 생성 흐름

```
1. 이미지 업로드 완료
   ↓
2. Celery Task 시작
   ↓
3. Replicate API 호출 (3회)
   - seed: random()
   - seed: random()
   - seed: random()
   ↓
4. prediction_id 저장
   ↓
5. Replicate → Webhook 호출
   ↓
6. 생성된 이미지 다운로드
   ↓
7. S3 업로드
   ↓
8. JobFile 생성 (type=generated)
   ↓
9. 3장 완료 시 → 리포트 생성 시작
```

---

## 🎨 SeeDream4 파라미터

```python
{
    "image": "https://s3.../original.jpg",
    "seed": random.randint(1, 1000000),
    "num_inference_steps": 50,
    "guidance_scale": 7.5
}
```

---

## 🔄 실행 순서

1. **Replicate 클라이언트** (`services/replicate-client.md`)
2. **이미지 생성 로직** (`services/image-generator.md`)
3. **Webhook 처리** (`services/webhook-handler.md`)
4. **Celery Task** (`tasks.md`)
5. **테스트** (`tests.md`)

---

## ✅ 완료 기준

- [ ] Replicate API 연동
- [ ] 3장 생성 성공
- [ ] Webhook 수신
- [ ] S3 저장 완료
- [ ] JobFile 생성
- [ ] 리포트 생성 트리거

---

## 🚀 다음 단계

AI Generation 완료 → **Phase 6: Report Generation**
