# Suno 인증 자동화 연구 히스토리

> 작성일: 2026-04-12  
> 목적: Suno API 쿠키 자동 갱신 방법 탐색 전 과정 기록

---

## 1. 초기 가정 (틀렸음)

- `__session` 쿠키가 Bearer JWT 역할, 만료 ~1시간
- 만료 시 수동으로 브라우저에서 복사해야 함

---

## 2. 실제 인증 구조 발견

### Clerk 기반 인증 흐름
```
__client (JWT refresh token, 1년 만료, auth.suno.com, httpOnly 쿠키)
  → GET https://clerk.suno.com/v1/client          → session_id 획득
  → POST /v1/client/sessions/{sid}/tokens         → JWT 60초짜리 발급
  → 모든 Suno API 호출: Authorization: Bearer {JWT}
```

- `__client` 쿠키: JWT 형태의 refresh token, **만료 1년**
- 단기 JWT: 60초마다 재발급 (자동)
- `__session`: 실제로는 보조적 역할, Bearer 토큰이 아님

---

## 3. 탐색 경로 및 실패 기록

### 3-1. Clerk API 직접 호출 시도
- **시도**: Clerk sign-in API로 이메일+비밀번호 로그인 프로그래밍 구현
- **결과**: 실패
- **원인**: Suno는 이메일+비밀번호 로그인을 **비활성화**함. Google OAuth, Apple, Discord, Facebook, Microsoft, SMS만 지원
- **교훈**: Clerk API의 이메일 로그인 엔드포인트가 존재해도 Suno 앱 키로는 차단됨

### 3-2. Google OAuth 자동화 시도
- **시도**: Selenium/Playwright로 Google OAuth 플로우 자동화
- **결과**: 실패
- **원인**: Google이 자동화 브라우저를 감지하여 차단 (reCAPTCHA, 계정 보호 정책)
- **교훈**: Google OAuth는 실제 사용자 세션(프로필)에서만 안정적으로 동작

### 3-3. Twilio SMS 자동화 시도
- **시도**: Twilio로 SMS 인증 자동화 (Suno의 SMS 로그인 활용)
- **결과**: 실패
- **원인**: Suno(Clerk)가 **VoIP 번호 차단** — Twilio 번호는 VoIP로 분류되어 SMS 수신 불가
- **교훈**: SMS 자동화는 실제 통신사 번호가 필요하므로 현실적으로 불가

### 3-4. check_auth 403 디버깅
- **증상**: `GET https://clerk.suno.com/v1/client` 호출 시 403 반환
- **원인 분석**: Origin, User-Agent 헤더 누락
- **해결**: 다음 헤더 필수 추가
  ```python
  headers = {
      "Origin": "https://suno.com",
      "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...",
      "Referer": "https://suno.com/",
  }
  ```
- **교훈**: Clerk API는 브라우저 환경을 시뮬레이션하지 않으면 403 반환

### 3-5. document.cookie로 httpOnly 쿠키 추출 시도
- **시도**: `browser-use eval "document.cookie"`로 `__client` 추출
- **결과**: 실패
- **원인**: `__client` 쿠키는 **httpOnly 플래그** → JavaScript에서 접근 불가
- **해결**: `browser-use cookies get` 명령 사용 (브라우저 내부 쿠키 저장소에 직접 접근)
  ```bash
  browser-use --profile nicejames cookies get --domain suno.com
  ```

---

## 4. 최종 솔루션

### 선택한 방법: browser-use 프로필 기반 쿠키 갱신

**왜 이 방법인가:**
- `__client` refresh token은 1년 만료 → 자주 갱신 불필요
- 갱신 필요 시 실제 사용자 프로필로 브라우저 열면 자동 재발급
- httpOnly 쿠키도 `browser-use cookies get`으로 추출 가능

### auto_refresh_cookie.py 스크립트

- **위치**: `~/Projects/clones/zach-suno-api/scripts/auto_refresh_cookie.py`
- **동작**: 30분 간격으로 현재 `__client` 쿠키 유효성 체크, 만료 임박 시 browser-use로 자동 갱신
- **사용법**:
  ```bash
  # 단일 실행 (현재 상태 확인 + 필요 시 갱신)
  python3 scripts/auto_refresh_cookie.py --once

  # 상시 감시 모드
  python3 scripts/auto_refresh_cookie.py
  ```

### 수동 갱신 절차 (쿠키 완전 만료 시)
```bash
# 1. browser-use로 nicejames 프로필로 Suno 열기
browser-use --profile nicejames open "https://suno.com"

# 2. httpOnly 쿠키 포함 전체 쿠키 추출
browser-use --profile nicejames cookies get --domain suno.com

# 3. __client 값을 .env에 업데이트
# SUNO_COOKIE=__client=eyJ...
```

---

## 5. 핵심 교훈 요약

| 항목 | 내용 |
|------|------|
| `__client` 만료 | **1년** (기존 "7일" 가정 틀림) |
| Clerk API 403 원인 | Origin, User-Agent 헤더 누락 |
| httpOnly 쿠키 추출 | `browser-use cookies get` 사용 (document.cookie 불가) |
| 이메일 로그인 | Suno에서 비활성화됨 |
| SMS 자동화 | Twilio VoIP 번호 차단으로 불가 |
| 최적 솔루션 | browser-use nicejames 프로필 + auto_refresh_cookie.py |

---

## 6. 관련 파일

- `step1_suno_generation_guide.md` — 2-4. 쿠키 관리 섹션 업데이트됨
- `~/Projects/clones/zach-suno-api/scripts/auto_refresh_cookie.py` — 자동 갱신 스크립트
- `~/.claude/projects/-Users-nicejames-Projects-clones/memory/reference_suno_auth.md` — 메모리 참조
