# STEP 1: Suno 음악 생성 가이드

> 자동화 파이프라인의 첫 단계. mp3 + 커버 + 가사 + suno_id + 메타데이터 일괄 저장.
> 전체 흐름: `step0_pipeline_overview.md` 참조.

---

## 1. 목표

Suno AI로 음악을 생성하면서 **모든 데이터를 빠짐없이 저장**한다.

### 필수 저장 항목
| 항목 | 파일/DB | 용도 |
|------|---------|------|
| mp3 파일 | `01_songs/{번호}_{곡명}.mp3` | 영상 오디오 |
| 커버아트 | `01_songs/{번호}_{곡명}.jpeg` | Remotion 커버 |
| 가사 텍스트 | `01_songs/{번호}_{곡명}.txt` | Whisper 매칭용 |
| suno_id (v1) | gems.db `suno_tracks.id` | v4.5 버전 |
| suno_id (v2) | gems.db `suno_tracks.v2_id` | v5.5 버전 (Cover 업그레이드) |
| Suno metadata | gems.db `suno_tracks.*` | title, style, duration 등 |
| 프롬프트 | gems.db `suno_tracks.prompt_json` | 재생성 가능하도록 |

### 왜 모든 데이터 저장?
- hero-vol1/vol2에서 suno_id를 저장하지 않아 **20곡분 가사 재취득 불가능**했던 이력
- v2 ID 미저장으로 cafe 20곡 낭비
- **나중에 Suno API `get?ids={id}` → lyric 필드**로 정확한 가사를 받으려면 ID 필수

---

## 2. 인프라

### 2-1. Suno API (zach-suno-api)
- **경로**: `~/Projects/clones/zach-suno-api/`
- **기반**: zach-fau/suno-api (Playwright + 2Captcha)
- **환경변수 (.env)**:
  ```bash
  SUNO_COOKIE=<브라우저에서 추출>
  CAPSOLVER_API_KEY=<2Captcha 키>
  BROWSER_LOCALE=en         # ⚠️ 필수! ko-KR은 캡차 파라미터 오류
  DEFAULT_MODEL=chirp-fenix  # v5.5
  ```

### 2-2. DB (gems.db)
```sql
CREATE TABLE suno_tracks (
  id TEXT PRIMARY KEY,        -- suno_id (v1, 예: chirp-v4.5)
  v2_id TEXT,                 -- suno_id (v2, 예: chirp-v5.5)
  title TEXT,
  lyric TEXT,                 -- Suno API에서 받은 정확한 가사
  style TEXT,                 -- 장르/스타일 키워드
  cover_url TEXT,             -- Suno 원본 커버 URL
  audio_url TEXT,             -- Suno 원본 오디오 URL
  duration REAL,              -- 초 단위
  prompt_json TEXT,           -- 생성 시 사용한 전체 프롬프트
  mp3_path TEXT,              -- 로컬 저장 경로
  jpeg_path TEXT,
  txt_path TEXT,
  playlist_id INTEGER,
  channel TEXT,
  created_at TIMESTAMP,
  status TEXT                 -- generated | downloaded | rendered | uploaded
);

CREATE TABLE suno_playlists (
  id INTEGER PRIMARY KEY,
  name TEXT UNIQUE,           -- "hero-vol1"
  channel TEXT,               -- "jpop" | "cafe" | "phonk"
  folder_path TEXT,
  track_count INTEGER,
  total_duration REAL,
  status TEXT,                -- planning | generating | rendered | uploaded
  created_at TIMESTAMP
);
```

### 2-3. Workspace 동기화 (suno_workspace_sync.py)

Suno의 **workspace = project** 단위로 곡을 관리하고 로컬과 자동 동기화.

- **스크립트**: `~/Music/suno-channels/scripts/suno_workspace_sync.py`
- **내부 API**: `https://studio-api-prod.suno.com` (zach-suno-api와 다른 URL)

```bash
# workspace 목록 확인
python3 scripts/suno_workspace_sync.py --list

# 특정 workspace 곡 목록 확인 (다운로드 없음)
python3 scripts/suno_workspace_sync.py --id <workspace_id> --dry-run

# 특정 workspace 동기화 (MP3 + 커버 + DB 저장)
python3 scripts/suno_workspace_sync.py --id <workspace_id>

# 전체 workspace 동기화
python3 scripts/suno_workspace_sync.py
```

**핵심 엔드포인트** (Proxelar 역분석으로 확인):
| 목적 | 방법 | URL |
|------|------|-----|
| workspace 목록 | GET | `/api/project/me?page=1&sort=max_created_at_last_updated_clip` |
| workspace 곡 목록 | POST | `/api/feed/v3` + `{"workspace": {"presence": "True", "workspaceId": "..."}}` |
| workspace 생성 | POST | `/api/project` |
| workspace에 곡 추가 | POST | `/api/project/{id}/clips` |
| workspace 이름/설명 변경 | POST | `/api/project/{id}/metadata` + `{"name":"...", "description":"..."}` |
| workspace 삭제/복원 | POST | `/api/project/trash` + `{"project_id":"...", "undo_trash": false\|true}` |

**인증**: `__client` 쿠키(JWT refresh token, 만료 1년) → Clerk API로 60초짜리 JWT 무한 갱신 → Bearer 토큰으로 사용

### 2-4. 쿠키 관리

> **중요 수정 (2026-04-12)**: 기존 `__session` 1시간 만료 가정은 틀렸음.  
> 실제 구조는 `__client` refresh token(1년) → 단기 JWT(60초) 무한 갱신이다.

#### 인증 흐름
```
__client (JWT refresh token, 1년 만료, auth.suno.com, httpOnly 쿠키)
  → GET https://clerk.suno.com/v1/client  → session_id 획득
  → POST /v1/client/sessions/{sid}/tokens → JWT 60초짜리 발급
  → 모든 Suno API 호출: Authorization: Bearer {JWT}
```

#### 핵심 사항
- `__client` 쿠키는 **httpOnly** → `document.cookie`로 추출 불가
- **browser-use cookies get**으로만 추출 가능:
  ```bash
  browser-use --profile nicejames cookies get --domain suno.com
  ```
- Clerk API 호출 시 **Origin, User-Agent 헤더 필수** (없으면 403)
- Suno 로그인 방식: Google OAuth, Apple, Discord, Facebook, Microsoft, SMS만 지원. **이메일+비밀번호 없음**

