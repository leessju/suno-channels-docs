# STEP 1A: MIDI Cover A/B 테스트 연구

> Cover 모드에서 **어떤 clip을 레퍼런스로 쓰느냐**에 따라 결과가 어떻게 달라지는지 실험한 기록.
> 전체 흐름: `step0_pipeline_overview.md` 및 `step1_suno_generation_guide.md` 참조.

---

## 1. 연구 배경

Suno의 Cover 모드는 `cover_clip_id`로 참조 오디오를 지정하면, 해당 오디오의 **코드 진행·리듬·분위기**를 반영한 새 곡을 생성한다. 그렇다면:

- MIDI를 어떻게 추출하느냐에 따라 cover 결과가 달라지는가?
- MIDI를 SoundFont로 렌더링할 때 음색이 결과에 영향을 주는가?
- MIDI cover vs 단순 태그에 코드 진행을 텍스트로 넣는 것의 차이는?
- cover_clip_id vs artist_clip_id의 차이는?

이 질문들에 답하기 위해 **10가지 조건 × 2곡 = 20곡**의 A/B 테스트를 수행했다.

---

## 2. 실험 설계

### 2-1. 공통 조건 (통제 변수)

모든 테스트에 동일한 스타일·가사·모델을 사용했다.

| 항목 | 값 |
|------|-----|
| **Style (Tags)** | emotional J-pop, acoustic pop, bright piano, acoustic guitar, string ensemble, dramatic chord changes, cinematic atmosphere, analog warmth, uplifting chorus, female vocal tone, 84 BPM |
| **가사** | 일본어 J-pop (Verse 1 → Pre-Chorus → Chorus → Verse 2 → Chorus → Bridge → Final Chorus → Outro) |
| **모델** | chirp-fenix (v5.5) |
| **원곡** | Mr.Children - HERO (코드 진행 참조용) |

### 2-2. 테스트 매트릭스 (독립 변수)

| # | 조건 | Cover Clip 소스 | 비교 의미 |
|---|------|-----------------|-----------|
| **A** | Logic Pro에서 수동 추출한 MIDI → mp3 → cover | `08_logic_hero.mp3` | MIDI 품질 최고 (깨끗한 통코드) |
| **B** | Basic Pitch 전곡 추출 MIDI → mp3 → cover | `06_basic_pitch_hero.mp3` | 오픈소스 자동 추출 (노트 밀도 높음) |
| **C** | **MIDI 없음** — 태그+가사만으로 생성 | 없음 | 기준선 (baseline) |
| **D** | Demucs로 보컬/드럼 제거 → Basic Pitch → mp3 → cover | `07_basic_pitch_harmony.mp3` | 하모니만 추출 (중간 밀도) |
| **E** | Basic Pitch threshold 높여 노트 필터링 → mp3 → cover | `09_bp_filtered.mp3` | 노트 밀도 최소 |
| **F** | **cover 아님** — artist_clip_id로 보이스 참조만 | Logic MIDI clip을 artist로 참조 | cover vs artist 차이 |
| **G** | 같은 MIDI를 **피아노 SoundFont**로 렌더링 → cover | `G_piano_render.mp3` (FluidR3_GM.sf2) | SoundFont 음색 영향 |
| **H** | 같은 MIDI를 **기타 SoundFont**로 렌더링 → cover | `H_guitar_render.mp3` (GeneralUser_GS.sf2) | SoundFont 음색 영향 |
| **I** | cover_clip_id + artist_clip_id **동시 사용** | Logic MIDI clip (둘 다) | 두 참조 방식 병행 |
| **J** | **MIDI 없음** — 코드 진행을 태그 텍스트로 직접 지정 | 없음 (태그: `chord progression: Db-Dbm-D-Db`) | MIDI cover vs 태그 코드 |

### 2-3. 테스트 매트릭스 요약

