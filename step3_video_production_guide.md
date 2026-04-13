# suno-video Remotion 파이프라인 완전 가이드

> 다른 세션에서 이 문서만 읽고 모든 작업을 수행할 수 있도록 작성됨.
> 최종 업데이트: 2026-04-13

---

## 1. 프로젝트 개요

- **목적**: Suno AI 음악을 카라오케 영상으로 자동 변환
- **프로젝트 경로**: `/Users/nicejames/Projects/clones/suno-video`
- **음악 경로**: `/Users/nicejames/Music/suno-channels`
- **기술 스택**: Remotion 4.0.443 + React 19 + TypeScript + mlx-whisper (Apple Silicon) + faster-whisper (VAD/fallback) + Demucs + ffmpeg + Vertex AI
- **하드웨어**: Apple M1 Pro, 32GB RAM, macOS Darwin 25.3.0
- **Python 환경**: `/opt/miniconda3/envs/suno-video/bin/python3` (Python 3.11)

---

## 2. 프로젝트 구조

```
/Users/nicejames/Projects/clones/suno-video/
├── .env                          # 환경 설정
├── package.json                  # Remotion 4.0.443 + React 19 + Zod 4
├── remotion.config.ts            # Webpack (Tailwind v4, JPEG, 동적 public dir)
├── src/
│   ├── index.ts                  # registerRoot 진입점
│   ├── Root.tsx                  # Composition 등록 (AudioVisualizer, DynamicVisualizer)
│   ├── helpers/
│   │   ├── schema.ts             # Zod 스키마 (props 타입 정의)
│   │   ├── font.ts               # IBM Plex Sans 500/700 로딩
│   │   ├── process-frequency-data.ts
│   │   └── WaitForFonts.tsx
│   └── Visualizer/
│       ├── Main.tsx              # 메인 비주얼라이저 (레이아웃 + 조합)
│       ├── Background.tsx        # 부유 파티클 배경 (90개 입자)
│       ├── AlbumArt.tsx          # 앨범 아트 (비트 펀치 + 스프링 바운스)
│       ├── LightWave.tsx         # 동심원 링 (3개, 비트 반응)
│       ├── LyricsBox.tsx         # 가사 표시 (카라오케 + 스크롤 + 번역)
│       ├── MiniEQ.tsx            # 미니 이퀄라이저 (64바)
│       ├── BassOverlay.tsx       # 베이스 오버레이 (현재 비활성화)
│       ├── Spectrum.tsx          # 스펙트럼 바
│       ├── Waveform.tsx          # 오실로스코프 웨이브폼
│       ├── VerticalBars.tsx      # 세로 바
│       └── SongInfo.tsx          # 곡 정보 (레거시, 미사용)
├── scripts/
│   ├── render.sh                 # 핵심: 단일 곡 8단계 렌더 파이프라인
│   ├── render_playlist.sh        # 플레이리스트 폴더 일괄 렌더
│   ├── render_all_jpop.sh        # 5개 볼륨 배치 렌더 (프로세스 격리, 좀비 방지)
│   ├── batch_render_jpop.sh      # [deprecated] 구 배치 스크립트 → render_all_jpop.sh로 대체
│   ├── nohup_render_remaining.sh # [deprecated] nohup 백그라운드 렌더 → 불안정하여 사용 보류
│   ├── extract_lyrics.py         # 가사 추출 + 타이밍 (1322줄, 핵심)
│   ├── translate_lyrics.py       # Vertex AI Gemini 영어 번역 (--force 재번역 지원)
│   ├── generate_lyrics_txt.py    # Gemini STT로 가사 txt 자동 생성
│   ├── generate_cover.py         # Gemini + Imagen 3 앨범 커버 생성
│   ├── gen_cover.py              # 수동 커버 생성 (프롬프트 직접 지정)
│   ├── verify_lyrics.py          # 기본 가사 품질 검증 (6항목)
│   ├── verify_all.py             # 정밀 가사 검증 (8항목)
│   ├── batch_verify.py           # 배치 추출 + 검증 통합
│   ├── generate_subtitles.py     # 10개국 SRT 자막 생성 (Claude Haiku)
│   ├── concat_videos.sh          # FFmpeg 영상 합성 (concat/xfade)
│   ├── youtube_upload.py         # YouTube 업로드 + 자막 첨부
│   ├── test_whisper_compare.py   # faster-whisper vs mlx-whisper 1곡 비교 테스트
│   └── test_whisper_batch.py     # faster-whisper vs mlx-whisper 20곡 배치 비교 테스트
└── .claude/CLAUDE.md             # 프로젝트 규칙 문서
```

