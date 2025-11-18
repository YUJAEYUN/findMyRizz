# Find My Rizz - 문서 검토 및 수정 사항

## 문서 정보

- **작성일**: 2024-01-01
- **목적**: 전체 문서 검토 후 발견된 모순 및 개선 사항 정리

---

## 1. 발견된 논리적 모순 및 해결 방안

### 1.1 전화번호 입력 시점 결정

**초기 검토:**

- 문서 간 불일치 발견 (결제 전 vs 결제 후)

**분석:**

- **결제 후 전화번호 입력이 더 합리적**
- 이유:
  1. **개인정보 최소 수집**: 결제 실패자의 전화번호 불필요
  2. **결제 장벽 최소화**: 9,900원 소액 결제에서 정보 요구 시 이탈 증가
  3. **충동구매 유도**: 바로 결제 가능하여 전환율 향상
  4. **결제 실패 시 리다이렉트**: 어차피 랜딩페이지로 돌아가므로 재입력 방지 효과 없음

**해결 방안:**

- **최종 플로우**: 결제 → 전화번호 입력 → 이미지 업로드
- 모든 문서 통일

**수정된 플로우:**

```
1. 랜딩 페이지 방문
2. "지금 바로 나의 리즈 알아보기" 클릭
3. 결제 진행 (9,900원)
   - 실패 시 → 랜딩 페이지 복귀
4. 결제 성공 → 전화번호 입력 페이지
5. 사진 가이드 확인
6. 이미지 업로드 및 크롭
7. 신청 완료
```

---

### 1.2 리포트 형식 불일치

**문제:**

- `project_plan.md`: "리포트 pdf와 3장의 사진들이 다운로드"
- `technical_architecture.md`: "리포트는 웹 URL로 발송"

**분석:**

- PDF 생성은 추가 복잡도 증가
- 웹페이지가 더 간단하고 유지보수 용이
- 사용자는 웹에서 조회 후 필요 시 스크린샷 가능

**해결 방안:**

- **최종 결정**: 웹페이지로 통일
- 다운로드 기능: 이미지 3장만 ZIP으로 제공
- PDF 생성 기능 제거

**수정된 다운로드 기능:**

```
"내 사진 모두 다운로드" 버튼
→ ZIP 파일 생성 (이미지 3장만)
→ 파일명: my_rizz_images_{YYYYMMDD}.zip
```

---

### 1.3 이미지 생성 방식 명확화

**문제:**

- `ai_image_generation_prompt.md`: 1장 생성용 프롬프트
- PRD: 3장 생성 필요
- 3장을 어떻게 생성할지 불명확

**해결 방안:**

- **방식**: 동일 프롬프트로 3회 API 호출
- **차별화**: 각 호출마다 다른 seed 값 사용
- **이유**: 다양한 결과 제공, 사용자 선택권 증가

**구현 방법:**

```python
seeds = [random.randint(0, 1000000) for _ in range(3)]

for i, seed in enumerate(seeds):
    prediction = replicate.predictions.create(
        version="ai_image_generation_model_version",
        input={
            "image": input_image_url,
            "prompt": AI_IMAGE_GENERATION_PROMPT,
            "seed": seed,
            "num_inference_steps": 50,
            "guidance_scale": 7.5
        }
    )
    # 결과 저장 (display_order: i+1)
```

---

### 1.4 사진 가이드 구체화 필요

**문제:**

- `project_plan.md`: "사진 가이드 (O/X 예시)" 언급만
- 구체적인 내용 없음

**해결 방안:**

- API 명세서에 이미 정의됨
- 실제 구현 시 예시 이미지 필요

**좋은 예시:**

1. **정면 얼굴**: 얼굴이 정면을 향하고 카메라를 직접 응시
2. **밝은 조명**: 자연광 또는 밝은 실내 조명
3. **선명한 화질**: 흔들림 없이 선명하게 촬영

**나쁜 예시:**

1. **측면 얼굴**: 얼굴이 옆을 향하거나 각도가 있음
2. **어두운 조명**: 역광, 그림자, 어두운 환경
3. **가림**: 선글라스, 마스크, 모자 등으로 얼굴 가림

---

## 2. 데이터베이스 스키마 개선 사항

### 2.1 추가된 테이블 및 필드

**jobs 테이블:**

