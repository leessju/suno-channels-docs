# Shorts 제작 가이드

> 모든 채널/플레이리스트에 범용 적용. 향후 자동화 파이프라인의 입력 스펙.

---

## 1. 출력 스펙

| 항목 | 값 |
|------|-----|
| 해상도 | **1080 × 1920 px** (9:16 세로) |
| 코덱 | H.264 (video) + AAC (audio) |
| FPS | 25 |
| 오디오 | 48000Hz, 2ch 스테레오 |
| 길이 | **~45~55초** (코러스 구간 자동 감지, Pre-Chorus 포함 시 늘어남) |
| 파일 크기 | ~8MB/개 |
| 포맷 | MP4 |
| 파일명 | `{번호}_{곡명}_shorts.mp4` |
| 저장 위치 | `{playlist_folder}/05_shorts/` |

---

## 2. 폴더 구조

```
{playlist_folder}/
├── 01_songs/          ← 입력: mp3 + jpeg + lyrics.json + txt
├── 02_videos/         ← Remotion 렌더 결과 (1920x1080 가로)
├── 03_subtitles/      ← 다국어 SRT
├── 04_final/          ← concat 머지 최종 영상
├── 05_shorts/         ← 쇼츠 영상 (1080x1920 세로)
└── tracklist.txt      ← YouTube 설명란 타임스탬프
```

### 입력 파일 (01_songs/ 내 4종 세트)
| 파일 | 필수 | 용도 |
|------|------|------|
| `{번호}_{곡명}.mp3` | ✅ | 오디오 소스 |
| `{번호}_{곡명}.jpeg` | ✅ | 커버아트 (Suno 생성) |
| `{번호}_{곡명}.lyrics.json` | ✅ | Whisper STT 타임스탬프 (코러스 감지용) |
| `{번호}_{곡명}.txt` | ⬜ | 가사 텍스트 (참고용) |

---

## 3. 쇼츠 생성 파이프라인

### 전체 흐름
```
입력: 02_videos/{곡명}.mp4 (1920x1080 가로 영상)
  ↓
1. 하이라이트 구간 선택 (~40초)
  ↓
2. 가로 → 세로 변환 (블러 배경 + 중앙 원본)
  ↓
3. 곡명 + 아티스트 텍스트 오버레이
  ↓
출력: 05_shorts/{번호}_{곡명}_shorts.mp4 (1080x1920 세로)
```

### 3-1. 하이라이트 구간 선택

우선순위:
1. `lyrics.json`에서 코러스 구간 자동 감지 (가사 반복 패턴)
2. `_energy_analysis.json` (librosa)에서 에너지 피크 구간
3. 폴백: 곡 전체 길이의 40~70% 구간 (대부분 코러스 위치)

```python
def find_highlight(lyrics_json_path, duration, target_length=40):
    """코러스/하이라이트 구간 찾기"""
    # lyrics.json에서 가사 라인 로드
    # 반복되는 가사 패턴 → 코러스로 판단
    # 에너지 분석 데이터 있으면 피크 구간 우선
    # 없으면 duration * 0.4 ~ 0.7 구간
    start = duration * 0.4
    return start, start + target_length
```

### 3-2. 가로→세로 변환 (ffmpeg)

```bash
ffmpeg -i {input_video} \
  -ss {start_time} -t {duration} \
  -filter_complex "
    [0:v]scale=1080:1920:force_original_aspect_ratio=increase,
         crop=1080:1920,boxblur=20:5[bg];
    [0:v]scale=1080:-1[fg];
    [bg][fg]overlay=(W-w)/2:(H-h)/2
  " \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  {output_shorts}
```

**원리:**
1. 원본을 세로 비율로 확대 + 크롭 → 블러 처리 → 배경
2. 원본을 세로 폭(1080px)에 맞게 축소 → 전경
3. 전경을 배경 중앙에 합성

### 3-3. 텍스트 오버레이 (선택)

```bash
# ffmpeg drawtext 또는 Pillow로 후처리
# 상단: 곡명 (일본어)
# 하단: 채널명 또는 "Full playlist in description"
```

---

