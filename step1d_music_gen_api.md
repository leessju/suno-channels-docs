# STEP 1D: Music-Gen API — MIDI/MP3 분석 → 가사/스타일/제목 생성

> Vertex AI(Gemini) 기반 음악 분석 + 콘텐츠 생성 파이프라인.  
> 실제 검증: `hero_1.mp3` → Lucid White 채널 3곡 생성 (2026-04-14 / 2026-04-15)  
> 전체 흐름: `step0_pipeline_overview.md` 참조.

---

## 1. 전체 흐름

```
[레퍼런스 파일 업로드]
  POST /api/music-gen/sessions/{id}/upload
       │
       ▼
[Vertex AI (Gemini) 분석]  ←  MP3: 멀티모달 오디오 분석
                            ←  MIDI: @tonejs/midi 로컬 파싱
  librosa BPM 보정 + ffprobe 재생시간 보정 (override)
       │
       ▼ (세션에 mediaAnalysis JSON 저장)
[콘텐츠 생성]
  POST /api/music-gen/generate
       │
       ▼
[출력]
  title_en / title_jp / lyrics / suno_style_prompts (5가지 variant) / narrative
```

---

## 2. 인증 방식

### 지원하는 3가지 계정 타입 (`config/accounts.json`)

```json
{
  "accounts": [
    {
      "type": "gemini-api",
      "name": "my-gemini-key",
      "apiKey": "AIza..."
    },
    {
      "type": "vertex-ai-apikey",
      "name": "my-vertex-apikey",
      "project": "my-gcp-project",
      "location": "us-central1",
      "apiKey": "AIza..."
    },
    {
      "type": "vertex-ai",
      "name": "my-vertex-sa",
      "project": "my-gcp-project",
      "location": "us-central1",
      "credentialsPath": "config/credentials_sa.json"
    }
  ]
}
```

| 타입 | 인증 방식 | 비고 |
|---|---|---|
| `gemini-api` | API Key | Gemini Developer API (generativelanguage.googleapis.com) |
| `vertex-ai-apikey` | **API Key** (OAuth 불필요) | Vertex AI + API Key 조합 |
| `vertex-ai` | Service Account (OAuth) | `google-auth-library` → Bearer 토큰 자동 갱신 |

**현재 사용 중: `vertex-ai-apikey` 또는 `gemini-api` (API Key 방식)**

### 단일 키 fallback (accounts.json 없을 때)

```bash
# .env
GEMINI_API_KEY=AIza...          # → gemini-api 타입으로 자동 처리
MUSIC_GEN_ACCOUNTS_PATH=config/accounts.json  # 멀티 계정 파일 경로
```

### Vertex AI 모델 폴백

Vertex AI에서 `gemini-3-flash-preview` 등 최신 프리뷰 모델은 404 반환.  
자동으로 아래와 같이 폴백됨:

| 요청 모델 | Vertex AI 실제 사용 모델 |
|---|---|
| `gemini-3-flash-preview` | `gemini-2.5-flash` |
| `gemini-3-pro-preview` | `gemini-2.5-pro` |

---

## 3. 사전 준비

### 채널 생성

```bash
POST http://localhost:3000/api/music-gen/channels
Content-Type: application/json

{
  "channel_name": "Lucid White",
  "system_prompt": "...(브랜드 DNA)...",
  "lyric_format": "jp_tagged",
  "forbidden_words": ["귀여운", "신나는"],
  "recommended_words": ["여백", "새벽", "서리"]
}
# → { "id": 9, ... }
```

### 세션 생성

```bash
POST http://localhost:3000/api/music-gen/sessions
Content-Type: application/json

{
  "channel_id": 9,
  "title": "hero_1.mp3 레퍼런스 세션",
  "constraints_json": "{\"mood\":\"차갑고 정적인\",\"vocal\":\"emotion-driven\"}"
}
# → { "id": "e2860a44-793c-440a-93aa-e0a7f341d0d9", ... }
```

---

## 4. MIDI / MP3 업로드 & 분석