#### 자동 갱신 (권장)
```bash
# 단일 체크 (현재 쿠키 유효성 확인 + 필요 시 갱신)
python3 ~/Projects/clones/zach-suno-api/scripts/auto_refresh_cookie.py --once

# 상시 감시 (30분 간격 자동 체크)
python3 ~/Projects/clones/zach-suno-api/scripts/auto_refresh_cookie.py
```

#### 수동 갱신 (쿠키 만료 시)
```bash
# browser-use로 Suno 열고 쿠키 추출
browser-use --profile nicejames open "https://suno.com"
browser-use --profile nicejames cookies get --domain suno.com
# __client 값을 .env의 SUNO_COOKIE에 업데이트
```

---

## 3. 코드/가사 참조 워크플로우

### 3-0. 레퍼런스 곡 분석 → Suno 프롬프트 변환

Suno에서 감흥 있는 곡을 만들기 위해 **유명 곡의 코드 진행과 가사 구조를 참조**한다.
Chordino를 이용한 자동 코드 추출이 가능하며, `step1a_midi_cover_research.md`에서 상세히 다룬다. 또한 **수동 분석 → DB 저장 → 프롬프트 활용** 방식도 병행.

#### 전체 흐름

```
1. 유명 곡 청취/악보 분석 (Mr.Children HERO, タガタメ 등)
   ↓
2. gems.db suno_chord_library에 수동 INSERT
   - name, artist, key_signature, bpm
   - chord_progression (예: "F#m → C#m → D → A")
   - section (verse/chorus), mood, emotion_effect
   ↓
3. Suno custom_generate style 태그에 코드 삽입
   - "emotional rock ballad, F#m → C#m → D → A, ..."
   ↓
4. 가사는 유명 작사가 곡의 구조/감정선 참고 후
   직접 새로 작성 ([Verse][Chorus][Bridge] 풀 구조)
```

#### suno_chord_library DB 스키마

```sql
-- gems.db
CREATE TABLE suno_chord_library (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,              -- 곡명 (예: "HERO")
    artist TEXT,            -- 아티스트 (예: "Mr.Children")
    key_signature TEXT,     -- 키 (예: "A major")
    bpm INTEGER,
    chord_progression TEXT, -- "F#m → C#m → D → A"
    section TEXT,           -- verse / chorus / bridge
    mood TEXT,              -- hopeful, dramatic, warm 등
    emotion_effect TEXT,    -- 감정 효과 설명
    created_at TIMESTAMP
);
```

#### 현재 저장된 데이터

| 곡명 | 아티스트 | 코드 진행 | 섹션 | 무드 |
|------|---------|-----------|------|------|
| HERO | Mr.Children | F#m→C#m→D→A | chorus | hopeful, bittersweet |
| HERO | Mr.Children | A→A→F#m→B | verse | warm, stable |
| タガタメ | Mr.Children | D→G→D→F#m→A→D→Em→D | verse-chorus | dramatic, anthemic |

#### 가사 참조 규칙

- 유명 작사가 곡의 **구조(Verse/Chorus/Bridge 배치)**, **감정선(희망→그리움→결심)**, **톤(구어체/문어체)**만 참조
- 가사 내용은 **직접 새로 작성** (저작권 이슈 회피)
- 일본어 곡: 僕/私 표현, 부드러운 구어체 유지

#### 새 레퍼런스 곡 추가 시

1. 곡을 듣거나 악보/Chordify 등에서 코드 확인
2. `gems.db`에 INSERT:
   ```sql
   INSERT INTO suno_chord_library (name, artist, chord_progression, section, mood)
   VALUES ('新曲名', 'アーティスト', 'Am → F → C → G', 'chorus', 'uplifting');
   ```
3. Suno style 태그에 코드 삽입하여 생성

> **참고**: librosa는 에너지 분석(트랙 순서 정렬)에만 사용. 코드 자동 추출 도구는 미구현 상태.

---

## 4. 생성 워크플로우

### 4-1. 곡 생성 전략 (감흥 있는 곡 공식)

**검증된 공식 (hero-playlist 기준):**

| 항목 | 필수값 | 금지 |
|------|--------|------|
| **장르** | `emotional rock ballad, acoustic pop` | `dream pop` 단독 (밍밍함) |
| **필수 키워드** | `dramatic chord changes, cinematic atmosphere, analog warmth, uplifting chorus` | - |
| **보컬** | `male vocal tone` 또는 `female vocal tone` 명시 | 미지정 (랜덤됨) |
| **가사** | 직접 작성, `[Verse][Pre-Chorus][Chorus][Bridge]` 풀 구조 | Suno 자동 생성 (감정선 없음) |
| **BPM** | 72~94 넓게 변형 | - |
| **악기** | `bright piano, acoustic guitar, string ensemble` 3종 세트 | - |
| **제한 키워드** | - | `no distortion` 등 "no" 계열 (감흥 죽임) |

### 4-2. 버전 전략 (v4.5 → v5.5 업그레이드)

Suno는 v4.5가 **느낌이 좋고**, v5.5가 **음질이 좋다**. 두 장점을 합치는 방법:

```
1. v4.5로 생성 → 감정/분위기 좋은 곡 선별
2. 선별된 곡을 Cover 모드로 v5.5 업그레이드
   - Audio Influence: 100% (원본 유지)
   - Style Influence: 30%
   - Weirdness: 20%
3. v2_id (v5.5 업그레이드 버전) 저장
```

### 4-3. 장르별 가사 톤

| 채널 | 언어 | 특징 | 금지 |
|------|------|------|------|
| jpop | 일본어 | 僕/私, 부드러운 표현, 구어체 | 俺/거친 표현 (파워풀해짐) |
| cafe | (instrumental) | `[Instrumental]` 프롬프트 필수 | 빈 문자열 (에러) |
| phonk | 영어/포르투갈어 | 俺/거침/슬랭 OK | 일본어 (부적합) |

### 4-4. 플레이리스트 내 변주

20곡 플레이리스트는 **같은 장르 안에서 코드/무드/BPM 변형**:

```
HERO 코드 예시:
  F#m → C#m → D → A (감정 핵심)

변주 방법:
  - BPM: 72, 78, 84, 90, 94 (5곡씩)
  - 분위기: 희망 → 그리움 → 고독 → 결심 → 해방
  - 악기 강조: 피아노 우세 / 기타 우세 / 스트링 우세
```

---

## 5. Python 자동화 스크립트

### 5-1. 플레이리스트 생성 설정

