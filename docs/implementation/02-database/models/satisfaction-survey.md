# SatisfactionSurvey 모델

## 📋 목표

사용자 만족도 조사를 저장하는 모델을 정의합니다.

---

## 🔧 구현

### 파일: `app/models/satisfaction_survey.py`

```python
"""
SatisfactionSurvey 모델
"""
from sqlalchemy import Column, String, Integer, Text

from app.core.database import Base
from app.models.base import TimestampMixin


class SatisfactionSurvey(Base, TimestampMixin):
    """
    SatisfactionSurvey 테이블
    
    독립 테이블 (Job과 관계 없음)
    익명 만족도 조사
    """
    __tablename__ = "satisfaction_surveys"
    
    # ===== 기본 필드 =====
    id = Column(String(36), primary_key=True)
    
    rating = Column(
        Integer,
        nullable=False,
        comment="만족도 점수 (1~5)"
    )
    
    feedback = Column(
        Text,
        nullable=True,
        comment="피드백 내용"
    )
    
    user_agent = Column(
        String(500),
        nullable=True,
        comment="User Agent"
    )
    
    ip_address = Column(
        String(45),
        nullable=True,
        comment="IP 주소"
    )
    
    def __repr__(self):
        return f"<SatisfactionSurvey(id={self.id}, rating={self.rating})>"
```

---

## 📊 테이블 스키마

```sql
CREATE TABLE satisfaction_surveys (
    id VARCHAR(36) PRIMARY KEY,
    rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    feedback TEXT,
    user_agent VARCHAR(500),
    ip_address VARCHAR(45),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_satisfaction_surveys_rating ON satisfaction_surveys(rating);
CREATE INDEX idx_satisfaction_surveys_created ON satisfaction_surveys(created_at);
```

---

## 🔍 사용 예시

### 만족도 조사 저장

```python
from app.models.satisfaction_survey import SatisfactionSurvey

survey = SatisfactionSurvey(
    id=str(uuid.uuid4()),
    rating=5,
    feedback="정말 유용했습니다!",
    user_agent=request.headers.get("User-Agent"),
    ip_address=request.client.host
)
db.add(survey)
db.commit()
```

### 평균 만족도 조회

```python
from sqlalchemy import func

avg_rating = db.query(func.avg(SatisfactionSurvey.rating)).scalar()
print(f"Average rating: {avg_rating:.2f}")
```

### 최근 피드백 조회

```python
recent_surveys = db.query(SatisfactionSurvey).order_by(
    SatisfactionSurvey.created_at.desc()
).limit(10).all()

for survey in recent_surveys:
    print(f"Rating: {survey.rating}, Feedback: {survey.feedback}")
```

---

## 📝 체크리스트

- [ ] app/models/satisfaction_survey.py 생성
- [ ] SatisfactionSurvey 클래스 정의
- [ ] rating CHECK 제약조건
- [ ] 테스트 작성

---

## 🚀 다음 단계

SatisfactionSurvey 모델 완료 → **migrations.md**

