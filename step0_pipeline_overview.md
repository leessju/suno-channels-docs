# 전체 파이프라인 마스터 문서

> Suno AI 음악 → YouTube 업로드까지 전체 자동화 파이프라인.
> 다른 세션에서 이 문서만 읽으면 전체 흐름 이해 가능.

---

## 0. 한눈에 보는 파이프라인

```
┌──────────────────────────────────────────────────────────┐
│ STAGE 1: Suno 음악 생성                                   │
│  입력: 프롬프트 (장르, 무드, 가사, BPM 등)                 │
│  출력: mp3 + jpeg + txt + suno_id(v1+v2) + metadata       │
│  도구: zach-suno-api (Playwright + 2Captcha 자동)         │
│  문서: suno_generation_guide.md                           │
└──────────────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────────────┐
│ STAGE 2: 오디오 후처리                                     │
│  입력: mp3                                                 │
│  출력: vocals.wav + lyrics.json + translated.json + srt   │
│  도구: Demucs (MPS), Whisper (medium), Vertex Gemini       │
│  문서: audio_processing_guide.md                           │
└──────────────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────────────┐
│ STAGE 3: 영상 제작                                         │
│  입력: mp3 + jpeg + lyrics.json                            │
│  출력: 곡별 mp4 (가로) + 풀영상 + 쇼츠 (세로)              │
│  도구: Remotion 4.0, ffmpeg                                │
│  문서: video_production_guide.md                           │
└──────────────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────────────┐
│ STAGE 4: 배경이미지 할당 + 썸네일 제작                     │
│  입력: assets/back_image/{genre}/ 이미지 풀                │
│  출력: 01_songs/ 배경이미지 교체 + 1280x720 PNG 썸네일     │
│  도구: assign_background_images.py, gen_vol_thumbnails.py  │
│  문서: thumbnail_guide.md                                  │
└──────────────────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────────────┐
│ STAGE 5: YouTube 업로드                                   │
│  입력: 풀영상 + 쇼츠 + 썸네일 + youtube_meta.json          │
│  출력: 영상 URL + 댓글 (풀영상 유입)                       │
│  도구: browser-use (업로드) + YouTube API (메타/댓글)      │
│  문서: youtube_upload_guide.md, shorts_guide.md            │
└──────────────────────────────────────────────────────────┘
```

---

## 1. 표준 폴더 구조

```
~/Music/suno-channels/
├── docs/                          ← 가이드 문서 (이 폴더)
│   ├── pipeline_overview.md       ← 이 문서 (마스터)
│   ├── suno_generation_guide.md
│   ├── audio_processing_guide.md
│   ├── video_production_guide.md
│   ├── thumbnail_guide.md
│   ├── shorts_guide.md
│   ├── youtube_upload_guide.md
│   └── logs/                      ← 작업 이력
├── assets/
│   ├── fonts/                     ← 한글/일본어 폰트
│   └── back_image/                ← 장르별 배경 이미지 풀
│       ├── jpop/                  ← j_001~j_NNN.png
│       │   └── cover/             ← 1번곡 전용 (썸네일 겸용)
│       └── phonk/                 ← p_001~p_NNN.png
│           └── cover/             ← 1번곡 전용
├── jpop/                          ← 季節のプレイリスト 채널
│   └── {YYMMDD}_{playlist-name}/  ← 플레이리스트 폴더
│       ├── 01_songs/              ← mp3 + jpeg + lyrics.json + txt
│       ├── 02_videos/             ← Remotion 렌더 (가로 1920x1080)
│       ├── 03_subtitles/          ← SRT 다국어 자막
│       ├── 04_final/              ← 머지 풀영상 + silence + concat_list
│       ├── 05_shorts/              ← 쇼츠 (세로 1080x1920)
│       ├── thumbnails/             ← 최종 썸네일 5장
│       ├── tracklist.txt           ← YouTube 설명 타임스탬프
│       └── youtube_meta.json       ← 제목/설명/태그
├── cafe/                          ← Hazy Roast 채널
├── phonk/                         ← ZERO MERCY BEATS 채널
└── _raw/                          ← 원본, 리서치, 후보 자료
    ├── design_research/
    └── thumbnail_research/
```

