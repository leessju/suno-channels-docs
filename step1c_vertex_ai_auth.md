# Vertex AI 인증 가이드

> 작성일: 2026-04-11  
> 목적: Vertex AI (Gemini) 크레딧 기반 영상 분석 인증 구조 및 계정 정보 기록

---

## 1. 인증 방식

**OAuth2 Refresh Token** (authorized_user 타입)

- GCP 서비스 계정 키(SA JSON)가 아닌 **사용자 OAuth2** 방식
- ADC(Application Default Credentials) 파일 건드리지 않고 직접 credential 주입
- `google.oauth2.credentials.Credentials` → `google.genai.Client`에 주입

```
credentials_{account}.json (refresh_token)
  → google.auth.transport.requests.Request()로 access token 갱신
  → genai.Client(vertexai=True, project=..., location=..., credentials=...)
  → Gemini 2.5 Flash API 호출
```

---

## 2. 계정 정보

### 활성 계정

| 계정명 | GCP 프로젝트 ID | 리전 | 상태 |
|--------|----------------|------|------|
| account3 | `project-3f3c3c98-9d26-4383-bcb` | us-central1 | **active** |

### 전체 계정 (로테이션용)

| 계정명 | quota_project_id | 상태 |
|--------|-----------------|------|
| account1 | `gen-lang-client-0100430126` | 비활성 (gcp_accounts.json 미등록) |
| account2 | `project-7ddaa991-9b35-4b8e-855` | 비활성 (gcp_accounts.json 미등록) |
| account3 | `project-3f3c3c98-9d26-4383-bcb` | **active** |

> account2, account3는 동일한 `client_id` 사용 (`764086051850-6qr4p6g...`)

---

## 3. 파일 위치

```
~/Projects/claude/test/niche_bending/infra/
├── gcp_accounts.json           ← 활성 계정 목록 및 현재 인덱스
├── credentials_account1.json   ← OAuth2 토큰 (account1)
├── credentials_account2.json   ← OAuth2 토큰 (account2)
├── credentials_account3.json   ← OAuth2 토큰 (account3) ← 현재 사용
├── vertex_client.py            ← 계정 로테이션 클라이언트
├── run_vertex.py               ← 배치 영상 분석 실행기
└── analysis_prompt.md          ← 분석 프롬프트
```

---

## 4. 사용법

### 배치 분석 (DB pending 영상 일괄)

```bash
cd ~/Projects/claude/test/niche_bending

# niche 타입 영상 5개 워커로 분석
PYTHONUNBUFFERED=1 python3 infra/run_vertex.py niche 5

# 워커 1개 (안정적)
PYTHONUNBUFFERED=1 python3 infra/run_vertex.py niche 1
```

> **주의**: `PYTHONUNBUFFERED=1` 없으면 출력이 버퍼링되어 실시간 확인 불가

### 단일 영상 직접 분석

```python
import sys, json, time, re
sys.path.insert(0, 'infra')
from google.genai import types
from vertex_client import generate_with_rotation

PROMPT = open('infra/analysis_prompt.md', encoding='utf-8').read()

video_info = """
---
영상 정보:
- video_id: VIDEO_ID
- channel: 채널명
- title: 영상 제목
- views: 조회수
- subscribers: 구독자수
- viral_ratio: VR값
- duration_seconds: 초
- format: shorts
"""

response = generate_with_rotation(
    contents=[
        types.Content(role='user', parts=[
            types.Part(text=PROMPT + video_info),
            types.Part(file_data=types.FileData(
                file_uri='https://www.youtube.com/shorts/VIDEO_ID',
                mime_type='video/*'
            ))
        ])
    ],
    max_retries=3,
)
print(response.text)
```

**소요 시간**: 영상당 약 45~90초

---

## 5. 계정 로테이션 구조

`vertex_client.py`가 API 리밋 초과 시 자동으로 다음 계정으로 전환.

```python
# gcp_accounts.json에 계정 추가하면 자동 로테이션에 포함
{
  "accounts": [
    { "name": "account1", "project": "...", "location": "us-central1", "status": "active" },
    { "name": "account2", "project": "...", "location": "us-central1", "status": "active" },
    { "name": "account3", "project": "...", "location": "us-central1", "status": "active" }
  ],
  "current_index": 0
}
```

---

## 6. 결과 저장 구조

| 항목 | 위치 |
|------|------|
| JSON 결과 파일 | `analysis_results/{video_id}.json` |
| DB | `~/.claude/gems/gems.db` → `analysis_results` 테이블 |
| 분석 상태 | `videos.analysis_status` → `pending` → `done` |

---

## 7. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| 실행 후 출력 없음 | 출력 버퍼링 | `PYTHONUNBUFFERED=1` 추가 |
| `type 'dict' is not supported` | genre 필드가 dict | `str()` 또는 `.get('main')` 변환 |
| `⏳ 모든 계정 리밋. 60초 대기...` | API 쿼터 초과 | 자동 대기 후 재시도 (정상 동작) |
| `channel_followers` 컬럼 없음 | DB 스키마 확인 | `PRAGMA table_info(videos)` 로 실제 컬럼명 확인 |
| 프로세스 실행 중 결과 없음 | API 응답 대기 | 단일 분석은 45~90초 정상 |

---

## 8. 관련 파일

- `step1b_auth_automation_research.md` — Suno OAuth2 인증 구조 (Clerk 기반)
- `~/Projects/claude/test/niche_bending/docs/vertex_ai_usage.md` — 상세 사용 가이드
- `~/.claude/projects/.../memory/reference_vertex_ai_usage.md` — 메모리 참조
