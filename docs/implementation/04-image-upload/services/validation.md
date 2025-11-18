# 이미지 검증 서비스

## 📋 목표

업로드된 이미지의 유효성을 검증합니다.

---

## 🔧 구현

### 파일: `app/utils/image.py`

```python
"""
이미지 검증 유틸리티
"""
from PIL import Image
import io
from typing import Tuple, Dict
import logging

logger = logging.getLogger(__name__)


class ImageValidator:
    """이미지 검증 클래스"""
    
    # 상수
    MAX_SIZE = 10 * 1024 * 1024  # 10MB
    MIN_SIZE = 10 * 1024          # 10KB
    ALLOWED_CONTENT_TYPES = ['image/jpeg', 'image/png']
    ALLOWED_EXTENSIONS = ['.jpg', '.jpeg', '.png']
    MIN_DIMENSION = 512
    MAX_DIMENSION = 4096
    
    @classmethod
    def validate(
        cls,
        file_data: bytes,
        content_type: str,
        filename: str
    ) -> Tuple[bool, str, Dict]:
        """
        이미지 전체 검증
        
        Args:
            file_data: 이미지 바이너리
            content_type: Content-Type
            filename: 파일명
            
        Returns:
            (is_valid, error_message, metadata)
        """
        # Step 1: 파일 크기 검증
        file_size = len(file_data)
        logger.info(f"[ImageValidator] Validating: size={file_size}")
        
        if file_size > cls.MAX_SIZE:
            return False, f"File size exceeds {cls.MAX_SIZE // (1024*1024)}MB", {}
        
        if file_size < cls.MIN_SIZE:
            return False, f"File size too small (min {cls.MIN_SIZE // 1024}KB)", {}
        
        # Step 2: Content-Type 검증
        if content_type not in cls.ALLOWED_CONTENT_TYPES:
            return False, "Only JPEG and PNG allowed", {}
        
        # Step 3: 확장자 검증
        file_ext = filename.lower().split('.')[-1] if '.' in filename else ''
        if f".{file_ext}" not in cls.ALLOWED_EXTENSIONS:
            return False, "Invalid file extension", {}
        
        # Step 4: 이미지 유효성 검증
        try:
            image = Image.open(io.BytesIO(file_data))
            width, height = image.size
            image_format = image.format
            logger.info(f"[ImageValidator] Image: {width}x{height}, {image_format}")
        except Exception as e:
            logger.error(f"[ImageValidator] Invalid image: {e}")
            return False, "Invalid or corrupted image", {}
        
        # Step 5: 해상도 검증
        if width < cls.MIN_DIMENSION or height < cls.MIN_DIMENSION:
            return False, f"Min resolution: {cls.MIN_DIMENSION}x{cls.MIN_DIMENSION}", {}
        
        if width > cls.MAX_DIMENSION or height > cls.MAX_DIMENSION:
            return False, f"Max resolution: {cls.MAX_DIMENSION}x{cls.MAX_DIMENSION}", {}
        
        # Step 6: 메타데이터 반환
        metadata = {
            "width": width,
            "height": height,
            "format": image_format,
            "size_bytes": file_size
        }
        
        logger.info(f"[ImageValidator] Validation passed: {metadata}")
        return True, "", metadata
```

---

## 🔍 검증 규칙

### 1. 파일 크기
```python
MIN_SIZE = 10KB
MAX_SIZE = 10MB
```

### 2. 파일 형식
```python
ALLOWED_CONTENT_TYPES = ['image/jpeg', 'image/png']
ALLOWED_EXTENSIONS = ['.jpg', '.jpeg', '.png']
```

### 3. 해상도
```python
MIN_DIMENSION = 512x512
MAX_DIMENSION = 4096x4096
```

---

## 🔍 사용 예시

```python
from app.utils.image import ImageValidator

# 검증
is_valid, error_msg, metadata = ImageValidator.validate(
    file_data=file_bytes,
    content_type="image/jpeg",
    filename="photo.jpg"
)

if not is_valid:
    raise AppException(
        status_code=400,
        error_code="E-IMG-005",
        message=error_msg
    )

print(metadata)
# {
#     "width": 1024,
#     "height": 768,
#     "format": "JPEG",
#     "size_bytes": 524288
# }
```

---

## ✅ 테스트 시나리오

### 1. 정상 이미지
```python
def test_valid_image():
    # 1024x768 JPEG
    is_valid, msg, meta = ImageValidator.validate(...)
    assert is_valid == True
    assert meta["width"] == 1024
```

### 2. 파일 크기 초과
```python
def test_file_too_large():
    # 11MB 파일
    is_valid, msg, meta = ImageValidator.validate(...)
    assert is_valid == False
    assert "exceeds" in msg
```

### 3. 잘못된 형식
```python
def test_invalid_format():
    # GIF 파일
    is_valid, msg, meta = ImageValidator.validate(
        file_data=gif_bytes,
        content_type="image/gif",
        filename="test.gif"
    )
    assert is_valid == False
```

---

## 📝 체크리스트

- [ ] app/utils/image.py 생성
- [ ] ImageValidator 클래스 구현
- [ ] validate() 메서드 구현
- [ ] 모든 검증 규칙 구현
- [ ] 테스트 작성

---

## 🚀 다음 단계

이미지 검증 완료 → **s3-upload.md**