### 파일명 규칙

| 파일 | 규칙 | 예시 |
|------|------|------|
| 플레이리스트 폴더 | `{YYMMDD}_{name}` | `260402_hero-vol1` |
| 곡 파일 | `{번호}_{곡명}.{확장자}` | `01_ヒーローになれなくても.mp3` |
| 풀영상 | `{playlist_name}_full.mp4` | `hero-vol1_full.mp4` |
| 쇼츠 | `{번호}_{곡명}_shorts.mp4` | `01_ヒーローになれなくても_shorts.mp4` |
| 썸네일 | `{playlist_name}.png` (1280x720) | `hero-vol1.png` |

---

## 2. 각 스테이지 핵심 규칙

### STAGE 1: Suno
- **모든 데이터 저장** (mp3, jpeg, txt, suno_id v1+v2, metadata)
- 가사는 Suno API `get?ids={id}` → `lyric` 필드 (프롬프트 가사 아님)
- BROWSER_LOCALE=en 필수 (ko-KR은 캡차 오류)
- v4.5로 느낌 잡고 Cover(Audio Influence 100%)로 v5.5 업그레이드
- **MIDI Cover 파이프라인**: BTC-ISMIR19 (코드 추출) → fluidsynth (MIDI→MP3) → upload_audio → cover_clip_id
  - 도구: BTC Transformer (`~/Projects/clones/BTC-ISMIR19/`, `conda activate chord310`)
  - Chordino 후순위 보류 (BTC 사용 불가 시 fallback; BTC 대비 Jaccard 33% vs 69.2%)

### STAGE 2: 오디오
- Demucs는 **MPS(Apple GPU) 필수** (`-d mps`) — CPU 대비 3배 빠름
- Whisper는 **medium** 모델 (큰 모델은 일본어 정확도 떨어질 수 있음)
- 가사는 4중 검증: txt 순서 / 중복 / 겹침 / words
- 번역은 Vertex AI Gemini (account3 크레딧)

### STAGE 3: 영상
- **nohup 필수** (프로세스 종료 시 moov atom 에러)
- 곡 간 **1.5초 무음 갭** 필수
- 재정렬 시 **4종 세트(mp3, jpeg, lyrics.json, txt) 함께 이동**
- 병렬 쇼츠 생성 시 **tmp 경로 곡별 분리** (파일 충돌 방지)
- render.sh의 **SAFE_STEM** 주의 (일본어 → MD5 해시 추가)

### STAGE 4: 썸네일
- 배경은 **애니 스타일** (J-POP은 Makoto Shinkai / 레트로 90s 추천)
- 인물은 반드시 **"anime girl"** 명시 (성별 중립 프롬프트 금지)
- 폰트는 **Black Han Sans** (한글 임팩트 최대)
- **후킹 텍스트만** (설명형 금지): "이 노래 왜 아무도 안 알려줬어?"
- 보라색 금지 (AI 의심)

### STAGE 5: 업로드
- 영상 파일 업로드: **browser-use** (quota 0)
- 제목/설명/공개설정: **YouTube API** (browser-use 텍스트 입력 불안정)
- 댓글: **YouTube API** (Polymer submit 문제 회피)
- 댓글은 **Public 영상에만** 가능
- 제목은 **검색 최적화형**, 후킹은 썸네일에 맡김

---

## 3. 자동화 함수 시그니처 (전체 파이프라인)