```bash
POST http://localhost:3000/api/music-gen/sessions/{sessionId}/upload
Content-Type: multipart/form-data

file=@/path/to/reference.mp3
```

**제약**
- 허용 MIME: `audio/mpeg`, `audio/midi`, `audio/mid`
- 최대 크기: 20MB

### 실제 분석 출력 예시 (`hero_1.mp3` 기준)

```
기본 메타데이터
BPM: 80.7 | 4/4 | F Major | 에너지: 3/10
악기: Acoustic Piano (단독)

리듬·밀도
음절/마디: 5~10 — 느린 BPM, 여백 많은 피아노 → 긴 호흡 가사 적합

구조 (5섹션)
Intro 에너지 2/10 → Verse 3/10 → Pre-Chorus → Chorus 4/10 → Outro 2/10

화성
버스:   Fmaj7 → Am7 → Bbmaj7 → C9sus4
코러스: Fmaj7 → Am7 → Bbmaj7 → C7 → Dm7 → Am7 → Bbmaj7 → Gm7/C
감정: 차분한 성찰 → 잔잔한 우수 유지

lyric_note 예시
Verse 1: 창가에 / 비친 / 너의 / 뒷모습 / 보며
Verse 2: 조용히 / 내려앉은 / 기억의 / 조각들
```

### JSON 응답 구조

```json
{
  "success": true,
  "data": {
    "sessionId": "e2860a44-793c-440a-93aa-e0a7f341d0d9",
    "mediaAnalysis": {
      "key": "F Major",
      "tempo_bpm": 80.7,
      "duration_seconds": 183,
      "time_signature": "4/4",
      "energy_level": 3,
      "instrumentation": ["Acoustic Piano"],
      "song_sections": [
        { "name": "Intro",       "start_time": 0,   "end_time": 12,  "energy": 2 },
        { "name": "Verse",       "start_time": 12,  "end_time": 48,  "energy": 3 },
        { "name": "Pre-Chorus",  "start_time": 48,  "end_time": 64,  "energy": 4 },
        { "name": "Chorus",      "start_time": 64,  "end_time": 106, "energy": 4 },
        { "name": "Outro",       "start_time": 160, "end_time": 183, "energy": 2 }
      ],
      "syllables_per_bar_min": 5,
      "syllables_per_bar_max": 10,
      "lyric_density_recommendation": "느린 BPM과 여백 많은 피아노 → 긴 호흡 가사 적합",
      "chord_progression": ["Fmaj7", "Am7", "Bbmaj7", "C9sus4"],
      "chord_progression_chorus": ["Fmaj7", "Am7", "Bbmaj7", "C7", "Dm7", "Am7", "Bbmaj7", "Gm7/C"],
      "chord_progression_confidence": 0.85,
      "mood": ["Melancholic", "Peaceful", "Reflective"],
      "emotional_keywords": ["그리움", "새벽", "회상", "위로"],
      "vocal_recommendation": "Mid-low, 감성적 호소, 여백 중시",
      "lyric_structure_guide": [
        {
          "section_name": "Intro",
          "time_range": "0:00 ~ 0:12",
          "energy": 2,
          "harmonic_character": "Fmaj7 pedal tone, 공간감 최대",
          "syllables_per_bar": "3~5",
          "lyric_style": "독백/배경 설명",
          "lyric_note": "冬の朝, 白い息, 消えてゆく — 짧은 이미지 단편"
        },
        {
          "section_name": "Verse",
          "time_range": "0:12 ~ 0:48",
          "energy": 3,
          "harmonic_character": "Fmaj7→Am7 서정적 하강 진행",
          "syllables_per_bar": "5~8",
          "lyric_style": "감정 변화/기대감",
          "lyric_note": "창가에 / 비친 / 너의 / 뒷모습 / 보며 — 호흡점 준수"
        },
        {
          "section_name": "Chorus",
          "time_range": "1:04 ~ 1:46",
          "energy": 4,
          "harmonic_character": "Fmaj7→Dm7 풀 보이싱, 감정 고조",
          "syllables_per_bar": "6~10",
          "lyric_style": "핵심 주제/감정 분출",
          "lyric_note": "조용히 / 내려앉은 / 기억의 / 조각들 — 고음 선율 맞춤"
        }
      ]
    }
  }
}
```

