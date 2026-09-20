# Email Search Keyword Extraction

When searching Zoom Mail by case Description, long or mixed-language
descriptions often fail to match. This reference explains the cleaning logic.

## Problem

ServiceNow ticket descriptions often contain:
- Prefixes in parentheses: `(Premier)`, `(Problem Report Sent)`
- Tags in brackets: `[New Tenant]`, `[신규 테넌트]`
- Mixed Korean and English text separated by `-`
- Special characters that break search tokenization

## Solution: extract_search_keywords()

```python
import re

def extract_search_keywords(desc: str) -> str:
    cleaned = desc
    # Remove common prefixes
    cleaned = re.sub(r'^\(Premier\)\s*', '', cleaned)
    cleaned = re.sub(r'^\(Problem Report Sent\)\s*', '', cleaned)
    cleaned = re.sub(r'\[New Tenant\]\s*', '', cleaned)
    cleaned = re.sub(r'\[신규 테넌트\]\s*', '', cleaned)

    # For mixed Korean-English with '- ', prefer the Korean portion
    if '- ' in cleaned and any('\uac00' <= c <= '\ud7a3' for c in cleaned):
        parts = cleaned.split('- ')
        korean_part = parts[0].strip()
        if len(korean_part) > 10:
            return korean_part

    # Remove special chars, keep Korean/English/numbers/spaces
    cleaned = re.sub(r'[^\w\s\uac00-\ud7a3]', ' ', cleaned)
    cleaned = re.sub(r'\s+', ' ', cleaned).strip()

    if len(cleaned) > 80:
        cleaned = cleaned[:80]

    return cleaned
```

## Examples

| Input | Output |
|-------|--------|
| `(Premier) [신규 테넌트] 관리자 콘솔에서 등록한 가상 배경 이미지가 이슈- [New Tenant] Issue with...` | `관리자 콘솔에서 등록한 가상 배경 이미지가 이슈` |
| `(Problem Report Sent) Unstable performance but not a network issue.` | `Unstable performance but not a network issue` |
| `워크스페이스 QR 예약 표시 값 문의` | `워크스페이스 QR 예약 표시 값 문의` |
| `ZoomRoom health error` | `ZoomRoom health error` |