```python
# STAGE 1
def generate_songs(
    prompts: list[dict],           # 곡별 프롬프트 (title, style, lyrics, bpm)
    playlist_folder: str,           # 출력 폴더
    model: str = "chirp-fenix",     # Suno 모델
    save_all: bool = True,          # suno_id v1+v2 + metadata 저장
) -> list[dict]:                    # [{suno_id, mp3, jpeg, txt, meta}]
    ...

# STAGE 2
def process_audio(
    playlist_folder: str,
    demucs_device: str = "mps",     # MPS GPU
    whisper_model: str = "medium",  # Whisper STT 모델
    translate_langs: list[str] = ["ko", "en"],  # 번역 언어
) -> dict:                          # {lyrics_json, srt_files, vocals_wav}
    ...

# STAGE 3
def render_videos(
    playlist_folder: str,
    render_full: bool = True,       # 플레이리스트 머지
    render_shorts: bool = True,     # 쇼츠 생성
    max_workers: int = 2,           # 병렬 수 (렌더)
) -> dict:                          # {individual_mp4s, full_mp4, shorts_mp4s}
    ...

# STAGE 4
def generate_thumbnail(
    playlist_folder: str,
    channel: str,                   # jpop/cafe/phonk
    title_line1: str, title_line2: str,
    bg_image: str = None,           # None이면 Imagen으로 자동 생성
) -> str:                           # 썸네일 파일 경로
    ...

# STAGE 5
def upload_to_youtube(
    playlist_folder: str,
    channel: str,
    privacy: str = "public",
    upload_shorts: bool = True,
    post_comments: bool = True,     # 쇼츠에 풀영상 링크 댓글
) -> dict:                          # {full_url, shorts_urls, comments}
    ...

# 전체 파이프라인
def run_full_pipeline(
    playlist_config: dict,          # {name, channel, prompts, thumbnail_config}
) -> dict:                          # 전체 실행 결과
    songs = generate_songs(...)
    audio = process_audio(...)
    videos = render_videos(...)
    thumb = generate_thumbnail(...)
    uploads = upload_to_youtube(...)
    return {"songs": songs, "videos": videos, "uploads": uploads}
```

---

## 4. 채널별 프로필

| 채널 | 장르 | 스타일 | Suno 프롬프트 | 15분+ |
|------|------|--------|--------------|:---:|
| **季節のプレイリスト** (jpop) | 감성 J-POP, 락발라드 | emotional rock ballad, acoustic pop | bright piano, acoustic guitar, string ensemble | ✅ |
| **Hazy Roast** (cafe) | 카페, lo-fi | instrumental jazz, lo-fi hip hop | warm vinyl, soft piano, chill beats | ❌ |
| **ZERO MERCY BEATS** (phonk) | Phonk, EDM | brazilian phonk, hard bass | aggressive 808, distortion, cowbell | ❌ |