### 분석 엔진 세부 동작

| 처리 단계 | 방법 | 비고 |
|---|---|---|
| BPM 측정 | `librosa.beat.beat_track` (로컬 Python) | Ground-truth로 Gemini 값 override |
| 재생 시간 | `ffprobe` (로컬) | 정밀 측정 |
| 코드 진행 | Gemini 멀티모달 | 재즈 보이싱 (Fmaj7, C9sus4 등) |
| 구조 분석 | Gemini 멀티모달 | 섹션별 에너지/특성 포함 |
| 음절 권장값 | Gemini (BPM + 음표 밀도 기반) | `syllables_per_bar_min/max` |
| `lyric_structure_guide` | Gemini | 섹션별 가사 작성 지침 (핵심 출력) |
| 환각 방지 | Anti-hallucination 시스템 프롬프트 | Double-time 오류, 없는 악기 생성 차단 |

**MP3** → Gemini 멀티모달 분석 → librosa/ffprobe 보정  
**MIDI** → `@tonejs/midi` 로컬 파싱 → BPM/코드 확정값 (Gemini 멀티모달 미호출)

---

## 5. 콘텐츠 생성 (가사 + 제목 + Suno 스타일)

```bash
POST http://localhost:3000/api/music-gen/generate
Content-Type: application/json

{
  "channel_id": 9,
  "session_id": "e2860a44-793c-440a-93aa-e0a7f341d0d9",
  "emotion_input": "차가운 새벽, 유리창 너머 가로등"
}
```

- `session_id` 생략 시: 해당 채널의 가장 최근 활성 세션의 mediaAnalysis 자동 사용
- `emotion_input` 생략 시: Gemini가 분석 데이터 기반으로 테마 자체 결정

### 분석 → 생성 프롬프트 3단계 매핑

```
① BPM 80.7 + syllables 5~10
   → "권장 음절/마디: 5~10 — 이 범위를 초과하면 곡의 여백이 무너집니다"

② 코드 Fmaj7→Am7→Bbmaj7→C9sus4
   → "버스 코드: Fmaj7 → Am7 → Bbmaj7 → C9sus4" (감정선 설계 기준)

③ lyric_structure_guide 섹션별
   → [Intro]      독백/배경설명, 3~5음절
   → [Verse]      감정변화/기대감, 5~8음절
   → [Pre-Chorus] 기대감 고조, 4~6음절
   → [Chorus]     핵심주제/감정분출, 6~10음절
   → [Outro]      여운, 3~4음절
```

### 실제 생성 결과 예시 (hero_1.mp3 기준)

3회 테스트에서 생성된 6곡:

| 테스트 | 곡 | title_en | title_jp |
|---|---|---|---|
| test-01 | 곡 1 | Blue Afterimage | 青い残像 |
| test-01 | 곡 2 | Silver Morning | 銀色の朝 |
| test-01 | 곡 3 | Traces on the Glass | 硝子の輪郭 |
| test-02 | 곡 1 | Margins of a Pale Morning | 褪せた朝の余白 |
| test-02 | 곡 2 | Azure Resonance | 蒼い残響 |
| test-02 | 곡 3 | Frostwork Traces | 霜の花の跡 |

→ 전체 가사 및 스타일: `docs/lucid-white-lyrics-test-01.md`, `docs/lucid-white-lyrics-test-02.md`

### 응답 구조