```python
# ~/Projects/clones/zach-suno-api/generate_playlist.py

PLAYLIST_CONFIG = {
    "name": "hero-vol1",
    "channel": "jpop",
    "folder": "~/Music/suno-channels/jpop/260402_hero-vol1",
    "songs": [
        {
            "order": 1,
            "title": "ヒーローになれなくても",
            "style": "emotional rock ballad, acoustic pop, dramatic chord changes, "
                     "cinematic atmosphere, analog warmth, uplifting chorus, "
                     "bright piano, acoustic guitar, string ensemble, male vocal tone",
            "bpm": 84,
            "lyrics": """
[Verse 1]
...
[Chorus]
...
""",
        },
        # ...19곡 더
    ]
}
```

### 5-2. 생성 + 저장 파이프라인

```python
import requests
import json
import sqlite3
import os
from pathlib import Path

SUNO_API = "http://localhost:3000/api"  # zach-suno-api 로컬 서버
DB_PATH = "~/Projects/claude/test/niche_bending/gems.db"

def generate_song(song_config: dict, playlist_folder: str):
    """한 곡 생성 + 모든 데이터 저장"""

    # 1. v4.5 생성
    resp = requests.post(f"{SUNO_API}/custom_generate", json={
        "title": song_config["title"],
        "tags": song_config["style"],
        "prompt": song_config["lyrics"],
        "mv": "chirp-v4-5",
        "make_instrumental": False,
        "wait_audio": True,
    })
    tracks = resp.json()

    if len(tracks) < 1:
        raise Exception("Suno generation failed")

    # 2. 두 버전 중 더 좋은 것 선택 (또는 둘 다 저장)
    v1_track = tracks[0]
    suno_id_v1 = v1_track["id"]

    # 3. v5.5 Cover 업그레이드
    resp_v2 = requests.post(f"{SUNO_API}/cover", json={
        "source_id": suno_id_v1,
        "mv": "chirp-fenix",
        "audio_influence": 1.0,
        "style_influence": 0.3,
        "weirdness": 0.2,
    })
    v2_track = resp_v2.json()[0]
    suno_id_v2 = v2_track["id"]

    # 4. Suno API에서 정확한 가사 받기 (프롬프트 가사 아님)
    lyric_resp = requests.get(f"{SUNO_API}/get?ids={suno_id_v2}")
    lyric = lyric_resp.json()[0]["lyric"]

    # 5. mp3 다운로드 (v2 버전)
    mp3_url = v2_track["audio_url"]
    order = song_config["order"]
    title = song_config["title"]
    base = f"{playlist_folder}/01_songs/{order:02d}_{title}"

    mp3_data = requests.get(mp3_url).content
    with open(f"{base}.mp3", "wb") as f:
        f.write(mp3_data)

    # 6. 커버아트 다운로드
    cover_url = v2_track["image_url"]
    jpeg_data = requests.get(cover_url).content
    with open(f"{base}.jpeg", "wb") as f:
        f.write(jpeg_data)

    # 7. 가사 txt 저장
    with open(f"{base}.txt", "w", encoding="utf-8") as f:
        f.write(lyric)

    # 8. DB에 모든 메타 저장
    conn = sqlite3.connect(os.path.expanduser(DB_PATH))
    conn.execute("""
        INSERT INTO suno_tracks
        (id, v2_id, title, lyric, style, cover_url, audio_url,
         duration, prompt_json, mp3_path, jpeg_path, txt_path,
         playlist_id, channel, status)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        suno_id_v1, suno_id_v2,
        song_config["title"],
        lyric,
        song_config["style"],
        cover_url,
        mp3_url,
        v2_track.get("duration"),
        json.dumps(song_config, ensure_ascii=False),
        f"{base}.mp3",
        f"{base}.jpeg",
        f"{base}.txt",
        song_config.get("playlist_id"),
        "jpop",
        "downloaded",
    ))
    conn.commit()
    conn.close()

    print(f"✅ {order:02d}_{title}: v1={suno_id_v1}, v2={suno_id_v2}")
    return {
        "suno_id_v1": suno_id_v1,
        "suno_id_v2": suno_id_v2,
        "mp3": f"{base}.mp3",
        "jpeg": f"{base}.jpeg",
        "txt": f"{base}.txt",
    }


def generate_playlist(config: dict):
    """플레이리스트 전체 생성"""
    folder = os.path.expanduser(config["folder"])
    os.makedirs(f"{folder}/01_songs", exist_ok=True)

    results = []
    for song in config["songs"]:
        try:
            result = generate_song(song, folder)
            results.append(result)
        except Exception as e:
            print(f"❌ {song['title']}: {e}")
            # 재시도 로직 추가 가능

    return results
```

### 5-3. 배경 이미지 자동 할당 (assign_background_images.py)

Vol 분리 + 순서 배치 후, 각 곡의 배경 이미지를 장르별 이미지 풀에서 균등 배분하여 할당한다.

- **스크립트**: `~/Music/suno-channels/scripts/assign_background_images.py`
- **이미지 풀 경로**:
  - `assets/back_image/jpop/` — 일반 배경 (j_001~j_100.png)
  - `assets/back_image/jpop/cover/` — 커버 전용 이미지 (썸네일 겸용 가능)
  - `assets/back_image/phonk/` — 일반 배경 (p_001~p_024.png)
  - `assets/back_image/phonk/cover/` — 커버 전용 이미지

**할당 규칙:**
| 곡 번호 | 사용 가능 풀 | 비고 |
|--------|------------|------|
| 01번 곡 | `cover/` 풀만 | 썸네일 이미지와 동일 소스 |
| 02번~ 곡 | `cover/` + 일반 풀 전체 | |

- **중복 방지**: vol 내에서 같은 이미지 재사용 금지
- **균등 배분**: gems.db `back_image_usage` 테이블에 사용 횟수 기록 → 적게 쓰인 이미지 우선
- **파일 형식**: 기존 `.jpeg` 삭제 후 `.png`로 통일하여 저장

**gems.db 신규 테이블:**
```sql
-- 이미지별 사용 횟수 추적
CREATE TABLE back_image_usage (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    image_path TEXT UNIQUE,   -- 예: "jpop/j_001.png"
    genre TEXT,               -- "jpop" | "phonk"
    image_type TEXT,          -- "cover" | "general"
    use_count INTEGER DEFAULT 0,
    updated_at TEXT
);

-- vol-곡 단위 배정 이력
CREATE TABLE vol_song_images (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    vol_name TEXT,            -- 예: "260402_hero-vol1"
    song_filename TEXT,       -- 예: "01_ヒーローになれなくても.png"
    image_path TEXT,          -- 배정된 이미지 경로
    assigned_at TEXT,
    UNIQUE(vol_name, song_filename)
);
```