---

## 3. 실행 명령어

### 3.1 플레이리스트 전체 렌더

```bash
bash /Users/nicejames/Projects/clones/suno-video/scripts/render_playlist.sh <플레이리스트_폴더>
```

예시:
```bash
bash /Users/nicejames/Projects/clones/suno-video/scripts/render_playlist.sh /Users/nicejames/Music/suno-channels/jpop/260402_hero-vol1
```

### 3.2 단일 곡 렌더

```bash
bash /Users/nicejames/Projects/clones/suno-video/scripts/render.sh /path/to/song.mp3
```

### 3.3 5개 볼륨 배치 렌더 (권장)

```bash
cd /Users/nicejames/Projects/clones/suno-video
bash scripts/render_all_jpop.sh
```

**환경변수 옵션:**
| 변수 | 기본값 | 설명 |
|------|--------|------|
| TRACK_TIMEOUT | 1800 (30분) | 곡당 최대 렌더 시간 (초). 초과 시 프로세스 그룹 강제 종료 |
| SKIP_EXISTING | 1 | 1이면 기존 mp4가 있는 곡 스킵 |
| RENDER_RETRY | 0 | 오디오 검증 실패 시 재시도 횟수 |

**주요 안전 기능:**
- 프로세스 그룹 격리: 각 곡 렌더를 서브셸 `( bash ... ) &`로 실행
- Watchdog 타임아웃: 30분 초과 시 프로세스 그룹 + 고아 chrome-headless-shell 자동 kill
- Remotion 캐시 자동 클리어: 시작 시 `node_modules/.cache`, `/tmp/remotion-*` 삭제
- `_origin.mp3`, `_vocals.mp3` 자동 필터링
- 요약 로그: `/tmp/jpop_render_summary.log`
- EXIT/INT/TERM 시그널에 `_batch_cleanup` 트랩

**사용 예시 (로그 기록):**
```bash
cd /Users/nicejames/Projects/clones/suno-video
bash scripts/render_all_jpop.sh 2>&1 | tee /tmp/jpop_render_all.log
```

> **nohup 사용 주의**: nohup 환경에서 ffmpeg 무한루프, 바이너리 로그 손상, 로케일 깨짐, 프로세스 추적 불가 등 다수 이슈가 확인됨. 안정성이 검증될 때까지 직접 세션 실행을 권장.

### 3.4 재렌더 시 (기존 mp4 삭제 필요)

```bash
# SKIP_EXISTING=0 으로 실행하면 기존 mp4 무시하고 재렌더
SKIP_EXISTING=0 bash scripts/render_all_jpop.sh

# 또는 특정 vol만 mp4 삭제 후 실행
rm -f /Users/nicejames/Music/suno-channels/jpop/260402_hero-vol1/02_videos/*.mp4

# 전체 5개 vol mp4 삭제
for vol in hero-vol1 hero-vol2 hero-vol3 ref-vol1 ref-vol2; do
  rm -f /Users/nicejames/Music/suno-channels/jpop/260402_${vol}/02_videos/*.mp4
done
```

### 3.5 플레이리스트 머지 (1.5초 무음 갭)

```bash
BASE=~/Music/suno-channels/jpop/260402_hero-vol1

# 무음 파일 생성
ffmpeg -y -f lavfi -i "color=c=black:s=1920x1080:r=30:d=1.5" \
  -f lavfi -i "anullsrc=r=48000:cl=stereo" -t 1.5 \
  -c:v libx264 -profile:v high -level 4.0 -pix_fmt yuv420p -preset fast -crf 23 \
  -c:a aac -ar 48000 -ac 2 -shortest "$BASE/04_final/silence_1.5s.mp4"

# concat_list.txt 생성
> "$BASE/04_final/concat_list.txt"
first=true
for f in "$BASE"/02_videos/*.mp4; do
  [ "$first" = true ] && first=false || echo "file '$BASE/04_final/silence_1.5s.mp4'" >> "$BASE/04_final/concat_list.txt"
  echo "file '$f'" >> "$BASE/04_final/concat_list.txt"
done

# 머지
ffmpeg -y -f concat -safe 0 -i "$BASE/04_final/concat_list.txt" \
  -c copy -movflags +faststart "$BASE/04_final/hero-vol1_playlist.mp4"
```

### 3.6 Remotion Studio (프리뷰)