```json
{
  "success": true,
  "data": {
    "content": {
      "id": 42,
      "emotion_theme": "새벽 유리창의 서리 — 차갑지만 아름다운 정지",
      "title_en": "Blue Afterimage",
      "title_jp": "青い残像",
      "lyrics": "[Intro]\n冬の朝,\n白い吐息,\n消えてゆく\n\n[Verse 1]\n...",
      "narrative": "새벽 유리창의 정지된 순간...",
      "suno_style_prompts": [
        "High-fidelity Cinematic Ambient, Neo-Soul blend, soft breathy female vocals, 80BPM, analog saturation",
        "90s Vintage J-Pop aesthetic, Lo-fi Hip-hop undertones, intimate male vocals, Rhodes piano, 80BPM",
        "Minimalist Acoustic Piano Ballad, emotional breathy vocals, sparse arrangement, 80BPM, deep plate reverb",
        "Contemporary Jazz Pop, soft brush drums, warm upright bass, intimate breathy vocals, 80BPM",
        "Ethereal Dream Pop textures, hushed female vocals, 80BPM, airy synth pads"
      ],
      "total_duration_sec": 220
    },
    "channel_id": 9
  }
}
```

### 생성 검증 3단계 (generator.ts)

1. **스키마 검증** (zod) — `title_en`, `lyrics`, `suno_style_prompts` 필드 존재 여부
2. **금지어 검사** — `channels.forbidden_words` 기준
3. **가사 형식 검증** — `jp_tagged`: [Verse]/[Chorus] 태그 2개 이상, 가사 4줄 이상

→ 실패 시 최대 3회 자동 재시도 (이전 실패 이유 포함해 재생성 요청)

---

## 6. 전체 API 엔드포인트 목록

| Method | Endpoint | 설명 |
|---|---|---|
| POST | `/api/music-gen/channels` | 채널 생성 |
| GET | `/api/music-gen/channels` | 채널 목록 |
| GET | `/api/music-gen/channels/{id}` | 채널 상세 |
| POST | `/api/music-gen/sessions` | 세션 생성 |
| GET | `/api/music-gen/sessions/{id}` | 세션 상세 |
| **POST** | **`/api/music-gen/sessions/{id}/upload`** | **MIDI/MP3 업로드 + Vertex AI 분석** |
| POST | `/api/music-gen/sessions/{id}/chat` | 대화형 가사 협업 (분석 결과 컨텍스트 포함) |
| GET | `/api/music-gen/sessions/{id}/messages` | 메시지 목록 |
| **POST** | **`/api/music-gen/generate`** | **가사/제목/스타일 5종 생성** |
| GET | `/api/music-gen/contents` | 생성된 콘텐츠 목록 |

---

## 7. 실행 전제 조건

```bash
# .env 필수 항목
GEMINI_API_KEY=AIza...                        # API Key 방식 (단일 계정)
# 또는
MUSIC_GEN_ACCOUNTS_PATH=config/accounts.json  # 멀티 계정 (vertex-ai-apikey 등)
GEMINI_MODEL=gemini-2.5-flash                 # 기본 모델
MUSIC_GEN_POLISH=1                            # (선택) gemini-2.5-pro로 최종 다듬기

# 로컬 의존성 (BPM/시간 정밀 보정)
pip install librosa       # Python — BPM ground-truth
brew install ffmpeg       # ffprobe — 재생시간 정밀 측정

# 서버 실행
pnpm dev
```

---

## 8. 시드 및 테스트

```bash
# Lucid White 채널 초기화
npx tsx scripts/seed-channels.ts

# E2E 테스트 (채널→세션→업로드→생성 전 과정)
bash scripts/test-music-gen.sh
```

---

## 9. 연결 문서

| 문서 | 내용 |
|---|---|
| `step1_suno_generation_guide.md` | Suno API 엔드포인트 전체 목록, cover/upload 파라미터 |
| `step1c_vertex_ai_auth.md` | Vertex AI 인증 설정 (service account vs API key) |
| `step1a_midi_cover_research.md` | MIDI → cover 생성 파이프라인 연구 |
| `docs/lucid-white-lyrics-test-01.md` | hero_1.mp3 기반 1차 생성 결과 (3곡) |
| `docs/lucid-white-lyrics-test-02.md` | hero_1.mp3 기반 2차 생성 결과 (3곡) |
