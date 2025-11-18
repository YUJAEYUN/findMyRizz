# Report API

## 📋 목표

리포트 조회 API를 구현합니다.

---

## 🔧 구현

### 파일: `app/api/v1/reports.py`

```python
"""
Report API
"""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session

from app.core.database import get_db
from app.models.job import Job
from app.models.report import Report
from app.schemas.report import ReportResponse

router = APIRouter(prefix="/reports", tags=["reports"])


@router.get("/{job_id}", response_model=ReportResponse)
async def get_report(
    job_id: str,
    db: Session = Depends(get_db)
):
    """
    리포트 조회
    
    Args:
        job_id: Job ID
    
    Returns:
        리포트 데이터
    
    Raises:
        404: Job 없음
        404: Report 없음
    """
    # Step 1: Job 조회
    job = db.query(Job).filter(Job.id == job_id).first()
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")
    
    # Step 2: Report 조회
    report = db.query(Report).filter(Report.job_id == job_id).first()
    if not report:
        raise HTTPException(status_code=404, detail="Report not found")
    
    # Step 3: 응답 생성
    return ReportResponse.from_orm(report)
```

---

## 🔧 스키마

### 파일: `app/schemas/report.py`

```python
"""
Report 스키마
"""
from pydantic import BaseModel
from typing import List, Dict, Any
from datetime import datetime


class KnowledgeItemSchema(BaseModel):
    """Knowledge Item 스키마"""
    id: str
    title: str
    description: str
    procedure_type: str | None
    estimated_cost: str | None
    
    class Config:
        from_attributes = True


class KnowledgeCategorySchema(BaseModel):
    """Knowledge Category 스키마"""
    id: str
    name: str
    description: str | None
    items: List[KnowledgeItemSchema]
    
    class Config:
        from_attributes = True


class ReportResponse(BaseModel):
    """Report 응답 스키마"""
    id: str
    job_id: str
    analysis_result: Dict[str, Any]
    knowledge_items: List[KnowledgeItemSchema]
    created_at: datetime
    
    class Config:
        from_attributes = True
    
    @classmethod
    def from_orm(cls, report: Report):
        """
        ORM 객체에서 변환
        """
        import json
        
        # analysis_result 파싱
        analysis_result = json.loads(report.analysis_result)
        
        # knowledge_items 변환
        knowledge_items = [
            KnowledgeItemSchema.from_orm(rk.knowledge_item)
            for rk in report.knowledge_items
        ]
        
        return cls(
            id=report.id,
            job_id=report.job_id,
            analysis_result=analysis_result,
            knowledge_items=knowledge_items,
            created_at=report.created_at
        )
```

---

## 🔍 응답 예시

```json
{
  "id": "report-uuid",
  "job_id": "job-uuid",
  "analysis_result": {
    "skin_tone": "uneven",
    "facial_features": ["dark_circles", "acne_scars"],
    "improvement_areas": ["skin", "eyes"],
    "recommendations": ["vitamin_c", "eye_cream"]
  },
  "knowledge_items": [
    {
      "id": "item-uuid-1",
      "title": "비타민 C 세럼",
      "description": "피부 톤을 밝게 하고...",
      "procedure_type": "cosmetic",
      "estimated_cost": "3만원~10만원"
    },
    {
      "id": "item-uuid-2",
      "title": "눈 주변 관리",
      "description": "아이크림을 사용하여...",
      "procedure_type": "cosmetic",
      "estimated_cost": "3만원~10만원"
    }
  ],
  "created_at": "2024-01-15T14:30:22.123456"
}
```

---

## ✅ 테스트

### curl 명령어

```bash
# 리포트 조회
curl -X GET "http://localhost:8000/api/v1/reports/{job_id}"
```

### pytest

```python
def test_get_report(client, db):
    # Job 생성
    job = Job(id=str(uuid.uuid4()), ...)
    db.add(job)
    
    # Report 생성
    report = Report(
        id=str(uuid.uuid4()),
        job_id=job.id,
        analysis_result='{"skin_tone": "uneven"}'
    )
    db.add(report)
    db.commit()
    
    # API 호출
    response = client.get(f"/api/v1/reports/{job.id}")
    
    # 검증
    assert response.status_code == 200
    data = response.json()
    assert data["job_id"] == job.id
    assert "analysis_result" in data
    assert "knowledge_items" in data
```

---

## 📝 체크리스트

- [ ] app/api/v1/reports.py 생성
- [ ] app/schemas/report.py 생성
- [ ] get_report() 엔드포인트
- [ ] ReportResponse 스키마
- [ ] 테스트 작성

---

## 🚀 다음 단계

Report API 완료 → **SMS retry**