```bash
cd /Users/nicejames/Projects/clones/suno-video && npx remotion studio
```

---

## 4. render.sh 8단계 파이프라인 상세

**파일:** `scripts/render.sh` (530줄)
**입력:** `<mp3_path> [color] [output_path]`

| 단계 | 동작 | 입력 | 출력 | 의존성 |
|------|------|------|------|--------|
| 1/8 | mp3 복사 + 이미지 탐색 | mp3 파일 | `public/{SAFE_STEM}.mp3` | 없음 |
| 2/8 | 가사 추출 (Whisper+Demucs+txt) | mp3 + (선택) txt | `{SAFE_STEM}.lyrics.json` | mlx-whisper (primary), faster-whisper (VAD/fallback), demucs |
| 3/8 | 가사 영어 번역 (Vertex AI Gemini) | lyrics.json | lyrics.json (in-place, `--force`) | Vertex AI |
| 4/8 | 커버 이미지 준비 | lyrics.json + 곡명 | `public/cover_{SAFE_STEM}.{ext}` | (Vertex AI) |
| 5/8 | 색상 추출 (상위 10% 채도 픽셀 평균) | 커버 이미지 | COLOR, BG_COLOR | PIL/Pillow |
| 6/8 | 후미 무음 감지 (-45dB 스캔) | mp3 | MUSIC_DURATION | ffmpeg/ffprobe (`-nostdin -vn`) |
| 7/8 | Remotion 렌더 (DynamicVisualizer) | props.json | mp4 파일 | Remotion (`--bundle-cache=false`) |
| 8/8 | 오디오 검증 (-60dB 미만이면 재렌더) | mp4 | 검증 결과 | ffmpeg (`-nostdin -vn`) |

### SAFE_STEM 생성 로직

1. 비ASCII 문자 제거
2. `[^a-zA-Z0-9_\-]` → `_`로 치환
3. 연속 `_` 제거
4. 숫자만 남거나 빈 문자열이면 MD5 해시 8자리 추가
   - 예: `01_ヒーローになれなくても` → `01_5a1175e8`

### 출력 경로 자동 결정

- `01_songs/` 안의 mp3 → `../02_videos/{MP3_STEM}.mp4`
- 그 외 → `mp3 위치/02_videos/{MP3_STEM}.mp4`

### public/ 정책

- 렌더 중 임시로 mp3/커버를 `public/`에 복사
- `trap EXIT`으로 렌더 완료 후 자동 삭제 (명시적 확장자별 삭제, zsh glob 안전)
- 영구 파일은 절대 `public/`에 저장하지 않음

### trap 클린업 상세

```bash
# zsh 호환: glob 패턴 대신 명시적 확장자 나열
_cleanup() {
  rm -f "$PROPS_FILE"
  rm -f "public/${SAFE_STEM}.mp3"
  rm -f "public/cover_${SAFE_STEM}.png" "public/cover_${SAFE_STEM}.jpg" \
        "public/cover_${SAFE_STEM}.jpeg" "public/cover_${SAFE_STEM}.webp"
  rm -f "public/bg_${SAFE_STEM}.png" "public/bg_${SAFE_STEM}.jpg" \
        "public/bg_${SAFE_STEM}.jpeg" "public/bg_${SAFE_STEM}.webp"
}
trap '_cleanup' EXIT
```

### 오디오 검증 상세 (8/8단계)

```bash
# 필수 플래그: -nostdin (non-TTY 입력 차단) + -vn (비디오 스트림 무시 → 1.2초 완료)
# -vn 없으면 풀 비디오 디코드 → 31초+, non-TTY에서 무한루프/좀비 발생
result=$(ffmpeg -nostdin -vn -i "$file" -af volumedetect -f null /dev/null 2>&1)

# set -e 안전한 비교 (bc 대체 → awk)
if awk "BEGIN{exit(!(${mean_num:--100} < -60))}"; then
  return 1  # 무음/손상
fi
```

**재렌더 시 에셋 복구:**
- 8/8 검증 실패 → 재렌더 전 `public/` 에셋(mp3, 커버, 배경 이미지) 자동 복구
- trap에 의해 이전 렌더에서 삭제됐을 수 있으므로 존재 여부 확인 후 재복사

---

## 5. Remotion 컴포지션 상세

### 5.1 영상 사양