- `merchant_uid` 추가: 결제 고유 ID
- `updated_at` 추가: 상태 변경 추적
- `TIMESTAMPOZ` → `TIMESTAMPTZ` 수정

**payments 테이블:**

- `imp_uid` 추가: PortOne 거래 ID
- `created_at` 추가: 결제 시도 시간

**job_files 테이블:**

- `s3_bucket` 추가: 버킷 이름
- `file_size_bytes` 추가: 파일 크기
- `mime_type` 추가: MIME 타입
- `crop_data` 추가: 크롭 정보 (JSONB)
- `display_order` 추가: 표시 순서

**신규 테이블:**

- `sms_logs`: SMS 발송 이력
- `payment_failures`: 결제 실패 로그
- `phone_verification_attempts`: 전화번호 인증 시도 로그

**Lookup 테이블 초기 데이터:**

- `lu_job_statuses`: 6개 상태 정의
- `lu_payment_statuses`: 5개 상태 정의
- `lu_payment_methods`: 6개 결제 수단 정의
- `lu_file_types`: 3개 파일 타입 정의

---

## 3. API 명세서 개선 사항

### 3.1 추가된 엔드포인트

**결제:**

- `POST /api/v1/payments/initialize`: 결제 초기화
- `POST /api/v1/payments/webhook`: PortOne Webhook
- `POST /api/v1/payments/failure`: 결제 실패 처리

**Job 관리:**

- `GET /api/v1/jobs/{job_id}/status`: Job 상태 조회
- `PATCH /api/v1/jobs/{job_id}/phone`: 전화번호 업데이트

**이미지:**

- `POST /api/v1/jobs/{job_id}/upload`: 이미지 업로드
- `GET /api/v1/upload-guide`: 업로드 가이드 조회

**결과:**

- `POST /api/v1/results/{job_id}/verify`: 전화번호 인증
- `GET /api/v1/results/{job_id}`: 결과 조회
- `GET /api/v1/results/{job_id}/download`: 이미지 다운로드

**만족도 조사:**

- `POST /api/v1/surveys`: 설문 제출

**Webhook:**

- `POST /api/v1/webhooks/replicate`: Replicate Webhook

**Health Check:**

- `GET /health`: 서버 상태 확인

---

## 4. 보안 정책 개선 사항

### 4.1 24시간 자동 파기 정책 구체화

**파기 대상:**

- 사용자 업로드 원본 이미지
- AI 생성 이미지 3장
- 리포트 데이터
- 전화번호
- Job 레코드 (CASCADE)

**보존 대상:**

- 결제 정보 (5년, 법적 의무)
- 익명화된 통계 데이터
- 만족도 조사 응답

**파기 프로세스:**

- Scheduled Task (매 시간 실행)
- S3 파일 삭제 → DB 레코드 삭제
- 파기 로그 저장

### 4.2 전화번호 인증 보안 강화

**실패 제한:**

- IP 기반: 3회 실패 시 1시간 차단
- Job 기반: 10회 실패 시 영구 차단

**로그 저장:**

- `phone_verification_attempts` 테이블
- IP 주소, 성공 여부, 시간 기록

---

## 5. 에러 핸들링 개선 사항

### 5.1 체계적인 에러 코드 정의

**형식:** `E-{CATEGORY}-{NUMBER}`

**카테고리:**

- PAY: 결제 관련
- IMG: 이미지 관련
- AI: AI 생성 관련
- RPT: 리포트 관련
- SMS: SMS 관련
- AUTH: 인증 관련
- FILE: 파일 관련
- SYS: 시스템 관련

**예시:**

- `E-PAY-001`: 결제 초기화 실패
- `E-IMG-003`: 얼굴 감지 실패
- `E-AI-002`: 생성 시간 초과

### 5.2 재시도 정책 표준화

**지수 백오프:**

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
```

**고정 간격:**

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(300)  # 5분
)
```

---

## 6. 기술 아키텍처 개선 사항

### 6.1 Background Task 처리

**MVP:**

- FastAPI BackgroundTasks 사용
- 간단하고 빠른 구현

**확장:**

- Celery + Redis 도입
- 대규모 트래픽 대응

### 6.2 데이터베이스 최적화

**인덱스 추가:**

- `jobs(status)`
- `jobs(expires_at)`
- `jobs(user_phone_number)` - HASH 인덱스
- `payments(job_id)`
- `payments(imp_uid)`
- `job_files(job_id)`
- `job_files(file_type)`