**사용법:**
```bash
# 전체 vol 처리
python3 scripts/assign_background_images.py

# 미리보기 (파일 변경 없음)
python3 scripts/assign_background_images.py --dry-run

# 특정 장르만
python3 scripts/assign_background_images.py jpop

# 특정 vol만
python3 scripts/assign_background_images.py --vol 260402_hero-vol1

# 사용 현황 통계 출력
python3 scripts/assign_background_images.py --stats
```

---

## 6. 트러블슈팅

### 6-1. 캡차 오류
**증상**: `"CAPTCHA_FAILED"` 또는 무한 대기
**해결**:
1. `.env`에 `BROWSER_LOCALE=en` 확인 (ko-KR이면 파라미터 오류)
2. `CAPSOLVER_API_KEY` 잔액 확인
3. Suno 웹에서 수동으로 한 번 로그인 → 쿠키 재추출

### 6-2. 쿠키 만료
**증상**: 401 Unauthorized
**해결**:
1. Chrome → Suno 웹 → F12 → Application → Cookies
2. 전체 복사 → `.env`의 `SUNO_COOKIE` 갱신
3. 서버 재시작

### 6-3. 가사 미스매치
**증상**: Suno 생성된 mp3의 가사가 프롬프트와 다름
**원인**: Suno가 가사를 재해석하거나 일부 변경
**해결**: **반드시 `get?ids={id}` API로 가사를 받아서 txt 저장** (프롬프트 가사 사용 금지)

### 6-4. v2 ID 누락
**증상**: Cover 업그레이드 실패 또는 v2_id 저장 누락
**해결**: 저장 실패 시 즉시 재시도, DB에 v2_id 컬럼이 NULL인 곡 정기 재처리

### 6-5. 여러 버전 중 선택
**증상**: Suno는 항상 2개 버전 반환 → 더 좋은 쪽 선택 필요
**해결**: 둘 다 저장하고 나중에 선별
```python
# tracks[0], tracks[1] 둘 다 저장
for i, track in enumerate(tracks):
    save_to_db(track, suffix=f"_v{i+1}")
```

---

## 7. 체크리스트

### 생성 전
- [ ] `.env` 설정 확인 (`BROWSER_LOCALE=en`)
- [ ] Suno 쿠키 유효 확인 (수동 테스트)
- [ ] 2Captcha 잔액 확인
- [ ] 플레이리스트 폴더 구조 생성
- [ ] 프롬프트 감흥 공식 적용 (장르 + 키워드 + vocal gender + 가사 작성)

### 생성 중
- [ ] v1, v2 ID 둘 다 저장
- [ ] Suno API에서 가사 받기 (프롬프트 가사 아님)
- [ ] mp3 + jpeg + txt 3종 세트 다운로드
- [ ] DB 기록 (모든 필드)

### 생성 후
- [ ] 20곡 전부 DB에 있는지 확인
- [ ] `01_songs/` 폴더에 3종 세트 × 20 = 60개 파일 확인
- [ ] 곡 길이 확인 (너무 짧거나 긴 곡 필터링)
- [ ] 다음 단계 (STEP 2: 오디오 후처리)로 이동

---

## 8. 자동화 함수 시그니처

```python
def generate_song(
    song_config: dict,      # {order, title, style, lyrics, bpm, ...}
    playlist_folder: str,
    channel: str = "jpop",
    save_v1: bool = True,
    save_v2: bool = True,
    cover_upgrade: bool = True,  # v4.5 → v5.5
) -> dict:
    """한 곡 생성 + 모든 데이터 저장"""
    ...


def generate_playlist(
    config: dict,           # PLAYLIST_CONFIG 구조
    retry_failed: bool = True,
    parallel: int = 1,      # Suno는 동시 생성 제한 있음
) -> list[dict]:
    """플레이리스트 전체 생성"""
    ...


def resync_missing_data(
    playlist_name: str,
) -> dict:
    """DB에 있지만 파일 없음 또는 그 반대 케이스 복구"""
    ...
```

---

## 9. 통합 파이프라인 설계 (Unified Pipeline)

이 섹션은 **단일 JSON 설정 파일로 전체 워크플로우를 자동화**하는 `generate_playlist.py` 스크립트의 설계 명세를 다룬다. 다른 세션이 이 문서만으로 구현할 수 있도록 극도로 상세하게 작성했다.

### 9-1. 전체 흐름

```
JSON 설정 파일
  ↓
Workspace 확인/생성 (jpop-vol10 없으면 자동 생성)
  ↓
[source 있으면] YouTube URL → yt-dlp → mp3 다운로드
[source 있으면] mp3 → Demucs (MPS GPU, -d mps) → 보컬 제거
[source 있으면] Chordino (roll_on=0.5) → Key 다이어토닉 필터 → 코드 추출
[source 있으면] 코드 → MIDI → fluidsynth → mp3 렌더링
[source 있으면] Suno 업로드 → clip_id (browser-token 필수)
  ↓
N곡 생성 (각각 다른 가사/제목/vocal_gender/weirdness/style_influence)
  - source 있으면 → Cover 모드 (cover_clip_id)
  - source 없으면 → 일반 생성 (스타일+가사만)
  ↓
생성된 곡을 Workspace에 추가 (POST /api/project/{id}/clips)
  ↓
mp3 + jpeg(커버아트) + txt(가사) 다운로드
  ↓
gems.db 저장 (suno_tracks 테이블)
  ↓
[optional] Demucs로 생성된 곡의 vocal 분리 (step2 준비)
```

### 9-2. JSON 설정 파일 구조

```json
{
  "source": "https://youtube.com/watch?v=xxx",
  "playlist": "jpop-vol10",
  "channel": "jpop",
  "model": "chirp-fenix",
  "style": "emotional J-pop, acoustic pop, bright piano, acoustic guitar, string ensemble, dramatic chord changes, cinematic atmosphere, analog warmth, uplifting chorus, 84 BPM",
  "negative_tags": "distortion, heavy metal",
  "soundfont": "~/Music/soundfonts/FluidR3_GM.sf2",
  "output_dir": "~/Music/suno-channels/jpop/260411_jpop-vol10",
  "songs": [
    {
      "order": 1,
      "title": "曲名1",
      "lyrics": "[Verse 1]\n朝焼けの空に...\n[Chorus]\nこの声が届くなら...",
      "vocal_gender": "female",
      "weirdness": 0.3,
      "style_influence": 0.8,
      "make_instrumental": false
    },
    {
      "order": 2,
      "title": "曲名2",
      "lyrics": "[Verse 1]\n...",
      "vocal_gender": "male",
      "weirdness": 0.2,
      "style_influence": 0.9,
      "make_instrumental": false
    }
  ]
}
```

