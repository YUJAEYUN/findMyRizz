# Phase 4: Image Upload - 이미지 업로드

## 📋 개요

결제 완료 후 사용자 이미지를 업로드하고 S3에 저장합니다.

---

## 🎯 목표

1. 이미지 검증 (크기, 형식, 내용)
2. S3 업로드
3. 크롭 데이터 저장
4. AI 생성 트리거

---

## 📁 문서 구조

```
04-image-upload/
├── README.md              # 이 파일
├── schemas.md             # Pydantic 스키마
├── services/
│   ├── validation.md     # 이미지 검증
│   ├── s3-upload.md      # S3 업로드
│   └── image-handler.md  # 이미지 처리
├── api.md                 # API 엔드포인트
└── tests.md              # 테스트
```

---

## 🔄 업로드 흐름

```
1. POST /api/v1/uploads/{job_id}
   - file: 이미지 파일
   - crop_data: {"x": 0, "y": 0, "width": 300, "height": 400}
   ↓
2. Job 상태 확인 (pending_upload)
   ↓
3. 이미지 검증
   - 크기: 10KB ~ 10MB
   - 형식: JPEG, PNG
   - 해상도: 512x512 ~ 4096x4096
   ↓
4. S3 업로드
   - 경로: jobs/{job_id}/original/{file_id}.jpg
   ↓
5. JobFile 레코드 생성
   - crop_data 저장
   ↓
6. Job 상태 업데이트 (processing)
   ↓
7. Celery Task: AI 생성 시작
```

---

## 📊 검증 규칙

### 파일 크기

- 최소: 10KB
- 최대: 10MB

### 파일 형식

- JPEG (.jpg, .jpeg)
- PNG (.png)

### 이미지 해상도

- 최소: 512x512
- 최대: 4096x4096

### 크롭 데이터

- 3:4 비율
- JSON 형식

---

## 🔄 실행 순서

1. **스키마 정의** (`schemas.md`)
2. **이미지 검증** (`services/validation.md`)
3. **S3 업로드** (`services/s3-upload.md`)
4. **이미지 처리** (`services/image-handler.md`)
5. **API 구현** (`api.md`)
6. **테스트** (`tests.md`)

---

## ✅ 완료 기준

- [ ] 이미지 업로드 API 동작
- [ ] 모든 검증 통과
- [ ] S3 업로드 성공
- [ ] JobFile 레코드 생성
- [ ] AI 생성 트리거
- [ ] 모든 테스트 통과

---

## 🚀 다음 단계

Image Upload 완료 → **Phase 5: AI Generation**