---

## 7. 남은 작업 및 권장 사항

### 7.1 즉시 수정 필요

1. **project_plan.md 수정**

   - 전화번호 입력 시점: 결제 후로 변경
   - 리포트 다운로드: PDF 제거, 이미지만 ZIP

2. **ai_image_generation_prompt.md 보완**
   - 3장 생성 방식 명시
   - seed 값 사용 설명 추가

### 7.2 구현 시 고려사항

1. **사진 가이드 이미지**

   - 실제 예시 이미지 준비 필요
   - S3 또는 CDN에 업로드

2. **환불 정책 페이지**

   - 별도 페이지 또는 FAQ 작성
   - 법적 검토 필요

3. **개인정보 처리 방침**
   - 법무팀 검토 필요
   - 사용자 동의 UI 구현

### 7.3 향후 확장 고려

1. **스타일 선택 기능**

   - 프롬프트 템플릿 확장
   - 사용자 선택 UI

2. **맞춤형 리포트**

   - LangChain/LangGraph 도입
   - GPT-4o 프롬프트 고도화

3. **다국어 지원**
   - i18n 라이브러리 도입
   - 영어, 일본어 등

---

## 8. 최종 체크리스트

### 8.1 문서 완성도

- [x] 기술 아키텍처 문서
- [x] API 명세서
- [x] 데이터베이스 스키마
- [x] 상세 PRD
- [x] 보안 및 프라이버시 정책
- [x] 에러 핸들링 문서
- [x] 검토 및 수정 사항 문서

### 8.2 논리적 일관성

- [x] 전화번호 입력 시점 통일
- [x] 리포트 형식 통일 (웹페이지)
- [x] 이미지 생성 방식 명확화
- [x] 에러 코드 체계화
- [x] 데이터베이스 스키마 완성

### 8.3 구현 준비도

- [x] 모든 API 엔드포인트 정의
- [x] 데이터베이스 스키마 완성
- [x] 에러 처리 방안 수립
- [x] 보안 정책 수립
- [ ] 환경변수 목록 작성 (technical_architecture.md에 포함)
- [ ] 배포 가이드 작성 (향후 필요)

---

## 9. 권장 개발 순서

### Phase 1: 기본 인프라 (1주)

1. FastAPI 프로젝트 초기화
2. PostgreSQL 설정 및 마이그레이션
3. AWS S3 설정
4. 환경변수 관리

### Phase 2: 결제 시스템 (1주)

1. PortOne 연동
2. 결제 API 구현
3. Webhook 처리
4. 테스트

### Phase 3: 이미지 처리 (1주)

1. 이미지 업로드 API
2. S3 업로드
3. 크롭 기능 (프론트엔드)
4. 검증 로직

### Phase 4: AI 생성 (1주)

1. Replicate API 연동
2. Background Task 구현
3. 이미지 생성 로직
4. 재시도 로직

### Phase 5: 리포트 생성 (1주)

1. OpenAI API 연동
2. 리포트 생성 로직
3. 관리법 매칭
4. 프론트엔드 렌더링

### Phase 6: SMS 및 결과 조회 (1주)

1. CoolSMS API 연동
2. SMS 발송 로직
3. 전화번호 인증
4. 결과 페이지

### Phase 7: 보안 및 최적화 (1주)

1. 24시간 자동 파기 구현
2. Rate Limiting
3. 로깅 및 모니터링
4. 성능 최적화

### Phase 8: 테스트 및 배포 (1주)

1. 통합 테스트
2. 부하 테스트
3. 보안 점검
4. 프로덕션 배포

**총 예상 기간: 8주 (2개월)**

---

## 10. 결론

모든 문서가 논리적으로 일관성 있게 작성되었으며, 발견된 모순은 이 문서에서 해결 방안을 제시했습니다.

**주요 수정 사항:**

1. 전화번호 입력 시점: 결제 후
2. 리포트 형식: 웹페이지 (PDF 제거)
3. 이미지 생성: 3회 API 호출 (다른 seed)

**구현 준비 완료:**

- 모든 기술 스펙 정의 완료
- API 명세서 완성
- 데이터베이스 스키마 완성
- 보안 정책 수립
- 에러 처리 방안 수립

**다음 단계:**

- 개발 착수
- 프론트엔드 디자인 및 개발
- 백엔드 API 구현
- 테스트 및 배포