| 항목 | 값 |
|------|-----|
| 해상도 | 1920x1080 (Full HD, 16:9) |
| FPS | 30 |
| 이미지 포맷 | JPEG |
| 코덱 | H.264 |
| 동시 처리 | 2 (REMOTION_CONCURRENCY) |
| 프레임 타임아웃 | 120000ms |
| 번들 캐시 | false (안정성 위해 비활성) |

### 5.2 Props 스키마 (`src/helpers/schema.ts`)

```typescript
visualizerCompositionSchema = z.object({
  visualizer: discriminatedUnion("type", [spectrum, oscilloscope]),
  textColor: zColor(),
  coverImageUrl: z.string(),
  songName: z.string(),
  artistName: z.string(),
  audioFileUrl: z.string(),
  audioOffsetInSeconds: z.number().min(0),
  backgroundColor: zColor().default("#06030f"),
  lyrics: z.array(lyricLineSchema).default([]),
  durationInSeconds: z.number().optional(),
})
```

**lyricLineSchema:**
```typescript
{ text: string, translation: string, start: number, end: number,
  words: [{word: string, start: number, end: number}] }
```

### 5.3 비주얼 이펙트

**활성:**
| 이펙트 | 컴포넌트 | 설명 |
|--------|----------|------|
| 부유 파티클 | Background.tsx | 90개 입자, 시드 기반 결정적 랜덤, 느린 상승 |
| 앨범 아트 비트 펀치 | AlbumArt.tsx | 베이스 반응 최대 18% 확장, 스프링 바운스, X/Y 스퀴시 |
| 동심원 링 | LightWave.tsx | 3개 링, 다른 크기/투명도로 비트 반응 |
| 미니 EQ | MiniEQ.tsx | 64바 좌우 대칭, 중앙 강조 |
| 카라오케 가사 | LyricsBox.tsx | 단어별 하이라이트, 영어 번역 동시 표시, 비트 쉐이크 |
| 글로우 | AlbumArt.tsx | 앨범 아트 주변 동적 글로우 (베이스 강도 비례) |
| 슬라이드업 | LyricsBox.tsx | 가사 라인 아래→위 등장 |

**비활성 (YouTube 압축 화질저하 방지):**
- BassOverlay, 비네팅, LightWave boxShadow

### 5.4 레이아웃 구조 (Main.tsx)

**가사 있을 때:**
- 왼쪽 40%: 앨범 아트 (480px) + 곡명 + MiniEQ
- 오른쪽 62% (38%~100%): 가사 박스

**인스트루멘탈:**
- 전체 100%: 앨범 아트 (600px) + 곡명 + MiniEQ (중앙 정렬)

### 5.5 색상 처리

**추출 (render.sh):**
1. 커버 100x100 리사이즈
2. 밝기 40~220 필터링
3. 채도(HSV S) 상위 10% 픽셀 RGB 평균
4. 채도 < 0.2이면 기본색 `#8B5CF6`
5. 배경색: 같은 Hue, 채도 75%, 밝기 10%

**보정 (Main.tsx vivifyColor):**
- HSV 변환 → 채도 100%, 명도 55% 강제 → 선명한 네온 색상

---

## 6. 가사 처리 파이프라인 상세

### 6.1 두 가지 모드

**모드 A: txt 있을 때 (주력, `align_lyrics()`):**
1. Demucs 보컬 분리 → VAD 보컬 시작점 감지
2. 원본 MP3 → Whisper STT (세그먼트 + 단어 타임스탬프)
3. VAD 시작점 이전 세그먼트 제거 (인트로 환청 필터)
4. Demucs 보컬 Whisper로 갭 보완
5. txt 텍스트 교정 (Whisper → txt 정답으로 교체)
6. mega-segment 분할 (하나의 세그먼트에 여러 txt 줄)
7. Whisper 잔여 제거 (txt에 없는 텍스트)
8. 미매칭 보간 삽입 (RMS 에너지 + VAD 간주 감지)
9. 최종 보장 (누락된 txt 줄 강제 삽입)
10. Demucs 보컬로 보간/극소 words 보완
11. 안전장치: words/text 불일치 시 균등 재생성

**모드 B: txt 없을 때 (폴백, `extract_lyrics()`):**
- mlx-whisper 우선, 없으면 faster-whisper fallback
- 노이즈 패턴 필터: `^[-─—\s.…♪*]+$` (mlx-whisper 특유의 `---`, `--`, `♪` 등)
- 크레딧 라인 필터, Whisper 아티팩트 제거

### 6.2 Whisper STT 설정

**mlx-whisper (Primary — Apple Silicon 최적화):**