## 4. 병렬 생성 규칙

### 안전한 병렬 실행
```python
from concurrent.futures import ProcessPoolExecutor

def generate_short(song_info):
    # 곡별 독립 tmp 디렉터리 사용
    tmp_dir = f"/tmp/shorts_{song_info['hash']}"
    os.makedirs(tmp_dir, exist_ok=True)
    # ... ffmpeg 실행 ...
    shutil.rmtree(tmp_dir)

with ProcessPoolExecutor(max_workers=4) as executor:
    executor.map(generate_short, songs)
```

### 주의사항
| 문제 | 원인 | 해결 |
|------|------|------|
| **tmp 파일 충돌** | 병렬 시 같은 /tmp/frame.jpg 사용 | **곡별 tmp 디렉터리 분리** |
| **lyrics.json 불일치** | mp3 파일명과 json 파일명 다름 | 번호 기준 매칭 또는 fuzzy match |
| **메모리 부족** | 동시 ffmpeg 4개 이상 | max_workers=4 제한 |

---

## 5. YouTube 업로드

### 방법 선택 기준

| 방법 | 채널 유형 | quota | 안정성 |
|------|----------|-------|--------|
| **browser-use** | 기본 채널 | 0 | 중간 |
| **YouTube API** | 브랜드 채널 | 1,600/건 | 높음 |

- **쇼츠 업로드는 browser-use 우선** (quota 절약)
- 브랜드 채널(Hazy Roast, ZERO MERCY BEATS)은 browser-use로 채널 전환 불가 → **API 필수**

### browser-use 업로드 절차

```python
# 1. YouTube Studio 이동
bu("go", "https://studio.youtube.com")

# 2. Upload 버튼 클릭
bu("click", "Upload videos")

# 3. 파일 선택 (input[type=file])
bu("js", f'document.querySelector("input[type=file]").value = "{filepath}"')

# 4. 제목/설명 입력

# 5. Shorts로 게시

# 6. Close 다이얼로그 (핵심 선택자)
bu("js", """
  Array.from(document.querySelectorAll('button'))
    .find(b => b.getAttribute('aria-label') === 'Close' && b.offsetParent !== null)
    ?.click()
""")
```

### 업로드 결과 저장
```python
# 업로드 완료 후 video_id 수집
# 저장: {playlist_folder}/05_shorts/upload_result.json
{
  "shorts_video_id": "filename",
  "Pa_CUwaXahQ": "01_名前のない感情_shorts.mp4",
  ...
}
```

---

## 6. 댓글 달기

### 확정 방식: YouTube Data API

browser-use 댓글은 불안정 (포커스 이탈, Polymer submit 문제). **항상 API 사용.**

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

def post_comment(token_path, video_id, comment_text):
    creds = Credentials.from_authorized_user_file(token_path)
    youtube = build("youtube", "v3", credentials=creds)
    youtube.commentThreads().insert(
        part="snippet",
        body={
            "snippet": {
                "videoId": video_id,
                "topLevelComment": {
                    "snippet": {
                        "textOriginal": comment_text
                    }
                }
            }
        }
    ).execute()
```

### 댓글 템플릿
```
{풀영상_제목}, 모았습니다
🎵아래 플레이리스트로 끝까지 들어보세요. 감사합니다.
👉 https://youtu.be/{풀영상_ID}
```

### quota 소비
| 작업 | 단위 비용 | 20곡 | 100곡 |
|------|----------|------|-------|
| commentThreads.insert | 50 units | 1,000 | 5,000 |
| **일일 한도** | **10,000 units** | | |

### 댓글 결과 저장
```
{playlist_folder}/05_shorts/comment_result.json
```

---

## 7. 쇼츠 제목/설명 규칙

### 제목
```
{곡명}
```
- 곡명 그대로 (일본어/한국어/영어)
- 해시태그 불필요 (YouTube 자동 #Shorts)
- 후킹 필요 없음 (쇼츠는 피드 노출, 제목보다 첫 3초가 중요)

### 설명
```
{곡명} - {채널명}

🎵 풀 플레이리스트:
👉 https://youtu.be/{풀영상_ID}