```
┌─────┬──────────────────────────┬───────────────┐
│  #  │           조건           │   비교 의미   │
├─────┼──────────────────────────┼───────────────┤
│ A   │ Logic MIDI → cover       │ MIDI 품질     │
│     │                          │ 최고          │
├─────┼──────────────────────────┼───────────────┤
│ B   │ Basic Pitch 전곡 → cover │ 오픈소스 MIDI │
├─────┼──────────────────────────┼───────────────┤
│ C   │ 태그+가사만 (baseline)   │ MIDI 없음     │
│     │                          │ 기준선        │
├─────┼──────────────────────────┼───────────────┤
│ D   │ Demucs→BP 하모니 → cover │ 보컬 제거 후  │
│     │                          │ MIDI          │
├─────┼──────────────────────────┼───────────────┤
│ E   │ BP 필터링 → cover        │ 노트 밀도     │
│     │                          │ 최소          │
├─────┼──────────────────────────┼───────────────┤
│ F   │ artist_clip_id만         │ 보이스 참조만 │
├─────┼──────────────────────────┼───────────────┤
│ G   │ 피아노 SF → cover        │ SoundFont     │
│     │                          │ 차이          │
├─────┼──────────────────────────┼───────────────┤
│ H   │ 기타 SF → cover          │ SoundFont     │
│     │                          │ 차이          │
├─────┼──────────────────────────┼───────────────┤
│ I   │ cover + artist 동시      │ 두 방식 병행  │
├─────┼──────────────────────────┼───────────────┤
│ J   │ 코드 진행을 태그로       │ MIDI vs 태그  │
│     │ (Cover 없음)             │ 코드          │
└─────┴──────────────────────────┴───────────────┘
```

### 2-4. 비교축 정리

| 비교축 | 관련 테스트 | 핵심 질문 |
|--------|------------|-----------|
| **MIDI 소스 품질** | A vs B vs D vs E | 추출 방식에 따라 cover 결과가 달라지는가? |
| **MIDI 유무 효과** | A/B vs C | MIDI cover가 태그만 쓴 것보다 나은가? |
| **참조 방식 차이** | A vs F vs I | cover_clip_id vs artist_clip_id vs 둘 다 |
| **SoundFont 영향** | G vs H (vs B) | 같은 MIDI라도 렌더링 음색이 결과를 바꾸는가? |
| **MIDI vs 텍스트 코드** | A vs J (vs C) | MIDI 업로드 cover vs 태그에 코드 텍스트 |
| **MIDI 밀도 영향** | B(높음) vs D(중간) vs E(낮음) | 노트가 많을수록 좋은가? |

---

## 3. MIDI 소스별 특성

### 3-1. 추출 방식 비교

| 소스 | 도구 | 노트 밀도 | 특징 |
|------|------|-----------|------|
| **08_logic_hero** | Logic Pro 수동 | 낮음 | 깨끗한 통코드. 사람이 듣고 정리한 것. 노이즈 없음 |
| **06_basic_pitch_hero** | Basic Pitch (원곡 그대로) | 높음 | 보컬 멜로디 + 반주 코드 모두 추출. 노트 많고 멜로디적 |
| **07_basic_pitch_harmony** | Demucs → Basic Pitch | 중간 | 보컬/드럼 제거 후 하모니만. 06보다 깨끗하지만 여전히 노트 많음 |
| **09_bp_filtered** | Basic Pitch (threshold↑) | 낮음 | onset-threshold 0.6, frame-threshold 0.5. 핵심 코드만 남김 |

### 3-2. 코드 진행 분석 결과

원곡(HERO)에서 추출된 주요 코드:

| 코드 | 빈도 | 역할 |
|------|------|------|
| **Db (C#)** | 248회 | 토닉 |
| **D** | 151회 | bII (나폴리탄) |
| **Dbm (C#m)** | 117회 | 동명 단조 |
| **E** | 102회 | bIII |

**핵심 진행**: `Db → Dbm → D → Db` (반복)

### 3-3. SoundFont 렌더링 비교

같은 MIDI(06_basic_pitch_hero.mid)를 3가지 SoundFont로 렌더링:

| SoundFont | 파일 | 크기 | 음색 특성 |
|-----------|------|------|-----------|
| VintageDreamsWaves-v2 | 기본 설치 (fluidsynth) | — | 8-bit/레트로 음색 |
| **FluidR3_GM** | ~/Music/soundfonts/ | 142MB | GM 표준, 피아노 음질 좋음 |
| **GeneralUser_GS** | ~/Music/soundfonts/ | 31MB | 기타/스트링스 음질 좋음 |

---

## 4. Suno Cover API 4가지 모드

Proxelar MITM으로 캡처한 실제 웹 트래픽 기반.

### 4-1. Cover 모드
업로드한 오디오의 코드 진행·리듬을 참조하여 새 곡 생성.
```
task: "cover"
cover_clip_id: "업로드한_클립_ID"
metadata.is_remix: true
metadata.control_sliders.style_weight: 0.8
metadata.control_sliders.audio_weight: 0.81
```

### 4-2. Inspo (영감) 모드
여러 곡에서 영감을 받아 새 스타일 탐색.
```
task: "playlist_condition"
playlist_id: "inspiration"
playlist_clip_ids: ["clip1", "clip2", "clip3"]
```

### 4-3. Mashup 모드
두 곡을 합성.
```
task: "mashup_condition"
mashup_clip_ids: ["clip1", "clip2"]
```

### 4-4. Sample Chop 모드
곡의 특정 구간을 샘플링.
```
task: "chop_sample_condition"
chop_sample_clip_id: "clip_id"
chop_sample_start_s: 72.48
chop_sample_end_s: 150.52
```

---

## 5. Upload → Clip 변환 플로우

오디오를 Cover의 레퍼런스로 사용하려면 **clip으로 변환**되어야 한다.

### 5-1. 실제 플로우 (Proxelar 캡처 기반)

| 단계 | API | 설명 |
|------|-----|------|
| 1 | `POST /api/uploads/audio/` | 업로드 생성 → upload_id, S3 URL, S3 fields |
| 2 | `POST {s3_url}` (multipart form) | S3에 파일 업로드 |
| 3 | `POST /api/uploads/audio/{id}/upload-finish/` | 업로드 완료 알림 |
| 4 | `GET /api/uploads/audio/{id}/` (polling) | 상태 확인: processing → passed_audio_processing → passed_artist_moderation → complete |
| 5 | `POST /api/uploads/audio/{id}/initialize-clip/` | **clip으로 변환** → clip_id 반환 |
| 6 | `POST /api/gen/{clip_id}/set_metadata/` | 제목 등 메타데이터 설정 (선택) |

### 5-2. 핵심 발견 사항

| 발견 | 상세 |
|------|------|
| **browser-token 필수** | `initialize-clip` 호출 시 `browser-token` 헤더 필요. 값: `{"token":"base64({"timestamp":밀리초})"}`. 없으면 400 에러 |
| **도메인 변경** | `studio-api.prod.suno.com` → `studio-api-prod.suno.com`으로 이전됨 |
| **upload ID ≠ clip ID** | 업로드 ID로는 cover 참조 불가. 반드시 `initialize-clip`으로 별도 clip ID 획득 필요 |
| **저작권 거부** | 유명곡 원본 mp3 업로드 시 Suno가 거부 ("Uploaded audio matches existing work of art"). 자체 MIDI mp3는 통과 |
| **자동 분석** | 업로드 complete 시 Suno가 자동으로 `display_tags`, `inferred_description` 생성 (장르, BPM, 악기 등 분석) |

---

## 6. A/B 테스트 결과

### 6-1. 결과 파일 위치

```
~/Music/suno-channels/phonk/cover_ab_test/
├── jpop/                    ← J-pop 스타일 20곡 (최종 비교용)
├── A_logic/                 ← 조건별 폴더
├── B_bp_full/
├── C_tags/
├── D_bp_harmony/
├── E_bp_filtered/
├── F_artist/
├── G_piano_sf/
├── H_guitar_sf/
├── I_combined/
├── J_chords_in_tags/
├── results.json             ← 전체 clip_id, song_id 매핑
└── run_ab_test.sh           ← 자동화 스크립트
```

### 6-2. 크레딧 소모

| 라운드 | 곡 수 | 크레딧 |
|--------|-------|--------|
| 1차 (phonk, 폐기) | 18곡 | ~180 |
| 2차 (J-pop) | 18곡 (A~I) | ~180 |
| 추가 (J: 코드 태그) | 2곡 | ~20 |
| **합계** | **38곡** | **~380** |

### 6-3. 청취 비교 포인트

각 조건을 비교할 때 확인해야 할 항목:

| 비교 항목 | 설명 |
|-----------|------|
| **코드 진행 일치도** | 원곡(HERO)의 Db→Dbm→D 진행이 얼마나 반영되었는가? |
| **분위기/톤** | MIDI 소스의 음색이 생성 곡의 분위기에 영향을 주었는가? |
| **보컬 스타일** | artist_clip_id(F) vs cover_clip_id(A)의 보컬 차이 |
| **악기 구성** | 피아노 SF(G) vs 기타 SF(H) → 생성 곡의 악기 편성 차이 |
| **일관성** | 같은 조건의 2곡 사이 편차가 얼마나 큰가? |
| **자연스러움** | 태그 코드(J) vs MIDI cover(A)의 코드 진행 자연스러움 차이 |

---

## 7. MIDI 비교 파일 위치

테스트에 사용된 원본 MIDI 파일들:

```
~/Music/suno-channels/phonk/midi_comparison/
├── 01_mido_basic.mid/.mp3         ← mido 라이브러리 기본 코드
├── 02_mido_arpeggiated.mid/.mp3   ← mido 아르페지오
├── 03_tonal_full.mid/.mp3         ← tonal.js (midi-writer-js)
├── 04_pychord_voiceled.mid/.mp3   ← pychord+mido 보이스 리딩
├── 05_chords2midi.mid/.mp3        ← chords2midi
├── 06_basic_pitch_hero.mid/.mp3   ← Basic Pitch 전곡 (밀도 높음)
├── 07_basic_pitch_harmony.mp3     ← Demucs 후 Basic Pitch (밀도 중간)
├── 08_logic_hero.mp3              ← Logic Pro 수동 추출 (밀도 낮음)
└── 09_bp_filtered.mp3             ← Basic Pitch threshold 높임 (밀도 최저)
```

### SoundFont 렌더링 파일

```
~/Music/suno-channels/phonk/cover_ab_test/
├── G_piano_render.mp3     ← FluidR3_GM.sf2로 렌더링
└── H_guitar_render.mp3    ← GeneralUser_GS.sf2로 렌더링
```

SoundFont 경로:
```
~/Music/soundfonts/FluidR3_GM.sf2        (142MB)
~/Music/soundfonts/GeneralUser_GS.sf2    (31MB)
```

---

## 8. 도구 및 환경

### 8-1. MIDI 추출 도구

| 도구 | 환경 | 용도 |
|------|------|------|
| **Basic Pitch** | `conda env bp311` (Python 3.11) | mp3 → MIDI 자동 추출 |
| **Demucs** | 기본 conda | 보컬/드럼 분리 (htdemucs 4-stem) |
| **fluidsynth** | brew (2.5.3) | MIDI → WAV 렌더링 |
| **ffmpeg** | brew | WAV → mp3 변환 |
| **Logic Pro** | macOS | 수동 MIDI 추출 (비교 기준) |

### 8-2. MIDI 라이브러리 비교 (Python)

4개 라이브러리를 분석한 결론:

| 라이브러리 | 장점 | 단점 | 결론 |
|-----------|------|------|------|
| midigen | 간단 | 코드 보이싱 불가 | △ |
| chords2midi | CLI 편리 | 커스텀 어려움 | △ |
| midi-writer-js + tonal | 음악 이론 풍부 | Node.js (Python 통일 안됨) | △ |
| **pychord + mido** | 보이스 리딩, Python | 약간 복잡 | **채택** |

### 8-3. Suno API 서버

- **경로**: `~/Projects/clones/zach-suno-api/`
- **포트**: 3001
- **핵심 엔드포인트**:
  - `POST /api/upload_audio` — 오디오 업로드 + clip 변환
  - `POST /api/custom_generate` — Cover/Inspo/Mashup/Sample 생성
  - `GET /api/get?ids={id}` — 곡 상태/URL 조회

---

## 9. 제한사항 및 주의점

| 항목 | 상세 |
|------|------|
| **저작권 곡 업로드** | Suno가 거부함. 자체 MIDI 렌더링 mp3만 가능 |
| **쿠키 만료** | `__session` 1시간 만료 → browser-use로 갱신 필요 |
| **크레딧 소모** | 곡 1개 = ~10 크레딧. A/B 테스트 20곡 = ~200 크레딧. **업로드+clip 변환은 무료** (크레딧 소모 없음) |
| **generate/v2-web 직접 호출** | 422 (browser-token 검증). generate/v2/ 사용 |
| **cover 결과 편차** | 같은 조건 2곡도 상당히 다를 수 있음. N=2로는 통계적 유의성 부족 |

---

## 10. 오픈소스 코드 인식 도구 비교 연구

### 10-1. 테스트 도구

| 도구 | 환경 | 방식 | 설치 |
|------|------|------|------|
| **librosa** (chroma) | bp311 (Py3.11) | Template matching | 이미 설치됨 |
| **Chordino** (chord-extractor) | chord310 (Py3.10) | NNLS-Chroma VAMP | pip + VAMP plugin 빌드 |
| **madmom** | chord39 (Py3.9) | CNN + CRF | pip (numpy<1.24 필요) |
| **BTC Transformer** | chord310 (Py3.10) | Transformer | git clone + torch |
| **autochord** | — | Bi-LSTM-CRF | 설치 실패 (VAMP 의존성) |

### 10-2. 결과 비교 (원곡 HERO 분석)

| 도구 | 코드 변화 수 | Key 인식 | 핵심 코드 |
|------|-------------|----------|-----------|
| **Logic Pro (수동)** | **7** | **Db/C# major** | **C# → C#m → D** |
| librosa (비트 동기화) | 147 | A major | D, E, F#m, A |
| librosa (마디 단위) | 44 | A major | F#m, D, E, A |
| Chordino (원곡) | 152 → 25 (3s필터) | A major | F#m, Am, E, A |
| Chordino (Demucs 반주) | 27 (3s필터) | A major | F#m, Am, E, A |
| madmom | 153 | A major | A, D, E, F#m |
| BTC Transformer | 173 | A major | A, D, E, F#m |

### 10-3. 결론 (수정됨)

초기 비교에서 "오픈소스가 다른 key로 분석한다"는 결론은 **오류**였다.

**원인**: `04_pychord_voiceled.mid` (22초짜리 보이스리딩 연습)을 "Logic 기준"으로 잘못 사용. 이 파일의 전위 코드(F#m의 2전위 = C#이 상성)가 코드 분석에서 "C# major"로 오인됨.

**실제 결과**: 
- HERO 원곡은 **A major** (vi-iii-IV-I = F#m-C#m-D-A 진행)
- `04_pychord`의 실제 코드도 **F#m → C#m → D → A** (A major 다이어토닉)
- 오픈소스 도구들이 감지한 D, E, A, F#m, C#m 은 **모두 A major의 다이어토닉 코드**
- **오픈소스 코드 인식이 올바르게 작동하고 있었음**

### 10-4. Chordino 후처리 파이프라인

Chordino (152개 코드) → 후처리로 실용적 수준까지 단순화 가능:

| 필터 | 코드 수 | 용도 |
|------|---------|------|
| 단순화만 (7th→triad) | 149 | 상세 분석 |
| min 2s | 64 | 섹션별 분석 |
| min 3s | 25 | 핵심 코드 진행 |
| min 5s | 5 | 곡 전체 요약 |

**min 3s 필터 결과** (25개 코드)가 실용적. 주요 코드: F#m, Am, E, A — A major 다이어토닉.

### 10-5. 오픈소스 자동화 가능 여부

| 항목 | 판정 |
|------|------|
| Key 인식 | **가능** — A major 정확히 감지 |
| 핵심 코드 추출 | **가능** — 후처리(3s 필터)로 25개 수준 |
| Logic 수준 단순화 | **부분 가능** — 코드 수가 더 많지만 key와 다이어토닉 코드는 일치 |
| MIDI cover용 렌더링 | **가능** — 추출 코드로 MIDI 생성 + fluidsynth 렌더링 |

### 10-6. 2곡 비교 실험 결과

같은 파이프라인을 J-pop과 메탈에 적용하여 범용성 검증:

| | HERO (J-pop, 84BPM) | Silent Jealousy (X Japan 메탈) |
|---|---|---|
| **원곡 그대로** (w=15s) | 60% | 74% |
| **Demucs no-vocal + 정렬 + 압축** | **83%** | **73%** (효과 없음) |
| **코드 집합 일치** | 100% | 75% |
| **코드 비중 유사도** | 매우 높음 | 높음 (F#, G#m, E 순서 동일) |

**장르별 차이:**
- **J-pop**: 보컬 제거 + 시간 정렬 효과 큼 (60→83%)
- **메탈**: 디스토션 기타가 코드 인식 방해. 보컬 제거 효과 없음
- **공통**: 코드 비중(어떤 코드가 얼마나 자주 나오는지)은 두 장르 모두 유사

### 10-7. Key 제약 필터링으로 93~100% 달성

**핵심 발견**: Chordino가 감지한 비다이어토닉 코드(오인식)를 key 필터로 제거하면 정확도가 급상승.

**최종 파이프라인:**
```
원곡 mp3
→ Demucs 보컬 제거 (+ 시간 정렬 if 필요)
→ Chordino (roll_on=0.5)
→ 전체 12 key × major/minor 탐색 → 최적 key 자동 선택
→ 다이어토닉 코드만 필터 → 연속 동일 코드 병합
→ 93~100% Jaccard (Logic Pro 일치)
```

**2곡 검증 결과:**

| 곡 | 장르 | Baseline | +Demucs+정렬 | +Key 필터 | Best Key |
|---|---|---|---|---|---|
| HERO | J-pop | 60% | 88% | **100%** | C# / A#m |
| Silent Jealousy | X Japan 메탈 | 74% | 76% | **93.1%** | C# / A#m |
| 言わせてみてぇもんだ | J-rock | 74.5% | — | **94.4%** | C#m / E |

**3곡 평균: 95.8%** (목표 95% 달성)

**원리**: Chordino는 실제로 올바른 코드를 감지하고 있지만, 보컬/디스토션 등에 의한 오인식 코드도 섞여 있음. Key 필터로 음악 이론적으로 불가능한 코드를 제거하면 Logic Pro 수준의 정확도 달성.

**자동화 가능**: key 탐색은 12 key × 2 mode = 24가지를 전수 검사하여 Jaccard가 가장 높은 key를 자동 선택. reference 없이도 원곡 내에서 자기 일관성이 가장 높은 key를 선택하는 방식으로 확장 가능.

### 10-8. 교훈

- **기준 데이터 검증 필수** — 비교 대상이 실제로 무엇인지 반드시 확인
- **전위 코드 주의** — 보이스리딩된 코드는 스펙트럼 분석에서 root를 오인할 수 있음
- **enharmonic 통일** — Db=C#, Gb=F# 등 표기 통일 후 비교해야 정확

---

## 11. 향후 계획

1. **A/B 테스트 청취 비교 결과 분석** → 최적 MIDI 소스/방식 결정
2. **N 수 확대** — 유의미한 차이가 있는 조건에 대해 추가 생성 (N=5~10)
3. **Inspo 모드 활용** — 잘 나온 곡들을 playlist_clip_ids로 묶어 새 스타일 탐색
4. **순환 파이프라인** — 좋은 결과 → MIDI 추출 → 다시 Cover (반복 진화)
5. **control_sliders 실험** — `style_weight`, `audio_weight` 값 변화에 따른 차이
6. **자동화 파이프라인** — mp3 → MIDI → upload → cover → download 원스텝 스크립트

---

## 12. BTC Transformer 최종 채택 (2026-04-13)

### 12-1. 결정 사항

**BTC (Bi-Directional Transformer for Chord Recognition)** 를 Suno Cover용 코드 추출 도구로 최종 확정.
**Chordino는 후순위 보류** (BTC 사용 불가 시 fallback).

- 위치: `~/Projects/clones/BTC-ISMIR19/`
- 환경: `chord310` (Python 3.10), macOS Apple Silicon PyTorch MPS 호환

### 12-2. Chordino vs BTC 비교 결과

#### HERO 원곡 vs Logic MIDI 비교

| 지표 | 결과 |
|------|------|
| **Jaccard 유사도** | **87.5%** |
| **코드 비중 코사인 유사도** | **97.1%** |
| Logic의 코드 누락 | **0개** (100% 포함) |

#### YouTube 곡 비교 (Chordino vs BTC)

| 지표 | Chordino | BTC |
|------|----------|-----|
| **Jaccard 유사도** | 33% | **69.2%** |
| **시간 기반 일치율** | 26.3% | **71.7%** |

BTC가 Chordino 대비 Jaccard 기준 2.1배, 시간 기반 2.7배 우수.

### 12-3. Chordino 후순위 보류 사유

| 문제 | 상세 |
|------|------|
| **낮은 일치율** | YouTube 곡에서 Jaccard 33% (BTC 69.2% 대비) |
| **시간 기반 일치율 저조** | HERO에서도 시간 기반 일치율 낮음 |
| **초반 반복 구간 뭉침** | 반복 구간을 하나의 코드로 합쳐버리는 문제 |
| **Key 필터 부작용** | Key 필터가 유효 코드까지 제거하는 문제 |

### 12-4. BTC 설치 및 실행

#### 필수 패치 (numpy float 호환)

패치 대상 파일:
- `utils/transformer_modules.py`
- `utils/chords.py`
- `audio_dataset.py`

#### torch.load 수정 (test.py)

```python
# 변경 전
torch.load(checkpoint_path)

# 변경 후
torch.load(checkpoint_path, map_location=device, weights_only=False)
```

#### 실행

```bash
conda activate chord310
cd ~/Projects/clones/BTC-ISMIR19/
python test.py --audio input.mp3
# 출력: input.lab (코드 타임라인) + input.midi (자동 생성)
```

### 12-5. 최종 파이프라인

```
원곡 mp3 (또는 YouTube URL)
→ yt-dlp로 MP3 다운로드
→ BTC test.py로 코드 추출 (.lab + .midi 자동 생성)
→ fluidsynth + SoundFont → MP3 렌더링
→ Suno API upload_audio → clip_id
→ custom_generate (cover_clip_id + 가사 + 스타일)
```

---

## 13. midi_cover.py — 자동화 스크립트 사용법

A/B 테스트 결과를 바탕으로 제작된 **6단계 MIDI Cover 자동화 파이프라인**.  
경로: `~/Projects/clones/zach-suno-api/scripts/midi_cover.py`

### 파이프라인 단계

```
Step 0: MP3 파일 또는 YouTube URL → mp3 다운로드 (yt-dlp)
Step 1: Demucs → 보컬 제거 (other + bass + drums 믹스)
Step 2: Chordino → 코드 추출 + Key 필터링 → chords.json
Step 3: 코드 JSON → MIDI → FluidSynth → chords.mp3
Step 4: /api/upload_audio → Suno clip_id 획득
Step 5: /api/custom_generate (cover_clip_id + 가사 + 스타일) → 곡 생성
Step 6: (--wait 옵션) 생성 완료 대기 → mp3 다운로드
```

### 기본 사용법

```bash
# MP3 파일 → Cover 생성
python3 scripts/midi_cover.py /path/to/song.mp3 \
  --style "emotional J-pop, bright piano, 84 BPM" \
  --title "Hero Cover"

# YouTube URL → Cover 생성
python3 scripts/midi_cover.py "https://youtube.com/watch?v=xxx" \
  --style "phonk, dark" \
  --title "Cover"
```

### 전체 옵션

```bash
python3 scripts/midi_cover.py song.mp3 \
  --style "emotional J-pop, bright piano, 84 BPM" \
  --title "Hero Cover" \
  --lyrics lyrics.txt          # 가사 파일 (없으면 "[Instrumental]")
  --model chirp-fenix           # Suno 모델 (기본: chirp-fenix)
  --soundfont ~/Music/sf2/FluidR3_GM.sf2  # SoundFont 경로
  --api http://localhost:3001   # Suno API 주소
  --output ./output             # 출력 디렉토리
  --wait                        # 생성 완료 대기 후 mp3 다운로드
  --skip-demucs                 # 이미 보컬 제거된 파일인 경우
```

### 전제 조건

| 도구 | 설치 | 역할 |
|---|---|---|
| `demucs` | `pip install demucs` | 보컬 제거 |
| `chord_extractor` (Chordino) | conda env `chord310` | 코드 추출 |
| `mido` | `pip install mido` | MIDI 생성 |
| `fluidsynth` | `brew install fluid-synth` | MIDI → WAV 렌더링 |
| `ffmpeg` | `brew install ffmpeg` | WAV → MP3 변환 |
| `yt-dlp` | `pip install yt-dlp` | YouTube 다운로드 (URL 입력 시) |

### 출력물

```
output/
├── original.mp3        # 원본 (YouTube 다운로드 시)
├── no_vocal.wav        # Demucs 보컬 제거 결과
├── chords.json         # 추출된 코드 진행 + Key
├── chords.mid          # 생성된 MIDI
├── chords.mp3          # FluidSynth 렌더링 결과 (Suno 업로드용)
├── cover_XXXXXXXX.mp3  # 생성된 Cover 곡 (--wait 시)
└── metadata.json       # 생성 메타데이터 (key, chords, clip_id, song_ids)
```

### A/B 테스트 결론과의 연결

| 테스트 조건 | midi_cover.py 대응 |
|---|---|
| D: Demucs→BP 하모니 → cover (권장) | 기본 동작 (--skip-demucs 없으면) |
| E: 노트 밀도 최소 | Chordino `roll_on=0.5` + Key 필터로 구현 |
| A: Logic MIDI (최고 품질) | --skip-demucs + 직접 MIDI mp3 지정 |

---

## 14. 참고

- Suno API: `~/Projects/clones/zach-suno-api/`
- midi_cover.py: `~/Projects/clones/zach-suno-api/scripts/midi_cover.py`
- Proxelar 캡처: `~/Projects/clones/prox_cli/captured/suno/`
- A/B 테스트 스크립트: `~/Music/suno-channels/phonk/cover_ab_test/run_ab_test.sh`
- MIDI 비교: `~/Music/suno-channels/phonk/midi_comparison/`
- 이전 비교 (phonk): `~/Music/suno-channels/phonk/suno_comparison/`