**필드 설명:**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `source` | string (URL 또는 path) | 선택 | YouTube URL 또는 로컬 mp3 경로. 있으면 MIDI Cover 모드, 없으면 일반 생성 |
| `playlist` | string | 필수 | Suno workspace 이름 (예: "jpop-vol10"). 없으면 자동 생성 |
| `channel` | string | 필수 | "jpop" \| "cafe" \| "phonk" — 폴더 구조 및 DB 태깅에 사용 |
| `model` | string | 선택 | "chirp-fenix" (v5.5) 기본값. "chirp-v4-5" 선택 가능 |
| `style` | string | 필수 | 모든 곡에 공통 적용할 스타일 태그 (곡별 override 가능) |
| `negative_tags` | string | 선택 | 피할 태그들 |
| `soundfont` | string | 선택 | MIDI 렌더링용 SoundFont 경로 (기본: VintageDreamsWaves-v2.sf2) |
| `output_dir` | string | 필수 | 결과물 저장 경로 |
| `songs` | array | 필수 | 곡 설정 배열 |

**곡 설정 (songs 배열 항목):**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `order` | integer | 필수 | 트랙 번호 (01, 02, ...) |
| `title` | string | 필수 | 곡 제목 |
| `lyrics` | string | 필수 | 전체 가사 ([Verse][Chorus][Bridge] 구조) 또는 "[Instrumental]" |
| `vocal_gender` | string | 선택 | "female" \| "male" \| null (미지정) |
| `weirdness` | float | 선택 | 0.0~1.0 (Suno weirdness 파라미터, 기본: 0.5) |
| `style_influence` | float | 선택 | 0.0~1.0 (Cover 모드에서 소스 스타일 따를 정도, 기본: 0.8) |
| `make_instrumental` | boolean | 선택 | Instrumental 곡 생성 여부 (기본: false) |

### 9-3. Workspace 자동 관리

**작동 순서:**

```
1. GET /api/project/me?page=1&sort=max_created_at_last_updated_clip
   → workspace 목록 조회

2. JSON "playlist" 필드와 일치하는 workspace 검색
   → 있으면: workspace_id 사용
   → 없으면: step 3으로

3. POST /api/project
   {"action":"create","name":"jpop-vol10"}
   → 신규 workspace 생성 후 workspace_id 획득

4. 곡 생성 완료 후:
   POST /api/project/{workspace_id}/clips
   {"clip_ids":["id1","id2",...,"idN"]}
   → Workspace에 N곡 일괄 추가
```

**Python 구현 예시:**

```python
import requests

API_BASE = "https://studio-api-prod.suno.com"
HEADERS = {
    "Authorization": f"Bearer {__session_cookie}",
    "Content-Type": "application/json"
}

def get_or_create_workspace(playlist_name: str) -> str:
    """workspace 이름으로 조회 또는 생성. workspace_id 반환"""
    
    # 1. 기존 workspace 조회
    resp = requests.get(
        f"{API_BASE}/api/project/me?page=1&sort=max_created_at_last_updated_clip",
        headers=HEADERS
    )
    projects = resp.json().get("data", [])
    
    for proj in projects:
        if proj.get("name") == playlist_name:
            return proj.get("id")
    
    # 2. 없으면 신규 생성
    resp = requests.post(
        f"{API_BASE}/api/project",
        headers=HEADERS,
        json={"action": "create", "name": playlist_name}
    )
    new_proj = resp.json()
    return new_proj.get("id")

def add_clips_to_workspace(workspace_id: str, clip_ids: list[str]):
    """Workspace에 곡들 추가"""
    resp = requests.post(
        f"{API_BASE}/api/project/{workspace_id}/clips",
        headers=HEADERS,
        json={"clip_ids": clip_ids}
    )
    return resp.json()
```

### 9-4. 코드 추출 파이프라인 (source 있을 때)

source가 URL 또는 로컬 mp3 경로로 지정된 경우, 다음 단계를 순서대로 실행한다.

#### 9-4-1. YouTube 다운로드 (source가 URL)

```bash
yt-dlp -x --audio-format mp3 --audio-quality 0 \
  -o "~/workspace/original.%(ext)s" \
  "https://youtube.com/watch?v=xxx"
```

**결과**: `~/workspace/original.mp3`

#### 9-4-2. Demucs 보컬 분리 (MPS GPU 사용)

```bash
demucs -n htdemucs -d mps -o ~/workspace/demucs ~/workspace/original.mp3
```

**결과**:
```
~/workspace/demucs/htdemucs/original/
  ├── drums.wav
  ├── bass.wav
  ├── other.wav
  └── vocals.wav
```

#### 9-4-3. No-Vocal Mix 생성

```bash
ffmpeg -y -i ~/workspace/demucs/htdemucs/original/other.wav \
  -i ~/workspace/demucs/htdemucs/original/bass.wav \
  -i ~/workspace/demucs/htdemucs/original/drums.wav \
  -filter_complex "amix=inputs=3:duration=longest" \
  ~/workspace/no_vocal.wav
```

#### 9-4-4. Chordino 코드 추출 (step1a_midi_cover_research.md 참조)

**환경**: conda env `chord310` (Python 3.10)

```bash
# Chordino 실행 (roll_on=0.5, simplify chords)
cd ~/Projects/chordino  # 또는 chord_extractor 스크립트 경로
python extract_chords.py \
  --input ~/workspace/no_vocal.wav \
  --output ~/workspace/chords.json \
  --roll_on 0.5 \
  --simplify
```

**출력 형식** (`chords.json`):
```json
{
  "key": "A major",
  "bpm": 84,
  "chords": [
    {"root": "A", "type": "major", "start_beat": 0, "duration": 2},
    {"root": "F#", "type": "minor", "start_beat": 2, "duration": 2},
    {"root": "D", "type": "major", "start_beat": 4, "duration": 2},
    {"root": "E", "type": "major", "start_beat": 6, "duration": 2}
  ]
}
```

**Key Diatonic Filter (자동 적용):**
- Chordino 출력의 모든 코드 검증
- 24개 키(12 notes × major/minor) 각각에 대해 diatonic 코드 개수 계산
- **self-consistency 최고인 키 자동 선택** (검증됨: 93~100% Jaccard match with Logic Pro)
- 비 diatonic 코드 제거, 연속 동일 코드 병합