| 설정 | 값 | 비고 |
|------|-----|------|
| 엔진 | mlx-whisper | Apple Silicon MLX 프레임워크 기반 |
| 모델 | `mlx-community/whisper-{model}` | HuggingFace MLX 변환 모델 |
| 기본 모델 크기 | medium (.env) | `mlx-community/whisper-medium` |
| word_timestamps | true | 단어별 타이밍 |
| condition_on_previous_text | false | 이전 컨텍스트 비의존 |
| temperature | 기본 튜플 (0.0, 0.2, 0.4, 0.6, 0.8, 1.0) | 0.0 고정보다 품질 우수 |
| 곡당 처리 시간 | ~9초 (M1 Pro) | faster-whisper 대비 4.5배 빠름 |

**faster-whisper (VAD 전용 + Fallback):**

| 설정 | 값 | 비고 |
|------|-----|------|
| 엔진 | faster-whisper | CTranslate2 기반 |
| 모델 | tiny (VAD) / medium (STT fallback) | VAD는 tiny로 충분 |
| device | cpu | Apple Silicon에서 GPU 미지원 |
| compute_type | int8 | CPU 최적화 양자화 |
| beam_size | 5 (STT) / 1 (VAD) | |
| vad_filter | true (VAD 모드) | mlx-whisper에는 vad_filter 없음 |
| 곡당 처리 시간 | ~42초 (M1 Pro) | |

**엔진 선택 로직 (extract_lyrics.py):**
```python
# mlx-whisper 우선, 없으면 faster-whisper fallback
use_mlx = False
try:
    import mlx_whisper
    use_mlx = True
except ImportError:
    from faster_whisper import WhisperModel
```

### 6.3 lyrics.json 스키마

```json
[
  {
    "text": "朝が来るたびに思い出す",
    "translation": "Every morning brings back memories",
    "start": 13.98,
    "end": 20.88,
    "words": [
      {"word": "朝", "start": 13.76, "end": 13.94},
      {"word": "が", "start": 13.94, "end": 15.28}
    ]
  }
]
```

> **번역 필드**: 2026-04-13 기준 영어(`English`). 이전에는 한국어였으나 영어로 변경됨.

### 6.4 핵심 규칙

- **txt = 절대 정답**: Whisper는 타이밍만, 텍스트는 txt가 유일한 소스
- **섹션 레이블 필터**: `[\[\(（【].*[\]\)）】]` 패턴 제거
- **카타카나→로마자 변환**: Whisper 히라가나/카타카나 혼동 보정
- **한국어 원본 스킵**: '가'~'힣' 비율 30% 이상이면 번역 건너뜀
- **mlx-whisper 노이즈 필터**: `^[-─—\s.…♪*]+$` 패턴 세그먼트 자동 제거
- **번역 재실행**: `translate_lyrics.py --force`로 기존 번역 덮어쓰기 (가사 텍스트가 아닌 번역만 갱신)

### 6.5 번역 파이프라인 상세

**파일:** `scripts/translate_lyrics.py` (108줄)

| 항목 | 값 |
|------|-----|
| 모델 | Vertex AI Gemini 2.5 Flash |
| 번역 방향 | 일본어(ja) → 영어(English) |
| 배치 크기 | 30줄 |
| 프로젝트 | GCP `project-3f3c3c98-9d26-4383-bcb` |
| 리전 | us-central1 |
| 인증 | `/Users/nicejames/Projects/claude/test/niche_bending/infra/credentials_account3.json` |
| `--force` | 이미 번역된 항목도 재번역 |

**프롬프트:**
```
Translate the following {src_lang} song lyrics to English.
Keep the translation natural and poetic, matching the emotional tone of the original.
Return ONLY the numbered translations, one per line, in the same order.
Do NOT include romanization, explanations, or original text.
```

---

## 7. 음악 폴더 구조

### 7.1 전체 구성

```
/Users/nicejames/Music/suno-channels/jpop/
├── 260402_hero-vol1/    (20곡, 4.6GB)
├── 260402_hero-vol2/    (20곡, 4.3GB)
├── 260402_hero-vol3/    (20곡, 2.7GB)
├── 260402_ref-vol1/     (20곡, 3.1GB)
├── 260402_ref-vol2/     (20곡, 3.2GB)
└── images_album/        (볼륨별 대표 이미지)
```

### 7.2 볼륨 내부 구조

