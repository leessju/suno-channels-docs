# Shorts 영상 제작 가이드 (2026-04-07)

> 다른 세션에서 이 문서만 읽으면 바로 쇼츠 생성 + 업로드 + 댓글까지 가능하도록 기록.

---

## ⚠️ 트러블슈팅 이력 (2026-04-07 재작업)

### 이슈 1: browser-use python 세션 timeout
- **증상**: `TimeoutError: timed out` (socket 수준). `browser-use python -c "long_script"` 형태로 긴 업로드 루프 한 번에 실행 시 반복 실패.
- **원인**: browser-use python session은 동기 소켓이라 기본 timeout이 짧음. 파일 업로드처럼 장시간 작업에서 대기 중 끊김.
- **해결**: `subprocess.run(["browser-use", ...])` **CLI 개별 호출** 방식으로 복귀. 각 동작마다 새 프로세스로 짧게 호출. `timeout=40` 명시. 업로드 동작만 `timeout=60`.
- **원본 성공 패턴**: `/tmp/upload_shorts.py` (2026-04-06)
- **영구 버전**: `~/Music/suno-channels/scripts/upload_shorts_playlist.py` (2026-04-07)

### 이슈 2: "실패로 오해"한 부분 성공
- **증상**: 에이전트가 업로드 실패 알림을 냈지만, 실제로는 20곡 중 12곡이 이미 업로드 + DB 기록 완료 상태였음.
- **원인**: 에러 메시지만 보고 "전체 실패 → 처음부터 재시도" 판단. DB 검증 없었음.
- **해결**: 실패 알림 후 반드시 **DB에서 `youtube_video_id IS NOT NULL` 카운트 확인**. 재시도 스크립트는 **skip 로직** 필수.
- **이중 업로드 방지**: 재시도 스크립트가 `existing_vid` 확인 후 SKIP 처리.

### 이슈 3: 가사 파일명 불일치로 인한 쇼츠 생성 skip
- **증상**: 일부 곡에서 `09_消えないしるし.mp3` ↔ `09_1c4eaac4.lyrics.json` 처럼 파일명 불일치로 코러스 감지 실패.
- **원인**: Suno/Whisper 파이프라인에서 lyrics.json이 해시 기반으로 저장되는 경우 있음.
- **해결**: 쇼츠 생성 시 가사는 **코러스 감지용일 뿐**, 영상 자체엔 필요 없음. 폴백 스크립트(`gen_shorts_fallback.py`)로 영상 중반 35% 지점부터 45초 구간 사용.

---

## 📋 업로드 스크립트 3원칙 (필수)

1. **영구 경로**: `/tmp/` 금지. `~/Music/suno-channels/scripts/`에 저장
2. **DB 즉시 쓰기**: 각 아이템 완료 직후 `commit()` 호출. 배치 몰아쓰기 금지
3. **Skip 로직**: 재실행 시 `youtube_video_id IS NOT NULL` 아이템은 자동 skip

---

## 1. 쇼츠 영상 스펙

| 항목 | 값 |
|------|-----|
| **해상도** | 1080 × 1920 (9:16 세로) |
| **코덱** | H.264 (video) + AAC (audio) |
| **FPS** | 25 |
| **오디오** | 48000Hz, 2ch 스테레오 |
| **길이** | ~40초 (곡 하이라이트 구간) |
| **파일 크기** | ~8MB/개 |
| **파일명 규칙** | `{번호}_{곡명}_shorts.mp4` (예: `01_ヒーローになれなくても_shorts.mp4`) |

---

## 2. 폴더 구조

```
~/Music/suno-channels/jpop/{YYMMDD}_{playlist-name}/
├── 01_songs/          ← mp3 + jpeg(커버아트) + lyrics.json + txt(가사)
│   ├── 01_곡명.mp3
│   ├── 01_곡명.jpeg
│   ├── 01_곡명.lyrics.json    ← Whisper STT 결과 (타임스탬프)
│   └── 01_곡명.txt            ← 가사 텍스트
├── 02_videos/         ← Remotion 렌더링 결과 (1920x1080 가로, 풀영상용)
├── 03_subtitles/      ← 다국어 SRT
├── 04_final/          ← concat 머지 최종 영상
├── 05_shorts/         ← 쇼츠 영상 (1080x1920 세로)
│   ├── 01_곡명_shorts.mp4
│   ├── 02_곡명_shorts.mp4
│   └── ...
└── tracklist.txt      ← YouTube 설명란 타임스탬프
```

---

## 3. 쇼츠 생성 방법