### 채널별 썸네일 가이드
| 채널 | 배경 스타일 | 강조색 | 후킹 톤 |
|------|------------|--------|---------|
| jpop | 레트로 애니 / Makoto Shinkai | 골드 (#FFD700) | 벅차오르는, 소름, 울컥 |
| cafe | 로파이 애니, 따뜻한 실내 | 카페 브라운 (#D4A574) | 아늑한, 편안한, 몰입 |
| phonk | 사이버펑크, 네온 | 시안 (#00FFFF) | 강렬, 미침, 떡상 + 💀 |

---

## 5. 자원/비용

| 자원 | 용도 | 비용 | 만료 |
|------|------|------|------|
| GCP account3 | Imagen, Vertex AI Gemini | ₩385K+ | 2026-06-27 |
| Suno 크레딧 | 음악 생성 | - | - |
| 2Captcha | Suno 캡차 자동 해결 | $$ | - |
| YouTube API | 메타 업데이트, 댓글, 자막 | 10,000 units/일 | - |

### YouTube API quota 단가
| 작업 | 비용 | 일일 최대 |
|------|:---:|:---:|
| videos.update | 50 | 200회 |
| commentThreads.insert | 50 | 200회 |
| captions.insert | 400 | 25회 |
| videos.insert (업로드) | 1,600 | 6회 |

**→ 업로드는 무조건 browser-use, API는 메타/댓글/자막에만 사용**

---

## 6. 인프라 위치

```
~/Projects/clones/
├── zach-suno-api/              ← Suno API 자동화
├── suno-video/                 ← Remotion 영상 파이프라인
│   ├── scripts/render.sh       ← 렌더 스크립트
│   ├── scripts/render_playlist.sh  ← 플레이리스트 전체 렌더
│   └── src/                    ← React/Remotion 코드
└── ismir25-ai-music-detector/  ← AI 음악 감지 (참고용)

~/.claude/
├── youtube_token_jpop.json     ← 季節のプレイリスト OAuth
├── youtube_token_cafe.json     ← Hazy Roast OAuth
├── youtube_token_phonk.json    ← ZERO MERCY BEATS OAuth
└── gems/gemini_keys.json       ← Gemini API 키 (무료 11개)

~/Projects/claude/test/niche_bending/
└── infra/credentials_account3.json  ← Vertex AI 인증
```

---

## 7. DB 스키마 (gems.db)

```
suno_tracks
  - id (suno_id)
  - v2_id
  - title
  - lyric
  - style
  - cover_url
  - mp3_path
  - created_at
  - playlist_id

suno_playlists
  - id
  - name
  - channel
  - folder_path
  - track_count
  - total_duration
  - status

back_image_usage                   ← 배경이미지 사용 횟수 추적
  - image_path (UNIQUE)            ← assets/back_image/ 기준 상대 경로
  - genre
  - image_type                     ← "cover" | "general"
  - use_count
  - updated_at

vol_song_images                    ← vol별 곡↔배경이미지 매핑
  - vol_name                       ← 예: "260402_hero-vol1"
  - song_filename                  ← 예: "01_ヒーローになれなくても.png"
  - image_path                     ← assets/back_image/ 기준 상대 경로
  - assigned_at
  - UNIQUE(vol_name, song_filename)
```

---

## 8. 실행 순서 (표준)

```bash
# 1. 새 플레이리스트 폴더 생성
mkdir -p ~/Music/suno-channels/jpop/260501_new-playlist/{01_songs,02_videos,03_subtitles,04_final,05_shorts,thumbnails}

# 2. Suno 음악 생성
python ~/Projects/clones/zach-suno-api/generate_playlist.py --playlist new-playlist

# 3. Remotion 렌더
bash ~/Projects/clones/suno-video/scripts/render_playlist.sh ~/Music/suno-channels/jpop/260501_new-playlist

# 4. 플레이리스트 머지
bash /tmp/merge.sh new-playlist

# 5. 쇼츠 생성
python /tmp/gen_shorts.py new-playlist

# 6. 배경 이미지 할당 + 썸네일 생성
python3 scripts/assign_background_images.py --vol 260501_new-playlist
python3 scripts/gen_vol_thumbnails.py --vol 260501_new-playlist

# 7. YouTube 업로드 (풀영상 + 쇼츠)
python /tmp/upload_yt.py new-playlist

# 8. 댓글 (Public 전환 후)
python /tmp/post_comments.py new-playlist
```

---

## 9. 작업별 소요 시간 추정

| 작업 | 20곡 기준 소요 시간 |
|------|------------------|
| Suno 생성 (v1+v2 40곡) | 30~60분 |
| Demucs 분리 (MPS) | 7분 (20초/곡) |
| Whisper STT | 10분 |
| 번역 (Gemini) | 5분 |
| Remotion 렌더 (2병렬) | 60~90분 |
| 플레이리스트 머지 | 15~25분 |
| 쇼츠 생성 (4병렬) | 10분 |
| 썸네일 생성 | 2분 |
| YouTube 풀영상 업로드 | 10~15분/개 |
| YouTube 쇼츠 업로드 | 1~2분/개 × 20 = 20~40분 |
| API 메타 업데이트 | 30초 |
| 댓글 달기 | 30초/곡 × 20 = 10분 |

**플레이리스트 1개 전체 파이프라인: ~4~6시간**

---

## 10. 다음 작업 참조 순서

1. **Suno 음악 생성** → `suno_generation_guide.md`
2. **오디오 후처리** → `audio_processing_guide.md`
3. **영상 제작** → `video_production_guide.md`
4. **썸네일 제작** → `thumbnail_guide.md`
5. **업로드** → `youtube_upload_guide.md`, `shorts_guide.md`

각 문서는 독립적으로 실행 가능하도록 작성됨. 전체 맥락이 필요하면 이 문서로 돌아옴.