```
260402_hero-vol1/
├── 01_songs/            # 원본 소스
│   ├── 01_곡명.mp3          # 원본 MP3 (3-4MB)
│   ├── 01_곡명.txt          # 가사 텍스트 (정답)
│   ├── 01_곡명.jpeg         # Suno 커버 이미지
│   ├── 01_곡명.lyrics.json  # 가사 JSON (원본 이름)
│   ├── 01_SAFE.lyrics.json  # SAFE_STEM 가사 JSON (캐시)
│   └── 01_곡명_vocals.wav   # Demucs 보컬 분리 (24-34MB)
├── 02_videos/           # 렌더 출력 (개별 mp4, 29-76MB)
├── 03_subtitles/        # 10개국 SRT 자막
├── 04_final/            # 합본 플레이리스트 영상 (1.3-1.7GB)
│   ├── silence_1.5s.mp4
│   ├── concat_list.txt
│   └── hero-vol1_playlist.mp4
├── 05_shorts/           # 숏폼 버전 (5.6-9.8MB)
├── tracklist.txt
└── youtube_meta.json
```

### 7.3 곡 목록 (hero-vol1 예시)

```
01_ヒーローになれなくても    11_星屑のワルツ
02_嵐の後で                12_約束の場所
03_名前のない感情          13_夜空のメッセージ
04_この声が届くまで        14_雨の帰り道
05_キミという光            15_明日への手紙
06_帰り道のハミング        16_冬の手紙
07_走り出せ                17_朝焼けのランナー
08_不完全な僕のままで      18_日曜日のヒーロー
09_海辺の夢                19_最高の仲間
10_写真の中の僕ら          20_不屈のメロディ
```

---

## 8. 환경 설정

### 8.1 .env

| 변수 | 값 | 용도 |
|------|-----|------|
| KMP_DUPLICATE_LIB_OK | TRUE | macOS PyTorch + OpenMP 충돌 방지 |
| WHISPER_MODEL | medium | Whisper STT 모델 |
| REMOTION_CONCURRENCY | 2 | 렌더 동시 처리 수 |
| REMOTION_TIMEOUT | 120000 | 프레임당 타임아웃 (ms) |
| DEFAULT_COLOR | #8B5CF6 | 기본 테마 색상 |
| MUSIC_ROOT | /Users/nicejames/Music/suno-channels | 음악 루트 |
| PYTHON | /opt/miniconda3/envs/suno-video/bin/python3 | Python 3.11 |

### 8.2 package.json 핵심 의존성

| 패키지 | 버전 | 용도 |
|--------|------|------|
| remotion | 4.0.443 | 영상 렌더 엔진 |
| @remotion/media-utils | 4.0.443 | 오디오 시각화 |
| @remotion/google-fonts | 4.0.443 | IBM Plex Sans |
| @remotion/tailwind-v4 | 4.0.443 | Tailwind CSS |
| zod | 4.3.6 | Props 스키마 검증 |
| react | 19.2.3 | UI |

### 8.3 Python 의존성

| 패키지 | 용도 | 비고 |
|--------|------|------|
| mlx-whisper | Whisper STT (Apple Silicon 최적화) | **Primary** — 곡당 ~9초, 4.5배 빠름 |
| faster-whisper | Whisper STT (CTranslate2) + VAD | **Fallback/VAD** — mlx-whisper에 vad_filter 없어서 VAD 전용 |
| demucs | 보컬 분리 (MPS 가속, CPU 대비 3배) | PyTorch MPS 백엔드 사용 |
| google-cloud-aiplatform, vertexai | Gemini 영어 번역, Imagen 커버 | Gemini 2.5 Flash |
| Pillow | 색상 추출 | |
| anthropic | Claude Haiku 자막 번역 | 10개국 SRT 생성용 |
| numpy | RMS 에너지 계산 | 간주 감지 |

---

## 9. 알려진 이슈 & 해결책

### 9.1 SAFE_STEM 캐시 충돌

**문제:** 일본어 곡명 → 비ASCII 제거 → 숫자만 남음 → 동일 SAFE_STEM → 가사 오염
**해결:** 숫자만 남으면 MD5 해시 8자리 추가. 배치 렌더 시 캐시 삭제 후 실행.

### 9.2 lyrics.json 이중 파일

`01_songs/`에 원본 이름(`01_곡명.lyrics.json`)과 SAFE_STEM(`01_hash.lyrics.json`) 공존.
- render.sh는 SAFE_STEM 버전만 사용
- verify_all.py는 glob으로 검색 → 잘못된 파일 참조 가능성

