# 프로젝트 초기화

## 📋 목표

FastAPI 프로젝트를 처음부터 생성합니다.

---

## 🔧 Step 1: Python 가상환경 생성

```bash
# 프로젝트 폴더 생성
mkdir fmr-api
cd fmr-api

# Python 3.11+ 가상환경 생성
python3.11 -m venv venv

# 가상환경 활성화 (macOS/Linux)
source venv/bin/activate

# 가상환경 활성화 (Windows)
venv\Scripts\activate
```

---

## 🔧 Step 2: FastAPI 설치

```bash
# FastAPI 및 Uvicorn 설치
pip install fastapi==0.104.1
pip install uvicorn[standard]==0.24.0
```

---

## 🔧 Step 3: 기본 main.py 생성

### 파일: `main.py`

```python
"""
Find My Rizz API
FastAPI 메인 애플리케이션
"""
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import logging

# 로깅 설정
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# FastAPI 앱 생성
app = FastAPI(
    title="Find My Rizz API",
    description="AI-powered attractiveness analysis service",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc"
)

# CORS 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 프로덕션에서는 특정 도메인만 허용
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.get("/")
async def root():
    """
    헬스 체크 엔드포인트
    
    Returns:
        dict: {"status": "ok", "message": "Find My Rizz API is running"}
    """
    return {
        "status": "ok",
        "message": "Find My Rizz API is running"
    }


@app.get("/health")
async def health_check():
    """
    상세 헬스 체크
    
    Returns:
        dict: {
            "status": "healthy",
            "version": "1.0.0",
            "database": "connected",  # 나중에 DB 연결 후 추가
            "redis": "connected"      # 나중에 Redis 연결 후 추가
        }
    """
    return {
        "status": "healthy",
        "version": "1.0.0"
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,  # 개발 모드에서만 True
        log_level="info"
    )
```

---

## 🔧 Step 4: 서버 실행 테스트

```bash
# 서버 실행
uvicorn main:app --reload

# 또는
python main.py
```

**예상 출력:**
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] using StatReload
INFO:     Started server process [12346]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

---

## ✅ 테스트

### 1. 브라우저 테스트
```
http://localhost:8000
→ {"status": "ok", "message": "Find My Rizz API is running"}

http://localhost:8000/docs
→ Swagger UI 표시

http://localhost:8000/health
→ {"status": "healthy", "version": "1.0.0"}
```

### 2. curl 테스트
```bash
curl http://localhost:8000
curl http://localhost:8000/health
```

---

## 📝 체크리스트

- [ ] Python 3.11+ 설치 확인
- [ ] 가상환경 생성 및 활성화
- [ ] FastAPI, Uvicorn 설치
- [ ] main.py 생성
- [ ] 서버 실행 성공
- [ ] http://localhost:8000 접속 확인
- [ ] http://localhost:8000/docs 접속 확인
- [ ] http://localhost:8000/health 접속 확인

---

## 🚀 다음 단계

프로젝트 초기화 완료 → **02-environment.md** (환경변수 설정)