#### 9-4-5. 코드 → MIDI 변환

```python
import mido
from mido import MidiFile, MidiTrack, Message

def chords_to_midi(chords_json: dict, output_midi: str, bpm: int = 120):
    """chords.json → MIDI 파일 변환"""
    
    mid = MidiFile()
    track = MidiTrack()
    mid.tracks.append(track)
    
    # 템포 설정 (BPM)
    track.append(Message('program_change', program=0))
    
    # Note mapping: C=60, D=62, E=64, F=65, G=67, A=69, B=71
    NOTE_MAP = {'C': 60, 'D': 62, 'E': 64, 'F': 65, 'G': 67, 'A': 69, 'B': 71}
    OFFSET = {'#': 1, 'b': -1}
    
    chords = chords_json.get("chords", [])
    time_ms_per_beat = 60000 / bpm
    
    for chord in chords:
        root = chord.get("root")
        chord_type = chord.get("type")  # "major" | "minor" | "7" 등
        duration = chord.get("duration", 1)  # beat 단위
        
        # Root note MIDI pitch 계산
        root_pitch = NOTE_MAP.get(root[0])
        if len(root) > 1:
            root_pitch += OFFSET.get(root[1], 0)
        
        # Chord voicing (대삼화음: root, third, fifth)
        if chord_type == "major":
            pitches = [root_pitch, root_pitch + 4, root_pitch + 7]
        elif chord_type == "minor":
            pitches = [root_pitch, root_pitch + 3, root_pitch + 7]
        else:
            pitches = [root_pitch, root_pitch + 4, root_pitch + 7]  # fallback
        
        # MIDI ticks 계산 (480 ticks per beat)
        ticks = int(duration * 480)
        
        # Note On
        for pitch in pitches:
            track.append(Message('note_on', note=pitch, velocity=64, time=0))
        
        # Note Off
        for i, pitch in enumerate(pitches):
            track.append(Message('note_off', note=pitch, velocity=0, 
                                time=ticks if i == 0 else 0))
    
    mid.save(output_midi)
```

#### 9-4-6. MIDI → mp3 렌더링 (fluidsynth + ffmpeg)

```bash
# MIDI → WAV (fluidsynth)
fluidsynth -ni \
  -F ~/workspace/chords.wav \
  -O s16 \
  -T wav \
  ~/Music/soundfonts/FluidR3_GM.sf2 \
  ~/workspace/chords.mid

# WAV → mp3 (ffmpeg)
ffmpeg -y -i ~/workspace/chords.wav \
  -b:a 192k \
  ~/workspace/chords.mp3
```

**결과**: `~/workspace/chords.mp3` (Suno에 업로드할 오디오)

### 9-5. Suno 업로드 및 Cover Mode 곡 생성

#### 9-5-1. 오디오 업로드 (browser-token 필수)

```python
import base64
import json
import time
import requests

def upload_audio_to_suno(mp3_path: str, session_cookie: str) -> str:
    """
    mp3 파일을 Suno에 업로드하여 clip_id 획득.
    
    Returns: clip_id (초기 생성된 클립 ID)
    """
    
    # browser-token 생성 (base64 encoded timestamp)
    timestamp_ms = int(time.time() * 1000)
    token_payload = {"timestamp": timestamp_ms}
    browser_token = base64.b64encode(
        json.dumps(token_payload).encode()
    ).decode()
    
    # 1. initialize-clip 호출 (서버에 업로드 시작 신호)
    init_resp = requests.post(
        "https://studio-api-prod.suno.com/api/clips/initialize",
        headers={
            "Authorization": f"Bearer {session_cookie}",
            "X-Browser-Token": browser_token
        },
        json={
            "type": "upload",
            "file_name": "chords.mp3",
            "duration": 30,  # 예상 길이 (초)
        }
    )
    clip_data = init_resp.json()
    clip_id = clip_data.get("id")
    
    # 2. mp3 파일 업로드
    with open(mp3_path, "rb") as f:
        files = {"file": f}
        upload_resp = requests.post(
            "https://studio-api-prod.suno.com/api/clips/upload",
            headers={
                "Authorization": f"Bearer {session_cookie}",
                "X-Browser-Token": browser_token
            },
            files=files,
            data={"clip_id": clip_id}
        )
    
    # 3. Suno가 처리 완료 대기 (poll)
    max_attempts = 30
    for attempt in range(max_attempts):
        check_resp = requests.get(
            f"https://studio-api-prod.suno.com/api/clips/{clip_id}",
            headers={"Authorization": f"Bearer {session_cookie}"}
        )
        clip = check_resp.json()
        
        if clip.get("status") in ["ready", "complete"]:
            return clip_id
        
        time.sleep(2)
    
    raise RuntimeError(f"Upload timeout for {clip_id}")

# 사용 예시
clip_id = upload_audio_to_suno("~/workspace/chords.mp3", SUNO_COOKIE)
print(f"Upload successful: clip_id={clip_id}")
```

#### 9-5-2. Cover 모드 곡 생성 (source 있을 때)

```python
def generate_song_cover(
    song_config: dict,
    cover_clip_id: str,
    suno_api_base: str = "http://localhost:3001",
) -> list[dict]:
    """
    Cover 모드로 곡 생성. 2개 버전 반환.
    
    Args:
        song_config: {"title", "lyrics", "vocal_gender", "weirdness", 
                     "style_influence", "style", "negative_tags"}
        cover_clip_id: Suno 업로드 클립 ID
    
    Returns: [track1, track2] (각각 id, audio_url, image_url 포함)
    """
    
    payload = {
        "title": song_config.get("title"),
        "prompt": song_config.get("lyrics"),
        "tags": song_config.get("style"),
        "negative_tags": song_config.get("negative_tags", ""),
        "model": "chirp-fenix",
        "cover_clip_id": cover_clip_id,
        "is_remix": True,
        "audio_influence": 1.0,  # 소스 오디오 100% 유지
        "style_influence": song_config.get("style_influence", 0.8),
        "weirdness": song_config.get("weirdness", 0.5),
        "wait_audio": False,
    }
    
    # 부성 지정 (선택)
    if song_config.get("vocal_gender"):
        payload["vocal_gender"] = song_config["vocal_gender"]
    
    resp = requests.post(
        f"{suno_api_base}/api/custom_generate",
        json=payload
    )
    
    tracks = resp.json()
    return tracks
```

#### 9-5-3. 일반 모드 곡 생성 (source 없을 때)