### 생성 스크립트
- 원본: `/tmp/gen_shorts_v2.py` (hero-vol1용)
- ref 복사본: `/tmp/gen_shorts_ref1.py`, `/tmp/gen_shorts_ref2.py`
- **주의**: /tmp 파일은 리부트 시 삭제됨. 필요시 재작성 필요.

### 쇼츠 생성 로직 (Python + ffmpeg)

```python
# 핵심 흐름
# 1. 02_videos/ 에서 가로 영상(1920x1080) 로드
# 2. 하이라이트 구간 선택 (코러스 또는 에너지 최고 구간, ~40초)
# 3. 세로(1080x1920)로 크롭/변환
# 4. 배경: 블러 확대 + 중앙에 원본 영상 배치
# 5. 곡명 + 아티스트 텍스트 오버레이
# 6. 05_shorts/ 에 저장
```

### ffmpeg 변환 핵심 명령

```bash
# 가로(1920x1080) → 세로(1080x1920) 변환
# 방법: 블러 배경 + 중앙 원본
ffmpeg -i input.mp4 \
  -filter_complex "
    [0:v]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=20:5[bg];
    [0:v]scale=1080:-1[fg];
    [bg][fg]overlay=(W-w)/2:(H-h)/2
  " \
  -ss {start} -t {duration} \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  output_shorts.mp4
```

### 하이라이트 구간 선택
- `lyrics.json`에서 코러스 구간 자동 감지
- 또는 `_energy_analysis.json` (librosa)에서 에너지 피크 구간
- 기본: 곡 중반 40초 (대부분 코러스가 위치)

### 병렬 생성 시 주의
- **tmp 파일 충돌**: 병렬 실행 시 tmp_frame, bg 파일이 충돌함
- **해결**: 곡별로 임시 경로 분리 (곡명 해시 기반 디렉터리)
- ref-vol1/vol2 동시 생성 시 이 문제 발생 → 경로 분리로 해결한 이력 있음

### lyrics.json 파일명 불일치 문제
- 일부 곡에서 mp3 파일명과 lyrics.json 파일명이 다를 수 있음
- 예: `09_消えないしるし.mp3` ↔ `09_消えない印.lyrics.json`
- **해결**: 개별 생성하거나 파일명 매칭 로직 추가

---

## 4. 쇼츠 업로드 방법

### 방법 1: browser-use (확정 방식)
```bash
# browser-use 로 YouTube Studio에 직접 업로드
# 프로필: nicejames (季節のプレイリスト 기본 채널)
browser-use --profile nicejames
```

**업로드 스크립트**: `/tmp/upload_shorts.py`
- YouTube Studio → Upload 버튼 → 파일 선택 → 제목/설명 입력 → Shorts로 게시
- **Close 버튼 선택자** (필수):
  ```javascript
  Array.from(document.querySelectorAll('button'))
    .find(b => b.getAttribute('aria-label') === 'Close' && b.offsetParent !== null)
    ?.click()
  ```
- 40개 순차 업로드 시 ~10분 소요

### 방법 2: YouTube API (quota 주의)
- API 업로드는 1회 1,600 units
- 20개 쇼츠 = 32,000 units (일일 한도 10,000 초과)
- **→ 쇼츠 업로드는 browser-use 필수, API 사용 금지**

### 업로드 결과 저장
- `/tmp/upload_shorts_result.json` — `{ "video_id": "filename" }` 매핑
- 이 파일이 댓글 달기에 필요

### 브랜드 채널 업로드 시
- **browser-use로 브랜드 채널 전환 불가** (확인된 제한)
- 季節のプレイリスト는 기본 채널이라 전환 불필요 → browser-use OK
- Hazy Roast, ZERO MERCY BEATS는 브랜드 채널 → **API 토큰 사용 필수**
- 채널별 토큰:
  - 季節のプレイリスト: `~/.claude/youtube_token_jpop.json`
  - Hazy Roast: `~/.claude/youtube_token_cafe.json`
  - ZERO MERCY BEATS: `~/.claude/youtube_token_phonk.json`

---

## 5. 쇼츠 댓글 달기

### 확정 방식: YouTube Data API (browser-use 방식은 불안정)

```python
# /tmp/api_comment.py 핵심 로직
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

creds = Credentials.from_authorized_user_file("~/.claude/youtube_token_jpop.json")
youtube = build("youtube", "v3", credentials=creds)

youtube.commentThreads().insert(
    part="snippet",
    body={
        "snippet": {
            "videoId": shorts_video_id,
            "topLevelComment": {
                "snippet": {
                    "textOriginal": comment_text
                }
            }
        }
    }
).execute()
```