#{장르} #감성 #플레이리스트
```

### 채널별 해시태그
| 채널 | 해시태그 |
|------|---------|
| jpop | `#JPOP #감성 #プレイリスト` |
| cafe | `#CafeMusic #Lofi #카페음악` |
| phonk | `#Phonk #Bass #운동음악` |

---

## 8. 전체 쇼츠 파이프라인 (자동화용)

```
입력:
  - playlist_folder: 플레이리스트 폴더
  - channel: jpop | cafe | phonk
  - full_video_id: 풀영상 YouTube ID
  - full_video_title: 풀영상 제목 (댓글용)
  - token_path: YouTube OAuth 토큰 경로

처리:
  1. 02_videos/ 에서 개별 곡 영상 목록 로드
  2. 각 곡별 하이라이트 구간 선택
  3. 가로→세로 변환 (병렬, tmp 경로 분리)
  4. 05_shorts/ 에 저장
  5. browser-use 또는 API로 업로드
  6. upload_result.json 저장
  7. API로 각 쇼츠에 댓글 달기
  8. comment_result.json 저장

출력:
  - 05_shorts/*.mp4 (쇼츠 영상)
  - 05_shorts/upload_result.json (업로드 ID 매핑)
  - 05_shorts/comment_result.json (댓글 완료 목록)
```

### 자동화 함수 시그니처 (향후 구현)
```python
def generate_shorts(
    playlist_folder: str,
    channel: str,                # "jpop" | "cafe" | "phonk"
    highlight_method: str = "auto",  # "auto" | "chorus" | "energy" | "middle"
    target_length: int = 40,     # 초
    max_workers: int = 4,        # 병렬 수
) -> list[str]:  # 생성된 파일 경로 리스트
    ...

def upload_shorts(
    playlist_folder: str,
    channel: str,
    method: str = "browser-use",  # "browser-use" | "api"
    token_path: str = None,       # API 사용 시
) -> dict:  # {video_id: filename}
    ...

def comment_shorts(
    playlist_folder: str,
    full_video_id: str,
    full_video_title: str,
    token_path: str,
) -> dict:  # {video_id: "commented"}
    ...
```

---

## 9. 풀영상 머지 참고

쇼츠와 함께 풀영상도 필요하므로 머지 규칙도 기록.

### ffmpeg concat
```bash
# concat_list.txt 생성
for f in 02_videos/*.mp4; do
  echo "file '$f'" >> concat_list.txt
  echo "file 'silence_1.5s.mp4'" >> concat_list.txt
done

# 머지 (nohup 필수 — 프로세스 종료 방지)
nohup ffmpeg -f concat -safe 0 -i concat_list.txt \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 192k \
  04_final/{playlist_name}.mp4 &
```

### 머지 규칙
| 규칙 | 값 |
|------|-----|
| 곡 간 무음 갭 | **1.5초** (silence_1.5s.mp4) |
| nohup 필수 | 에이전트 종료 시 moov atom not found 방지 |
| 곡 재정렬 시 | **4종 세트 함께 이동** (mp3, jpeg, lyrics.json, txt) |

### silence 파일 생성
```bash
ffmpeg -f lavfi -i anullsrc=r=48000:cl=stereo -t 1.5 \
  -f lavfi -i color=c=black:s=1920x1080:r=30 -t 1.5 \
  -c:v libx264 -c:a aac silence_1.5s.mp4
```

---

## 10. YouTube 채널 정보

| 채널 | ID | 토큰 | 15분+ | 업로드 방식 |
|------|-----|------|-------|-----------|
| 季節のプレイリスト | UC2ofS0Y6ynIjRSVwexUUWQg | `~/.claude/youtube_token_jpop.json` | ✅ | browser-use OK |
| Lucid White | UCoHpJmMju00FPBogQr6D_Kw | `~/.claude/youtube_token_lucidwhite.json` | ✅ | browser-use OK |
| Hazy Roast | UCSvzzpXpaXwRWi3G-grk_7A | `~/.claude/youtube_token_cafe.json` | ❌ 미인증 | API 필수 |
| ZERO MERCY BEATS | UC0OSrx55lFo7ELCMo88mxUw | `~/.claude/youtube_token_phonk.json` | ❌ 미인증 | API 필수 |