```python
def generate_song_normal(
    song_config: dict,
    suno_api_base: str = "http://localhost:3001",
) -> list[dict]:
    """
    일반 생성 모드 (source 없음). 2개 버전 반환.
    """
    
    payload = {
        "title": song_config.get("title"),
        "prompt": song_config.get("lyrics"),
        "tags": song_config.get("style"),
        "negative_tags": song_config.get("negative_tags", ""),
        "model": "chirp-fenix",
        "make_instrumental": song_config.get("make_instrumental", False),
        "wait_audio": False,
    }
    
    if song_config.get("vocal_gender"):
        payload["vocal_gender"] = song_config["vocal_gender"]
    
    resp = requests.post(
        f"{suno_api_base}/api/custom_generate",
        json=payload
    )
    
    tracks = resp.json()
    return tracks
```

### 9-6. 생성 후 처리 및 DB 저장

각 곡이 생성되면 다음 단계를 순서대로 실행:

```python
import sqlite3
import os
from pathlib import Path

def process_generated_song(
    song_config: dict,
    track: dict,
    output_dir: str,
    channel: str,
    playlist_id: int,
) -> dict:
    """
    생성된 곡 1개 처리: 폴 → 다운로드 → DB 저장
    
    Args:
        song_config: {order, title, ...}
        track: Suno API 응답 (id, audio_url, image_url, duration 등)
        output_dir: 저장 경로
        channel: "jpop" | "cafe" | "phonk"
        playlist_id: DB playlist ID
    
    Returns: {mp3_path, jpeg_path, txt_path, suno_id}
    """
    
    suno_id = track.get("id")
    order = song_config.get("order")
    title = song_config.get("title")
    
    # 1. 폴 (최대 60초 대기)
    max_poll = 60
    for attempt in range(max_poll):
        resp = requests.get(f"http://localhost:3001/api/get?ids={suno_id}")
        current = resp.json()[0]
        
        if current.get("status") in ["streaming", "complete"]:
            audio_url = current.get("audio_url")
            image_url = current.get("image_url")
            duration = current.get("duration")
            lyric = current.get("lyric", "")
            break
        
        time.sleep(1)
    else:
        raise TimeoutError(f"Song {title} generation timeout")
    
    # 2. 파일 경로 생성
    os.makedirs(f"{output_dir}/01_songs", exist_ok=True)
    base_path = f"{output_dir}/01_songs/{order:02d}_{title}"
    mp3_path = f"{base_path}.mp3"
    jpeg_path = f"{base_path}.jpeg"
    txt_path = f"{base_path}.txt"
    
    # 3. mp3 다운로드
    mp3_data = requests.get(audio_url).content
    with open(mp3_path, "wb") as f:
        f.write(mp3_data)
    
    # 4. 커버아트 다운로드
    jpeg_data = requests.get(image_url).content
    with open(jpeg_path, "wb") as f:
        f.write(jpeg_data)
    
    # 5. 가사 저장 (API 응답의 lyric 사용 — 프롬프트 가사 아님!)
    with open(txt_path, "w", encoding="utf-8") as f:
        f.write(lyric)
    
    # 6. DB 저장
    db_path = os.path.expanduser("~/Projects/claude/test/niche_bending/gems.db")
    conn = sqlite3.connect(db_path)
    conn.execute("""
        INSERT INTO suno_tracks
        (id, title, lyric, style, cover_url, audio_url, duration,
         prompt_json, mp3_path, jpeg_path, txt_path, playlist_id,
         channel, status, created_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, datetime('now'))
    """, (
        suno_id,
        title,
        lyric,
        song_config.get("style"),
        image_url,
        audio_url,
        duration,
        json.dumps(song_config, ensure_ascii=False),
        mp3_path,
        jpeg_path,
        txt_path,
        playlist_id,
        channel,
        "downloaded",
    ))
    conn.commit()
    conn.close()
    
    return {
        "suno_id": suno_id,
        "mp3_path": mp3_path,
        "jpeg_path": jpeg_path,
        "txt_path": txt_path,
    }
```

### 9-7. EQ / 마스터링 후처리

생성된 곡에 장르별 EQ를 자동 적용한다. 상세 세팅은 `step2_audio_processing_guide.md` 섹션 3-5 참조.

```python
def apply_eq(mp3_path, channel, output_path):
    """장르별 EQ 자동 적용"""
    if channel in ["jpop", "jpop_emotional"]:
        # A: 보컬 EQ (Demucs 분리 → 보컬만 EQ → 합성)
        # HPF 100Hz, 250Hz -4dB, 800Hz -4dB, 3kHz +3dB,
        # 5.5kHz +5dB, 8kHz -2dB, LPF 16kHz, 컴프 3:1, afftdn
        apply_vocal_eq(mp3_path, output_path)
    elif channel in ["edm", "rock", "phonk"]:
        # B: 전체 마스터링 (Demucs 없이)
        # HPF 25Hz, 60Hz +2dB, 300Hz -3dB, 3.5kHz +3dB,
        # 5.5kHz +3dB, 10kHz+ +3dB shelf, 컴프 3:1, loudnorm -10
        apply_master_eq(mp3_path, output_path)
    else:
        # cafe/instrumental: 전체 마스터링 약하게
        apply_master_eq(mp3_path, output_path, gentle=True)
```

**JSON 설정에서 EQ 제어:**
```json
{
  "eq_mode": "auto",
  "eq_options": {
    "vocal_eq": true,
    "master_eq": false,
    "skip_eq": false
  }
}
```
- `"auto"` (기본): channel 기반 자동 선택
- `"vocal"`: 강제 보컬 EQ
- `"master"`: 강제 전체 마스터링
- `"skip"`: EQ 건너뛰기

### 9-8. Demucs 보컬 분리 (선택: step2 준비)

생성된 곡들의 vocal stem 추출:

```bash
demucs -n htdemucs -d mps -o {output_dir}/02_stems {output_dir}/01_songs/*.mp3
```

Whisper STT (step2)에서 가사 추출 시 이 vocals.wav를 입력으로 사용한다.
9-7에서 보컬 EQ 적용 시 이미 Demucs가 실행되므로, 중복 실행을 피한다.

### 9-8. 전체 통합 스크립트 사용법

#### 설정 파일 작성