### 9.3 Whisper 타이밍 한계

vol1 곡12/16은 Whisper가 정확한 타이밍을 잡지 못함. 강제 패치하면 다른 곳이 깨짐.

### 9.4 오디오 검증 + 자동 재렌더

렌더 후 mean_volume -60dB 미만이면 무음/손상 → 최대 2회 자동 재렌더.
원인: Chrome headless 메모리 누적으로 Web Audio API 간헐적 실패.
**재렌더 시 public/ 에셋 자동 복구** (mp3, 커버, 배경 이미지).

### 9.5 Instrumental 곡 처리

txt 첫 줄이 `[Instrumental]`이면 빈 가사 배열 → 앨범아트 600px 중앙 레이아웃.

### 9.6 ffmpeg non-TTY 환경 무한루프 (해결됨)

**문제:** `ffmpeg -i file.mp4 -af volumedetect -f null /dev/null`에서 `-vn` 플래그 없이 실행하면 비디오 스트림까지 풀 디코드. 3분 23초 영상 기준 TTY에서 31초, non-TTY(nohup)에서는 무한루프 → 좀비 프로세스 (400%+ CPU, 7시간+ 방치).
**해결:** 모든 ffmpeg volumedetect 호출에 `-nostdin -vn` 추가.
- `-vn`: 비디오 스트림 무시 → 오디오만 처리 (1.2초 완료)
- `-nostdin`: non-TTY 환경에서 stdin 읽기 차단

```bash
# 올바른 사용법
ffmpeg -nostdin -vn -i "$file" -af volumedetect -f null /dev/null 2>&1
```

### 9.7 zsh glob 패턴 에러 (해결됨)

**문제:** `rm -f public/cover_${SAFE_STEM}.*`에서 매칭 파일이 없으면 zsh가 `no matches found` 에러 발생 → `set -e`에 의해 스크립트 종료.
**해결:** glob 패턴 대신 명시적 확장자 나열 (`*.png`, `*.jpg`, `*.jpeg`, `*.webp` 각각).

### 9.8 nohup 환경 불안정 (미해결 — 사용 보류)

**문제:**
- ffmpeg 무한루프 (9.6과 연관)
- 로그 파일 바이너리 손상 (tqdm 진행바 + ANSI escape)
- 일본어 파일명 로케일 깨짐 (`LANG/LC_ALL` 미설정)
- 프로세스 추적/kill 어려움 (고아 chrome-headless-shell 누적)

**현재 상태:** render_all_jpop.sh에 `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8` 설정 + 프로세스 그룹 격리로 개선했으나, 안정성 검증 전까지 직접 세션 실행 권장.

### 9.9 mlx-whisper 비결정적 출력

**문제:** 동일 입력 파일에 대해 실행마다 약간 다른 세그먼트/타이밍 생성.
**완화:** temperature를 0.0으로 고정하면 오히려 품질 저하. 기본 튜플 `(0.0, 0.2, 0.4, 0.6, 0.8, 1.0)`이 fallback을 통해 더 안정적인 결과 생성.
**영향:** 특정 곡에서 간혹 첫 가사 타이밍이 수초 늦게 감지됨 (예: vol1 20번곡 29.1초 시작). 수동 조정 필요한 경우 있음.

### 9.10 Remotion 번들 캐시 손상

**문제:** 첫 렌더 정상(-14dB) → 후속 렌더에서 무음(-91dB). Chrome headless 메모리 누적 추정.
**해결:** 배치 시작 시 캐시 자동 삭제 + `--bundle-cache=false`:
```bash
rm -rf node_modules/.cache /tmp/remotion-*
```

### 9.11 macOS 호환성 이슈 (해결됨)

| 이슈 | 원인 | 해결 |
|------|------|------|
| `setsid` 없음 | macOS에 미포함 | 서브셸 `( bash ... ) &`로 대체 |
| `timeout` 없음 | macOS에 미포함 | watchdog 서브프로세스로 대체 |
| `set -e` + `bc -l` 에러 | 산술 실패 시 스크립트 종료 | `awk "BEGIN{exit(...)}"` 패턴 |

---

## 10. YouTube 채널 설정

`youtube_upload.py`에 3개 채널 프리셋:

| 채널 | 이름 | 장르 |
|------|------|------|
| jpop | 季節のプレイリスト | J-POP/K-POP/City Pop |
| cafe | Hazy Roast | Cafe/Lofi/Chill |
| phonk | ZERO MERCY BEATS | Phonk/EDM/Workout |