### 15분 초과 영상 인증
- YouTube Studio → Settings → Channel → Feature eligibility → Intermediate features
- 전화번호 인증 필요 (채널별 1회)
- **미인증 채널에 15분 초과 영상 업로드 시 "Video is too long" 에러**

---

---

## 11. Lucid White 채널 전용 쇼츠 제작

> 스크립트: `scripts/gen_shorts_lucidwhite.py`
> 스타일: **Glassmorphism + Non-no 감성** (밝은 화이트 톤, 세련된 일본어 폰트)

### 11-1. 레이아웃 구조

```
┌─────────────────────────────┐  ← 0px
│   상단 영역 (TOP_H = 656px) │
│                              │
│   [영어 제목] (Georgia Italic)│
│   ─────────── (영어 폭과 동일) │
│   [일어 제목] (凸版文久見出し) │
│                              │
├─────────────────────────────┤  ← 656px (VIDEO_Y)
│                              │
│   원본 영상 (1080 × 607px)  │  ← 1080 * 9/16 = 607.5
│   scale=1080:607             │
│                              │
├─────────────────────────────┤  ← 1263px (BOT_Y)
│   하단 영역 (BOT_H = 657px) │
│                              │
│  ┌──────────────────────┐   │  ← Glassmorphism 박스
│  │   Lucid White        │   │
│  │  Synced Lyrics ❘ ... │   │
│  └──────────────────────┘   │
└─────────────────────────────┘  ← 1920px
```

### 11-2. 폰트 스펙

| 역할 | 폰트 | 경로 | 크기 |
|------|------|------|------|
| Latin 제목 (영어) | **Georgia Italic** | `/System/Library/Fonts/Supplemental/Georgia Italic.ttf` | 48pt |
| Japanese 제목 (일어) | **凸版文久見出し明朝 ExtraBold** | `/System/Library/AssetsV2/.../ToppanBunkyuMidashiMinchoStdN-ExtraBold.otf` | 50pt |
| 브랜드/UI 텍스트 | **Pretendard Medium** | `/tmp/pretendard/public/static/Pretendard-Medium.otf` | 26pt |

> ⚠️ **PIL(Pillow)은 폰트 fallback 없음** — 한 폰트로 한자/특수문자 혼합 불가.
> Latin과 Japanese 파트를 분리해서 각각 다른 폰트로 렌더링해야 함.

### 11-3. 특수문자 sanitize 규칙

PIL은 유니코드 특수문자(`❘`, `⊹` 등)를 지원하는 폰트가 없어 □(두부)로 렌더됨.

```python
def _sanitize_for_render(text):
    return text.replace('❘', '|').replace('⊹', '✦')
```

> `✦`도 Toppan Bunkyu에서 지원 안 됨 → 해당 기호가 포함된 라인은 렌더에서 제외.

### 11-4. 상단 제목 렌더링 로직

`TOP_TEXT[0]` = `"Melody Beyond Glass ❘ 硝子の旋律"` (youtube_meta.json title에서 `⊹` 기준 앞부분)

`❘` 를 기준으로 영어/일어 분리 → 3줄 렌더:

```python
# ❘ 기준 분리
lat_part, jp_part = raw_title.split('❘', 1)

# 영어 (Georgia Italic) — 위
draw_shadow(draw, (center_x - tw_lat/2, lat_y), lat_part, font_lat, WHITE)

# 밑줄 — 영어 텍스트 폭과 정확히 동일
draw.rectangle([(center_x - tw_lat/2, line_y), (center_x + tw_lat/2, line_y+1)], ...)

# 일어 (凸版文久見出し明朝) — 아래
draw_shadow(draw, (center_x - tw_jp/2, jp_y), jp_part, font_jpn, WHITE)
```

### 11-5. TOP_TEXT 로딩 방식