```bash
# ~/Music/suno-channels/configs/jpop_cover.json 예시
cat > jpop_cover.json << 'EOF'
{
  "source": "https://youtube.com/watch?v=dQw4w9WgXcQ",
  "playlist": "jpop-vol10",
  "channel": "jpop",
  "model": "chirp-fenix",
  "style": "emotional J-pop, acoustic pop, bright piano, acoustic guitar, string ensemble, dramatic chord changes, cinematic atmosphere, analog warmth, uplifting chorus, 84 BPM",
  "negative_tags": "distortion, heavy metal",
  "soundfont": "~/Music/soundfonts/FluidR3_GM.sf2",
  "output_dir": "~/Music/suno-channels/jpop/260411_jpop-vol10",
  "songs": [
    {"order": 1, "title": "曲1", "lyrics": "[Verse 1]\n...", "vocal_gender": "female", "weirdness": 0.3, "style_influence": 0.8, "make_instrumental": false}
  ]
}
EOF
```

#### 스크립트 실행

```bash
# 표준 실행 (전체 파이프라인)
python3 ~/Projects/clones/zach-suno-api/scripts/generate_playlist.py jpop_cover.json

# 설정 검증만 (dry-run)
python3 ~/Projects/clones/zach-suno-api/scripts/generate_playlist.py jpop_cover.json --dry-run

# 특정 곡만 재생성 (예: 3번, 5번, 7번)
python3 ~/Projects/clones/zach-suno-api/scripts/generate_playlist.py jpop_cover.json --songs 3,5,7

# Prepare 단계만 (다운로드 + Demucs까지, 생성 안 함)
python3 ~/Projects/clones/zach-suno-api/scripts/generate_playlist.py jpop_cover.json --prepare-only

# 자세한 로그 출력
python3 ~/Projects/clones/zach-suno-api/scripts/generate_playlist.py jpop_cover.json --verbose
```

#### CLI 옵션

| 옵션 | 설명 |
|------|------|
| `--dry-run` | 설정 검증만 (실행 안 함) |
| `--songs 1,3,5` | 특정 곡만 생성 (DB는 업데이트하지 않음, 파일만 덮어씀) |
| `--prepare-only` | prepare 단계만 (YouTube 다운로드 + Demucs + Chordino) |
| `--skip-demucs` | Demucs 보컬 분리 스킵 (빠른 실행, step2는 수동 처리) |
| `--verbose` | 모든 중간 단계 로그 출력 |
| `--db-path PATH` | gems.db 경로 지정 (기본: ~/Projects/claude/test/niche_bending/gems.db) |

### 9-9. 스크립트 구조 및 파일 위치

```
~/Projects/clones/zach-suno-api/
├── scripts/
│   ├── generate_playlist.py          ← 메인 통합 스크립트 (이 섹션 구현 대상)
│   ├── midi_cover.py                 ← Chordino + Key filter 모듈
│   ├── suno_workspace_sync.py        ← 기존 workspace 동기화 (재사용)
│   └── config_examples/
│       ├── jpop_cover.json           ← Cover 모드 예시
│       ├── jpop_normal.json          ← 일반 생성 예시
│       ├── cafe_instrumental.json    ← Instrumental 예시
│       └── phonk_remix.json
├── .env                               ← Suno API 인증 정보
│   ├── SUNO_COOKIE=...
│   ├── CAPSOLVER_API_KEY=...
│   ├── BROWSER_LOCALE=en
│   └── DEFAULT_MODEL=chirp-fenix
└── requirements.txt
    ├── requests
    ├── yt-dlp
    ├── librosa
    ├── mido
    └── demucs (torch with MPS support)
```

### 9-10. 의존성

**Python 패키지** (시스템 Python 또는 venv):

```bash
pip install requests yt-dlp librosa mido sqlite3
```

**외부 프로그램** (Homebrew/시스템):

```bash
# macOS
brew install ffmpeg fluidsynth demucs

# 또는 conda (Demucs MPS 지원)
conda install -c conda-forge demucs pytorch::pytorch-macos
```

**Conda env (Chordino용)**:

```bash
# step1a_midi_cover_research.md에서 설정
conda activate chord310
python extract_chords.py ...
```

**API 인증** (필수):

- Suno API 서버: `http://localhost:3001` (zach-suno-api 실행)
- `.env` 파일: `SUNO_COOKIE`, `CAPSOLVER_API_KEY` 설정
- `__session` 쿠키 만료 시 갱신 필요

### 9-11. 핵심 주의사항

| 항목 | 주의 사항 |
|------|---------|
| **쿠키 만료** | `__session` JWT 토큰 ~1시간 만료. 장시간 작업 시 .env 갱신 필수 |
| **크레딧** | 곡 1개 생성 = ~10 크레딧. N곡 × 2버전 = N×20 크레딧 소비 |
| **browser-token** | upload 시 base64({"timestamp": milliseconds}) 필수 |
| **API 도메인** | Suno internal API: `https://studio-api-prod.suno.com` (zach-suno-api와 다름) |
| **가사 저장** | API 응답의 `lyric` 필드 사용 필수. 프롬프트 가사 사용 금지 |
| **Workspace 동기화** | 곡 생성 후 workspace에 추가해야 Suno 웹에서 관리/편집 가능 |
| **Demucs GPU** | `-d mps` 옵션으로 Apple Silicon GPU 사용 (CPU 대비 3~5배 빠름) |
| **Chordino 환경** | `conda activate chord310` (Python 3.10) 필수. 다른 env에서 실행 불가 |
| **Key Filter** | 24 key 전수 탐색 후 self-consistency 최고인 키 자동 선택. 검증됨: 93~100% Jaccard match |
| **mp3 파일명** | `{order:02d}_{title}.mp3` 형식 (Remotion 렌더링 시 정렬 기준) |

### 9-12. 트러블슈팅

**문제: "upload timeout for clip_id"**
- 원인: Suno 서버 느림 또는 mp3 파일 손상
- 해결: max_attempts를 60으로 증가, mp3 파일 재생 확인

**문제: "Chordino not found"**
- 원인: conda env 활성화 안 됨 또는 경로 오류
- 해결: `conda activate chord310` 확인, extract_chords.py 경로 확인

**문제: "401 Unauthorized"**
- 원인: `__session` 쿠키 만료 또는 잘못된 형식
- 해결: `.env` SUNO_COOKIE 갱신 (Chrome DevTools에서 복사)

**문제: "workspace not found after creation"**
- 원인: GET /api/project 조회 지연
- 해결: get_or_create_workspace 함수에 0.5초 sleep 추가

---

## 10. 다음 단계

생성 완료 후 `step2_audio_processing_guide.md`로 이동:
- Demucs 보컬 분리 (MPS)
- Whisper STT (medium)
- 가사 타임스탬프 매칭
- 번역 + SRT 생성