---

## 11. 렌더 성능 기준

### 11.1 전체 볼륨 렌더 (2026-04-07 측정, faster-whisper 사용 시)

| 플레이리스트 | 곡 수 | 소요 시간 |
|---|---|---|
| hero-vol1 | 20 | 1시간 3분 |
| hero-vol2 | 20 | 1시간 7분 |
| hero-vol3 | 20 | 38분 |
| ref-vol1 | 20 | 50분 |
| ref-vol2 | 20 | 51분 |
| **합계** | **100** | **4시간 28분** |

곡당 평균 약 **2분 41초** (Mac M1 Pro, REMOTION_CONCURRENCY=2 기준)

### 11.2 Whisper STT 엔진 성능 비교 (2026-04-13 측정, medium 모델, 20곡 배치)

| 항목 | faster-whisper (CPU int8) | mlx-whisper (Apple Silicon) |
|------|---------------------------|------------------------------|
| 곡당 평균 시간 | ~42초 | ~9초 |
| 20곡 총 시간 | ~840초 (14분) | ~180초 (3분) |
| 속도 비율 | 1x (기준) | **4.5x 빠름** |
| 매칭 품질 | 기준 | 동등 (노이즈 필터 적용 후) |
| 노이즈 세그먼트 | 적음 | 많음 (필터로 제거) |
| VAD 지원 | O (vad_filter=True) | X → faster-whisper tiny 사용 |

> **참고**: mlx-whisper 전환으로 STT 단계에서 곡당 ~33초 절약. 100곡 기준 약 55분 단축 예상.

---

## 12. 핵심 파일 레퍼런스

| 파일 | 역할 | 줄 수 |
|------|------|-------|
| scripts/render.sh | 8단계 렌더 진입점 | ~530 |
| scripts/render_all_jpop.sh | 5볼륨 배치 렌더 (프로세스 격리) | ~168 |
| scripts/extract_lyrics.py | 가사 추출 엔진 (가장 복잡) | 1322 |
| scripts/translate_lyrics.py | Gemini 영어 번역 | ~108 |
| scripts/verify_all.py | 8항목 정밀 검증 | ~300 |
| scripts/generate_subtitles.py | 10개국 SRT 생성 | ~200 |
| scripts/youtube_upload.py | YouTube 업로드 | ~300 |
| src/Visualizer/Main.tsx | 메인 레이아웃 | ~170 |
| src/Visualizer/LyricsBox.tsx | 카라오케 가사 | ~234 |
| src/Visualizer/AlbumArt.tsx | 비트 반응 앨범아트 | ~88 |
| src/helpers/schema.ts | Props Zod 스키마 | ~68 |
| src/Root.tsx | 컴포지션 등록 | ~128 |

---

## 13. 트러블슈팅 체크리스트

### 렌더가 안 되거나 무음일 때

1. **Remotion 캐시 클리어**: `rm -rf node_modules/.cache /tmp/remotion-*`
2. **public/ 에셋 확인**: `ls public/{SAFE_STEM}.mp3 public/cover_{SAFE_STEM}.*`
3. **오디오 검증**: `ffmpeg -nostdin -vn -i output.mp4 -af volumedetect -f null /dev/null 2>&1 | grep mean_volume`
4. **좀비 프로세스 확인**: `ps aux | grep -E "chrome-headless|remotion|ffmpeg" | grep -v grep`
5. **좀비 정리**: `pkill -f "chrome-headless-shell.*suno-video"`

### 가사가 안 나오거나 타이밍이 이상할 때

1. **lyrics.json 확인**: `cat 01_songs/{SAFE_STEM}.lyrics.json | python3 -m json.tool | head -20`
2. **가사 재추출**: lyrics.json 삭제 후 render.sh 재실행 (2/8 단계부터 재시작)
3. **번역만 재실행**: `python3 scripts/translate_lyrics.py lyrics.json --force` (가사 텍스트는 유지, 번역만 갱신)
4. **수동 타이밍 조정**: lyrics.json 직접 편집 → start/end 값 수정 → render.sh 7/8 단계만 재실행

### 배치 렌더 중단 후 재개

1. `SKIP_EXISTING=1` (기본값)이므로 그냥 다시 실행하면 완료된 곡 자동 스킵
2. 실패 곡 확인: `cat /tmp/jpop_render_summary.log`
3. 특정 곡만 재렌더: `bash scripts/render.sh /path/to/specific_song.mp3`