```python
def _load_top_text(base_dir):
    # youtube_meta.json의 title 필드에서 ⊹ 기준 분리
    # 예: "Melody Beyond Glass ❘ 硝子の旋律 ⊹ Lucid White"
    # → TOP_TEXT[0] = "Melody Beyond Glass ❘ 硝子の旋律"
    # → TOP_TEXT[1] = "⊹ Lucid White"  (현재 미사용)
```

> 웹 프로젝트 전환 시 → 링크 대상 풀영상 제목을 API에서 읽어오는 방식으로 교체 예정.

### 11-6. 하단 Glassmorphism 박스

```python
pad_x = 180   # 좌우 여백 (줄일수록 박스 넓어짐)
box_h = 140   # 박스 높이

glassmorphism_box(bg, pad_x, box_y1, W - pad_x, box_y2,
                  blur_r=18, white_alpha=45, border_alpha=50)

# 박스 내부:
# "Lucid White"         — 凸版文久見出し明朝 58pt, WHITE
# "Synced Lyrics  ❘  Melancholic J-POP"  — Pretendard 26pt, 회색
```

### 11-7. 배경 생성 방식

```python
def build_background(video_path, frame_time, work_dir):
    # 1. ffmpeg로 코러스 시작+5초 지점 프레임 추출
    # 2. 1080x1920 비율로 크롭
    # 3. GaussianBlur(40) 적용
    # 4. 화이트 오버레이 (255,255,255, alpha=55) — Lucid White 톤
```

> Dark 채널과 달리 흰색 오버레이 사용. alpha=55가 밝은 톤의 핵심.

### 11-8. ffmpeg 합성 명령

```bash
ffmpeg -ss {actual_start} -t {duration} -i {video_path} \
       -loop 1 -i {bg_png} \
       -filter_complex \
         "[0:v]scale=1080:607[scaled]; \
          [1:v]scale=1080:1920[bg]; \
          [bg][scaled]overlay=0:656[vout]" \
       -map [vout] -map 0:a \
       -c:v libx264 -preset fast -crf 18 \
       -c:a aac -ar 48000 \
       -shortest {output}
```

### 11-9. 업로드 스크립트

> 스크립트: `scripts/upload_shorts_lucidwhite.py`

- DB 없이 `05_shorts/*.mp4` 직접 스캔
- 결과 저장: `05_shorts/upload_result_lucidwhite.json`
- 제목: 풀영상 제목과 동일 (`VIDEO_TITLE` 하드코딩 — 향후 youtube_meta.json에서 읽도록 교체 예정)
- Privacy: PRIVATE → "Save" 버튼 / PUBLIC → "Publish" 버튼
- 업로드 직후 browser-use로 댓글 달기 (API 사용 안 함)

### 11-10. 댓글 템플릿 (Lucid White)

```
The echoes that linger long after the music stops. ❘ Experience the full 51-minute melancholic journey here. 👉 https://youtu.be/{FULL_VIDEO_ID} ⊹✧
```

> ⚠️ Private 상태에서도 browser-use 댓글은 시도 가능하나 실제 게시 여부 확인 필요.
> Public 전환 후 댓글 권장 (API 방식의 경우 Private에서 403 에러).

---

## 12. 주의사항 체크리스트

### 생성 전
- [ ] 02_videos/ 에 모든 곡 렌더 완료 확인
- [ ] lyrics.json 파일명과 mp3 파일명 일치 확인
- [ ] 렌더 전 4가지 검증: txt 순서 / 중복 / 겹침 / words

### 생성 중
- [ ] 병렬 실행 시 tmp 경로 곡별 분리
- [ ] ffmpeg 프로세스 완료 확인 (moov atom 에러 주의)

### 업로드 전
- [ ] 쇼츠 영상 재생 확인 (가사 가독성, 오디오 싱크)
- [ ] 풀영상 ID 확보 (댓글 링크용)
- [ ] 채널 15분+ 인증 상태 확인 (풀영상용)

### 업로드 후
- [ ] upload_result.json 저장 확인
- [ ] 비공개 상태에서 영상 확인 → 공개 전환
- [ ] 댓글 달기 완료 → comment_result.json 저장
- [ ] Vol 번호 없이 독립 제목 (시리즈 아님)
- [ ] AI 언급 절대 금지