### 댓글 형식 (확정)
```
{풀영상 제목}, 모았습니다
🎵아래 플레이리스트로 끝까지 들어보세요. 감사합니다.
👉 https://youtu.be/{풀영상_ID}
```

예시:
```
한번 들으면 하루종일 머릿속에 진짜 맴도는 노래들, 모았습니다
🎵아래 플레이리스트로 끝까지 들어보세요. 감사합니다.
👉 https://youtu.be/SzJ2nijGzBg
```

### 줄바꿈
- API 사용 시 `\n` 자동 줄바꿈 → 문제 없음
- browser-use 시 `document.execCommand('insertLineBreak')` 필요 (Shift+Enter 안 됨)

### quota 소비
- `commentThreads.insert` = 50 units/회
- 20개 쇼츠 댓글 = 1,000 units
- 40개 = 2,000 units
- **업로드와 달리 댓글은 API가 효율적 (quota 적게 사용)**

### 댓글 결과 저장
- `/tmp/api_comment_done.json` — 완료 목록

---

## 6. 쇼츠 제목/설명 규칙

### 제목
- 곡명 그대로 사용 (일본어)
- 예: `ヒーローになれなくても`
- 해시태그 없음 (YouTube가 자동으로 #Shorts 추가)

### 설명
```
{곡명} - {아티스트명 또는 채널명}

🎵 풀 플레이리스트:
👉 https://youtu.be/{풀영상_ID}

#JPOP #감성 #플레이리스트
```

---

## 7. 현재 쇼츠 현황

### 생성 완료 (로컬, 100개)
| 플레이리스트 | 위치 | 곡수 | 상태 |
|-------------|------|------|------|
| hero-vol1 | `260402_hero-vol1/05_shorts/` | 20개 | ✅ 생성 완료 |
| hero-vol2 | `260402_hero-vol2/05_shorts/` | 20개 | ✅ 생성 완료 |
| hero-vol3 | `260402_hero-vol3/05_shorts/` | 20개 | ✅ 생성 완료 |
| ref-vol1 | `260402_ref-vol1/05_shorts/` | 20개 | ✅ 생성 완료 |
| ref-vol2 | `260402_ref-vol2/05_shorts/` | 20개 | ✅ 생성 완료 |

### 업로드 이력 (모두 삭제됨 — 모바일 가사 문제)
- hero-vol1 쇼츠 20개: 업로드 + 댓글 완료 → **삭제**
- hero-vol2 쇼츠 20개: 업로드 + 댓글 완료 → **삭제**
- hero-vol3 쇼츠 20개: 업로드 + 댓글 완료 → **삭제**
- ref-vol1 쇼츠 20개: 업로드 완료 → **삭제 대상**
- ref-vol2 쇼츠 20개: 업로드 완료 → **삭제 대상**

### 삭제 이유
- 모바일에서 번인 가사가 너무 작아 안 보임
- 구독자 1명, 조회수 거의 없음 → 손실 없이 재제작 가능
- **→ Remotion 레이아웃 재설계 후 100곡 재렌더 → 쇼츠 재생성 → 재업로드 예정**

---

## 8. 쇼츠 업로드 결과 (참고용 — 삭제 전 ID)

### Vol2 쇼츠 (풀영상: https://youtu.be/SzJ2nijGzBg)
| # | 제목 | 쇼츠 ID |
|---|------|---------|
| 01 | 名前のない感情 | Pa_CUwaXahQ |
| 02 | キミという光 | wdUnz8wzfR0 |
| 03 | 冬の手紙 | 032qP2Tt7_s |
| 04 | 約束の場所 | RypYnm8X-nI |
| 05 | 海辺の夢 | PStYz9MqdXM |
| 06 | 写真の中の僕ら | Pu7GwErCu3k |
| 07 | 星屑のワルツ | QnV_17u0foE |
| 08 | 最高の仲間 | NZI5xFpnMXE |
| 09 | 朝焼けのランナー | Ob4HQUTvfr8 |
| 10 | 雨の帰り道 | 3Su4A8Hjx1I |
| 11 | ヒーローになれなくても | 7wpX3lOaljc |
| 12 | 走り出せ | yN7-nu_Z2Mc |
| 13 | 夜空のメッセージ | U57SRLKtORs |
| 14 | この声が届くまで | z5YSMVRZPbM |
| 15 | 明日への手紙 | h181AzfH9gw |
| 16 | 不完全な僕のままで | q0uZ-yfZNJo |
| 17 | 不屈のメロディ | sShr8t4WHqk |
| 18 | 嵐の後で | 5HZwr9xw-bw |
| 19 | 帰り道のハミング | TnqY7tvYHPE |
| 20 | 日曜日のヒーロー | OmOeltDNP4A |

### Vol3 쇼츠 (풀영상: https://youtu.be/j_ZMiZk9jfI)
| # | 제목 | 쇼츠 ID |
|---|------|---------|
| 01 | ペダルを漕いで | NkMcEtgYhL4 |
| 02 | 図書館の秘密 | GPNNv2yNYjQ |
| 03 | 手紙 | QYVNlmb5O_w |
| 04 | ただいまの温度 | 57F-wlENgmw |
| 05 | 君と見た夕焼け | S2Hs7B8dWNA |
| 06 | 最終電車 | b5VhOo703jQ |
| 07 | 雨のち晴れ | gWIPeA9Opos |
| 08 | さよならの練習 | JFBNjFs3tiM |
| 09 | おはようの魔法 | SwPTALD1A0E |
| 10 | 小さな勇気 | iwBKqP92Lto |
| 11 | 週末アドベンチャー | HBxCJFqEp2w |
| 12 | 夏祭りの夜 | wHR8Ul613T8 |
| 13 | 雪の降る街 | mpPGtRPad9E |
| 14 | 写真 | aCs8lerO7FA |
| 15 | 走れ、僕のスニーカー | doUXgsd8npM |
| 16 | ありがとうの歌 | nawItr5vgIQ |
| 17 | あの空の向こうへ | zeoopjk15uU |
| 18 | 隣の席の君 | bP2pEE73RKQ |
| 19 | 忘れられない味 | V5p-I6kRnBE |
| 20 | 夜明けのメロディ | Xg6FZaYKCDc |

---

## 9. 다음 작업 순서 (재제작 플로우)

```
1. Remotion 레이아웃 재설계 (모바일 가사 가독성)
   → docs/thumbnail_strategy_v2.md "Remotion 영상 레이아웃 재설계" 섹션 참조
   → ~/Music/suno-channels/_raw/design_research/gemini_design_direction.md

2. 100곡 재렌더링 (1920x1080 가로)
   → bash ~/Projects/clones/suno-video/scripts/render_playlist.sh <폴더>
   → 5개 플레이리스트 순차 실행

3. 쇼츠 재생성 (1080x1920 세로)
   → gen_shorts_v2.py 기반 스크립트
   → 병렬 생성 시 tmp 경로 분리 필수

4. 풀영상 머지
   → ffmpeg concat (1.5초 무음 갭 필수)
   → nohup으로 실행 (프로세스 종료 방지, moov atom 에러 방지)

5. 썸네일 적용
   → docs/thumbnail_strategy_v2.md 참조

6. YouTube 업로드
   → 풀영상: API (브랜드 채널) 또는 browser-use (기본 채널)
   → 쇼츠: browser-use (quota 절약)
   → 항상 비공개 먼저 → 확인 후 공개

7. 댓글 달기
   → YouTube API commentThreads.insert
   → 50 units/회, 총 ~5,000 units (100개)

8. SRT 자막 업로드
   → YouTube API captions.insert (400 units/건)
   → 상위 10개 언어
```

---

## 10. 핵심 주의사항

1. **API quota 보존**: 업로드는 browser-use, 댓글/자막만 API 사용
2. **곡 합성 시 1.5초 무음 갭 필수** — 재정렬 시 4종 세트(mp3, jpeg, lyrics.json, txt) 함께 이동
3. **nohup으로 ffmpeg 실행** — 에이전트 프로세스 종료 시 moov atom not found 에러
4. **렌더 전 4가지 검증**: txt 순서 / 중복 / 겹침 / words
5. **병렬 쇼츠 생성 시 tmp 경로 분리** — 파일 충돌 방지
6. **lyrics.json 파일명 매칭 확인** — mp3와 불일치 가능
7. **Vol 번호 금지** — YouTube 영상은 독립 배포 (시리즈 아님)
8. **비공개 먼저 업로드** → 영상/자막/썸네일 확인 → 공개 전환
9. **15분 초과 영상**: Intermediate features 인증 필요 (채널별 전화번호)
   - 季節のプレイリスト: ✅ 인증 완료
   - Hazy Roast: ❌ 미인증
   - ZERO MERCY BEATS: ❌ 미인증

---

## 11. YouTube 채널 정보

| 채널 | ID | 토큰 | 15분+ | 기본/브랜드 |
|------|-----|------|-------|-----------|
| 季節のプレイリスト | UC2ofS0Y6ynIjRSVwexUUWQg | `youtube_token_jpop.json` | ✅ | 기본 (browser-use OK) |
| **Lucid White** | **UCoHpJmMju00FPBogQr6D_Kw** | (nicejames 프로필) | ✅ | 기본 (browser-use OK) |
| Hazy Roast | UCSvzzpXpaXwRWi3G-grk_7A | `youtube_token_cafe.json` | ❌ | 브랜드 (API 필수) |
| ZERO MERCY BEATS | UC0OSrx55lFo7ELCMo88mxUw | `youtube_token_phonk.json` | ❌ | 브랜드 (API 필수) |

---

## 13. Lucid White 채널 쇼츠 규칙 (2026-04-13)

### 채널 정체성
- 채널명: **Lucid White** (`@Lucid White`)
- 채널 ID: `UCoHpJmMju00FPBogQr6D_Kw`
- 컨셉: "Pure sound. Lucid vision. Eternal lingering." / "Beyond the frosted glass, memories begin to flow."
- 업로드 스크립트: `scripts/upload_shorts_lucidwhite.py`
- 결과 저장: `05_shorts/upload_result_lucidwhite.json`

### 쇼츠 제목 규칙
- 풀영상 제목과 동일 사용: **`Melody Beyond Glass ❘ 硝子の旋律 ⊹ Lucid White`**
- 20개 쇼츠 모두 동일 제목

### 쇼츠 설명
```
#JPOP #LucidWhite #JpopPlaylist #Melancholic #Aesthetic #Visualizer #Lyrics #NightVibes #EmotionalJpop #JapanMusic #ChillJpop
```

### 쇼츠 업로드 공개 설정
- 항상 **비공개(private)** 로 먼저 업로드 → 확인 후 공개 전환

### 댓글 (업로드 직후 자동 작성)
```
The echoes that linger long after the music stops. ❘ Experience the full 51-minute melancholic journey here. 👉 https://youtu.be/{풀영상_ID} ⊹✧
```
- hero-vol1 풀영상 ID: `6h7U7VnfzF4`

### 풀영상 메타 규칙
- 제목: `Melody Beyond Glass ❘ 硝子の旋律 ⊹ Lucid White`
- 설명 헤더: `"Beyond the frosted glass, memories begin to flow."`
- 태그: `#JPOP #LucidWhite #JpopPlaylist #Melancholic #Aesthetic #Visualizer #Lyrics #NightVibes #EmotionalJpop #JapanMusic #ChillJpop`
- Privacy 업로드: Save 버튼 (Public은 Publish 버튼) — `upload_full_video.py` 수정 완료

### 기술 주의사항
- DB skip 로직 우회: `upload_shorts_lucidwhite.py`는 DB 대신 `upload_result_lucidwhite.json`으로 중복 방지
- 기존 `suno_tracks` DB 경로가 재정렬 후 불일치 → 파일시스템 직접 스캔 방식 사용
- STUDIO_URL: `https://studio.youtube.com` (채널 특정 URL 사용 금지 — permission 오류)
- Private 업로드 시 "Save" 버튼 대기 (Publish 아님)

---

## 12. Remotion 렌더 참고

### 풀영상 렌더 (가로 1920x1080)
```bash
# 단일 곡
bash ~/Projects/clones/suno-video/scripts/render.sh /path/to/song.mp3

# 플레이리스트 전체
bash ~/Projects/clones/suno-video/scripts/render_playlist.sh ~/Music/suno-channels/jpop/260402_hero-vol1
```

### 렌더 파이프라인 (8단계 자동)
1. mp3 → public/ 임시 복사
2. Whisper STT → lyrics.json (캐시됨)
3. Vertex AI Gemini → 가사 번역
4. 커버 이미지 준비 (Suno 이미지 우선, 없으면 Imagen 생성)
5. 커버 → 대표 색상 추출
6. 후미 무음 감지 → duration 계산
7. props.json → Remotion DynamicVisualizer 렌더
8. 오디오 검증 (volumedetect)

### 환경 변수 (.env)
```bash
WHISPER_MODEL=medium
REMOTION_CONCURRENCY=2
REMOTION_TIMEOUT=120000
DEFAULT_COLOR=#8B5CF6
MUSIC_ROOT=/Users/nicejames/Music/suno-channels
```

### 프로젝트 경로
- Remotion: `~/Projects/clones/suno-video/`
- 스크립트: `~/Projects/clones/suno-video/scripts/`
- 소스 코드: `~/Projects/clones/suno-video/src/`
