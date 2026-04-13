# STEP 2: 오디오 후처리 가이드

> Suno mp3 → 보컬 분리 + 가사 타임스탬프 + 다국어 자막 생성.
> 전체 흐름: `step0_pipeline_overview.md`, 이전 단계: `step1_suno_generation_guide.md`

---

## 1. 목표

Suno mp3를 영상 렌더링 가능한 상태로 가공한다.

### 산출물
| 파일 | 경로 | 용도 |
|------|------|------|
| 보컬 분리 wav | `01_songs/{번호}_{곡명}_vocals.wav` | Whisper STT 정확도↑, VAD |
| 가사 JSON | `01_songs/{번호}_{곡명}.lyrics.json` | Remotion 카라오케 타이밍 |
| 에너지 분석 | `01_songs/_energy_analysis.json` | 쇼츠 하이라이트 구간 선택 |
| 다국어 자막 | `03_subtitles/{번호}_{곡명}.{lang}.srt` | YouTube 업로드 자막 |

### lyrics.json 구조
```json
{
  "lines": [
    {
      "start": 15.234,
      "end": 19.876,
      "text": "あの日の君の横顔を",
      "translation_ko": "그날의 네 옆모습이",
      "words": [
        {"word": "あの日の", "start": 15.234, "end": 16.100},
        {"word": "君の", "start": 16.100, "end": 16.800},
        {"word": "横顔を", "start": 16.800, "end": 19.876}
      ]
    }
  ]
}
```

---

## 2. 파이프라인 아키텍처

```
mp3
 │
 ├─→ [A] Demucs (MPS GPU)
 │      - 보컬 분리
 │      - vocals.wav 저장
 │
 ├─→ [B] Whisper STT (medium)
 │      입력: mp3 + vocals.wav (둘 다 사용, 보완)
 │      출력: segments + words + timestamps
 │
 ├─→ [C] txt 매칭 파이프라인
 │      - 순서 제약 매칭
 │      - mega-segment 분할
 │      - 카타카나/로마자 정규화
 │      - 잔여 제거 (Counter)
 │      - 근접 중복 제거 (1초 이내)
 │      - 보간 삽입 (back-align, max 4초, RMS+Demucs VAD)
 │      - 최종 보장 (_txt_idx 기반)
 │      - 첫 줄 강제 삽입
 │
 ├─→ [D] 안전장치
 │      - words < text 70% → 균등 재생성
 │      - entry.start/end ↔ words 범위 맞춤
 │      - 겹침 해소 + 작은 갭(<6초) 메우기
 │      - words 클램핑 + 역전 엔트리 제거
 │
 ├─→ [E] 번역 (Vertex AI Gemini)
 │      - 한국어, 영어, 기타 다국어
 │      - 줄 단위 번역 유지
 │
 └─→ [F] SRT 생성
        - 언어별 SRT 파일
        - 일본어 원문 + 번역 병행 옵션
```

---

## 3. Demucs 보컬 분리

### 3-1. 설치 + 패치 (Apple Silicon MPS)

```bash
pip install demucs

# ⚠️ torchcodec 우회 패치 필수
# /opt/miniconda3/lib/python3.13/site-packages/demucs/audio.py
# /opt/miniconda3/lib/python3.13/site-packages/demucs/wav.py
# ta.save() → soundfile.write() 교체
```

### 3-2. 실행

```bash
# 단일 곡
python3 -m demucs --two-stems=vocals -d mps \
    -o ~/Music/suno-channels/jpop/260402_hero-vol1/01_songs/_demucs \
    ~/Music/suno-channels/jpop/260402_hero-vol1/01_songs/01_ヒーローになれなくても.mp3

# 병렬 (MPS는 2개까지 안전)
# CPU는 8병렬 가능하지만 속도 느림
```

### 3-3. 성능

| 방식 | 속도 | 비고 |
|------|------|------|
| MPS (Apple GPU) | **~20초/곡** | 권장 |
| CPU | ~60초/곡 | 3배 느림 |

20곡 플레이리스트: MPS ~7분, CPU ~20분.

### 3-4. 결과 파일 위치

Demucs 기본 출력 구조:
```
01_songs/_demucs/htdemucs/01_ヒーローになれなくても/
├── vocals.wav    ← 이것만 사용
└── no_vocals.wav
```

**후처리**: vocals.wav만 `01_songs/01_ヒーローになれなくても_vocals.wav`로 복사/이동.

```python
import shutil
from pathlib import Path

def move_vocals(playlist_folder: str):
    songs_dir = Path(playlist_folder) / "01_songs"
    demucs_out = songs_dir / "_demucs" / "htdemucs"
    for song_dir in demucs_out.iterdir():
        if song_dir.is_dir():
            vocals = song_dir / "vocals.wav"
            if vocals.exists():
                target = songs_dir / f"{song_dir.name}_vocals.wav"
                shutil.copy(vocals, target)
    # 정리
    shutil.rmtree(songs_dir / "_demucs")
```

---

## 3-5. EQ / 마스터링 후처리

Suno AI 생성 곡은 **보수적으로 마스터링**되어 있어 후처리가 필요하다.
2가지 검증된 세팅: 장르에 따라 선택.

### A. 보컬 EQ (J-pop, 발라드 등 보컬 중심 장르)

Demucs로 보컬 분리 → 보컬에만 EQ 적용 → 반주와 재합성.

```bash
# 1. 보컬 EQ 적용
ffmpeg -y -i vocals.wav -af "\
highpass=f=100:p=2,\
equalizer=f=250:t=q:w=0.5:g=-4,\
equalizer=f=800:t=q:w=2.0:g=-4,\
equalizer=f=3000:t=q:w=1.0:g=3,\
equalizer=f=5500:t=q:w=1.0:g=5,\
equalizer=f=8000:t=q:w=1.0:g=-2,\
lowpass=f=16000:p=2,\
acompressor=threshold=-18dB:ratio=3:attack=10:release=100:makeup=2,\
afftdn=nr=5:nt=w:om=o" vocals_eq.wav

# 2. 반주와 합침 (amix는 볼륨을 나누므로 volume=4로 보정)
ffmpeg -y \
  -i vocals_eq.wav -i drums.wav -i bass.wav -i other.wav \
  -filter_complex "[0:a][1:a][2:a][3:a]amix=inputs=4:normalize=0,volume=4,loudnorm=I=-13:TP=-1:LRA=11" \
  -b:a 320k output.mp3
```

| 밴드 | 주파수 | 게인 | 역할 |
|------|--------|------|------|
| HPF | 100Hz | — | 초저음 컷 |
| 머드 제거 | 250Hz | -4dB (wide Q) | 울림/탁함 제거 |
| 비음 제거 | 800Hz | -4dB (narrow Q) | AI 보컬 박스감 제거 |
| Presence | 3kHz | +3dB | 보컬 선명도 |
| Air/공감대 | 5.5kHz | +5dB | 존재감, 감동 |
| De-esser | 8kHz | -2dB | ㅅ/ㅊ 치찰음 감소 |
| LPF | 16kHz | — | 디지털 아티팩트 제거 |
| 컴프레서 | — | 3:1, makeup 2 | 다이나믹 균일화 |
| 노이즈 감소 | — | afftdn nr=5 | 미세 노이즈 제거 (매끄러움) |

### B. 전체 마스터링 EQ (EDM, Rock 등 반주 중심 장르)

Demucs 분리 없이 전체 믹스에 직접 적용. "갇힌 느낌" 해소 + 파워풀하게.

```bash
ffmpeg -y -i input.mp3 -af "\
highpass=f=25:p=2,\
equalizer=f=60:t=q:w=0.8:g=2,\
equalizer=f=300:t=q:w=0.7:g=-3,\
equalizer=f=3500:t=q:w=1.0:g=3,\
equalizer=f=5500:t=q:w=1.0:g=3,\
equalizer=f=10000:t=h:g=3,\
acompressor=threshold=-14dB:ratio=3:attack=5:release=100:makeup=3,\
loudnorm=I=-10:TP=-0.5:LRA=8" \
  -b:a 320k output.mp3
```

| 밴드 | 주파수 | 게인 | 역할 |
|------|--------|------|------|
| HPF | 25Hz | — | 서브소닉 컷 (EDM은 낮게) |
| 서브베이스 | 60Hz | +2dB | EDM 무게감/킥 바디 |
| 머드 제거 | 300Hz | -3dB | 갇힌 느낌 해소 |
| Presence | 3.5kHz | +3dB | 앞으로 나오는 느낌 |
| Air | 5.5kHz | +3dB | 공기감 |
| Brilliance | 10kHz+ | +3dB shelf | 열린 느낌 |
| 컴프레서 | — | 3:1, threshold -14dB | 꽉 찬 느낌 |
| loudnorm | — | -10 LUFS, LRA 8 | 파워풀 (유튜브 기준보다 강) |

### C. 장르별 세팅 선택 가이드

| 장르 | 세팅 | 이유 |
|------|------|------|
| J-pop, 발라드, acoustic | **A (보컬 EQ)** | 보컬 선명도가 핵심 |
| EDM, Rock, Metal | **B (전체 마스터링)** | 전체 밸런스 + 파워 |
| Cafe/Lo-fi (instrumental) | **B (약하게)** | 보컬 없으므로 전체 처리 |

### C-1. Pedalboard (Spotify) — 최종 마스터링 엔진

ffmpeg보다 **Pedalboard(Spotify)**가 더 좋은 결과를 냄. Limiter가 loudnorm보다 파워풀.

**설치:** `pip install pedalboard`

**EDM/Rock 마스터링 (최종 확정):**
```python
from pedalboard import (Pedalboard, Compressor, Gain, Limiter,
                         HighpassFilter, PeakFilter)
from pedalboard.io import AudioFile

board = Pedalboard([
    HighpassFilter(cutoff_frequency_hz=25),
    PeakFilter(cutoff_frequency_hz=60, gain_db=2, q=0.8),     # 서브베이스
    PeakFilter(cutoff_frequency_hz=300, gain_db=-3, q=0.7),   # 머드 제거
    PeakFilter(cutoff_frequency_hz=3500, gain_db=3, q=1.0),   # 존재감
    PeakFilter(cutoff_frequency_hz=5500, gain_db=3, q=1.0),   # 공기감
    PeakFilter(cutoff_frequency_hz=10000, gain_db=3, q=0.7),  # 브릴리언스
    Compressor(threshold_db=-14, ratio=3, attack_ms=5, release_ms=100),
    Gain(gain_db=3),
    Limiter(threshold_db=-0.5),
])

with AudioFile("input.mp3") as f:
    audio = f.read(f.frames)
    sr = f.samplerate

result = board(audio, sr)

with AudioFile("output.mp3", "w", sr, result.shape[0], quality=320) as f:
    f.write(result)
```

**J-pop 보컬 EQ (Pedalboard 버전):**
```python
vocal_board = Pedalboard([
    HighpassFilter(cutoff_frequency_hz=100),
    PeakFilter(cutoff_frequency_hz=250, gain_db=-4, q=0.5),   # 머드
    PeakFilter(cutoff_frequency_hz=800, gain_db=-4, q=2.0),   # 비음
    PeakFilter(cutoff_frequency_hz=3000, gain_db=3, q=1.0),   # presence
    PeakFilter(cutoff_frequency_hz=5500, gain_db=5, q=1.0),   # 공감대
    PeakFilter(cutoff_frequency_hz=8000, gain_db=-2, q=1.0),  # de-esser
    LowpassFilter(cutoff_frequency_hz=16000),
    Compressor(threshold_db=-18, ratio=3, attack_ms=10, release_ms=100),
    Gain(gain_db=2),
    Limiter(threshold_db=-0.5),
])
```

**ffmpeg vs Pedalboard 비교 (2차 테스트, 2026-04-12):**

| 항목 | ffmpeg | Pedalboard |
|------|--------|-----------|
| Limiter | loudnorm (LUFS 기준) | Limiter (피크만 깎음) |
| Saturation | 불가능 | `Distortion(drive_db=2)` |
| 코드 통합 | subprocess 호출 | Python 네이티브 |
| 속도 | 빠름 | 빠름 (C++ 백엔드) |
| **보컬 깨짐** | **안 깨짐** | **일부 곡에서 찢어짐 발생** |
| **볼륨 제어** | loudnorm이 LUFS 기준이라 volumedetect와 불일치 | Limiter가 평균 음량을 끌어올려 Gain 조정이 안 먹힘 |
| **결론** | **파이프라인 기본 엔진으로 변경** | ~~채택~~ → 보컬 깨짐 이슈로 폐기 |

**Pedalboard 폐기 사유:**
- 보컬 EQ 부스트(5.5kHz +5dB) + Limiter 조합에서 일부 곡 보컬 찢어짐 (02_海辺の夢 등)
- Limiter가 평균 음량을 끌어올려 Gain(-2~-3.5dB) 조정해도 효과 없음 (-12.4 → -12.0 수준)
- 이중 마스터링 실수 위험 (origin이 아닌 mastered 파일에 재적용 시 즉시 깨짐)

### C-1-1. 최종 파이프라인: F2 (HPF+LPF) + volume 정규화

**확정 세팅 (2026-04-13, 5vol 100곡 적용 완료):**

#### EQ 마스터링 연구 히스토리 (2026-04-12~13)

**Phase 1: Pedalboard + Demucs 4-stem 분리 방식**
Suno 곡을 Demucs로 보컬/반주 분리 → 보컬에 EQ → 재합성하는 방식.

| 세팅 | 250Hz | 800Hz | 3kHz | 5.5kHz | Comp | afftdn | 믹스 비율 | 결과 |
|------|-------|-------|------|--------|------|--------|----------|------|
| A (Pedalboard) | -4 | -4 | +3 | +5 | 3:1,m+2 | nr=5 | v1.0/b1.0 | **보컬 찢어짐** (일부 곡) |
| A (Pedalboard) | -4 | -4 | +3 | +5 | 3:1,m+2 | nr=5 | v1.0/b1.2 | 반주 과보정 |
| B (ffmpeg) | -4 | -4 | +2 | +3 | 3:1,m+2 | nr=5 | v1.0/b1.0 | 보컬 강함 |
| C (ffmpeg) | -4 | -4 | +3 | +3 | 3:1,m+1 | nr=5 | v1.0/b1.0 | 보컬 존재감 약함 |
| D (ffmpeg) | -4 | -4 | +1 | +2 | 3:1,m+2 | nr=5 | v1.0/b1.1 | 배경음과 자연스럽게 섞임 |

- **Pedalboard 폐기 사유**: Limiter가 평균 음량을 끌어올려 Gain 조정 안 먹힘, 일부 곡 보컬 찢어짐
- **Demucs 분리/재합성 문제**: 위상 왜곡, 스펙트럼 블리딩 → 원본 대비 자연스러움 저하
- **amix 볼륨 문제**: `volume=4`는 loudnorm 없이 과도한 증폭(-3.8dB), loudnorm은 LUFS 기준이라 volumedetect와 불일치
- **믹스 비율 테스트**: 1:1(보컬 과다) → 1:1.2(과보정) → 1:1.1(OK) → 최종적으로 직접 EQ로 전환

**Phase 2: ffmpeg + Demucs 분리 + EQ 경량화**
Demucs 분리는 유지하되 EQ를 점진적으로 줄여봄.

| 세팅 | 변경점 | 결과 |
|------|--------|------|
| E2 | 250/800 -4→-2, afftdn 제거 | 더 자연스러움 |
| E3 | + Compressor 3:1→2:1, makeup 2→1 | 원본에 가까움 |

- **afftdn 제거 이유**: Suno 곡은 이미 깨끗. 노이즈 감소가 보컬 디테일(숨소리, 공기감)까지 깎아 인위적 느낌
- **Compressor 완화**: 3:1 + makeup+2는 다이나믹을 과도하게 눌림 → 2:1 + makeup+1

**Phase 3: Demucs 분리 제거 → 직접 EQ**
E3 세팅을 원본에 직접 적용 (Demucs 분리 없음).

| 방식 | 결과 |
|------|------|
| E3 + Demucs 분리 재합성 | Demucs 아티팩트로 부자연스러움 |
| **E3 직접 EQ (Demucs 없음)** | 훨씬 자연스러움 |
| Pedalboard 직접 EQ | ffmpeg와 비슷, Limiter 있어서 EDM에 적합 |

- **핵심 발견**: Demucs 분리 자체가 최대 품질 손실 원인. 직접 EQ가 압도적으로 자연스러움.

**Phase 4: 직접 EQ도 과하다 → 최소 처리**
E3 직접 EQ(3k+1, 5.5k+2)도 청취 1분 후 피로감 발생.

| 세팅 | 내용 | 결과 |
|------|------|------|
| E3 직접 EQ | 250-2, 800-2, 3k+1, 5.5k+2, 8k-2, comp 2:1 | **청취 피로감** — 부스트가 아티팩트 증폭 |
| 컷만 (부스트 없음) | 250-2, 800-2, 8k-2, 컴프 없음 | 안전하지만 **감동 없음** |
| noisereduce (spectral gating) | 노이즈 50% 감소 | **소리가 좁아짐** — 사용 불가 |
| F1 (볼륨만) | EQ 없음, -15.5dB 정규화만 | 원본 그대로 |
| **F2 (HPF+LPF만)** | HPF 80Hz + LPF 16kHz + -15.5dB | **최종 채택** |
| F3 (울트라 라이트) | 컷 -1dB만, 부스트/컴프 없음 | F2와 차이 미미 |

- **부스트(3kHz, 5.5kHz)가 청취 피로의 원인**: 시원하고 뚫리는 느낌을 만들지만, Suno 원곡의 미세한 디지털 아티팩트까지 증폭
- **F2가 최종 채택된 이유**: 들리는 영역(1~16kHz)은 안 건드리고, 안 들리는 양 끝(초저음/초고음)만 정리. 원본 톤 거의 100% 보존

#### 최종 확정 파이프라인

**J-pop / 발라드 (F2 방식):**
```bash
# 1. HPF + LPF만 (중간 주파수 안 건드림)
ffmpeg -y -i origin.mp3 -af "\
highpass=f=80:p=2,\
lowpass=f=16000:p=2" \
  -b:a 320k temp.mp3

# 2. -15.5dB 정규화 (volumedetect 기준)
CUR=$(ffmpeg -i temp.mp3 -af "volumedetect" -f null /dev/null 2>&1 | grep "mean_volume" | sed 's/.*mean_volume: //' | sed 's/ dB//')
DIFF=$(python3 -c "print(-15.5 - ($CUR))")
ffmpeg -y -i temp.mp3 -af "volume=${DIFF}dB" -b:a 320k output.mp3
```

**Phonk / EDM (Pedalboard 직접 EQ):**
```python
from pedalboard import (Pedalboard, Compressor, Gain, Limiter,
                         HighpassFilter, PeakFilter)
from pedalboard.io import AudioFile

board = Pedalboard([
    HighpassFilter(cutoff_frequency_hz=25),
    PeakFilter(cutoff_frequency_hz=60, gain_db=2, q=0.8),     # 서브베이스
    PeakFilter(cutoff_frequency_hz=300, gain_db=-3, q=0.7),   # 머드 제거
    PeakFilter(cutoff_frequency_hz=3500, gain_db=3, q=1.0),   # 존재감
    PeakFilter(cutoff_frequency_hz=5500, gain_db=3, q=1.0),   # 공기감
    PeakFilter(cutoff_frequency_hz=10000, gain_db=3, q=0.7),  # 브릴리언스
    Compressor(threshold_db=-14, ratio=3, attack_ms=5, release_ms=100),
    Gain(gain_db=3),
    Limiter(threshold_db=-0.5),   # Limiter로 음압↑ — EDM에 필수
])
```

**장르별 엔진/타겟:**

| 장르 | 엔진 | EQ 방식 | 타겟 mean_volume |
|------|------|---------|-----------------|
| **J-pop / 발라드** | **ffmpeg** | **F2 (HPF+LPF만)** | **-15.5 dB** |
| **Phonk / EDM** | **Pedalboard** | 직접 EQ (부스트 OK) | -12.0 dB |
| **Cafe / Lo-fi** | **ffmpeg** | F2 (HPF+LPF만) | -15.0 dB |

**핵심 결정:**

| 항목 | 값 | 이유 |
|------|-----|------|
| Demucs 분리 | **마스터링에 사용 안 함** | 분리/재합성 아티팩트가 최대 품질 손실 원인 |
| EQ 부스트 | **J-pop에 사용 안 함** | 부스트가 Suno 아티팩트를 증폭 → 청취 피로 |
| EQ 방식 | **HPF 80Hz + LPF 16kHz만** | 안 들리는 양 끝만 정리, 원본 톤 100% 보존 |
| 정규화 타겟 | **-15.5 dB** | 원본(-15~-16.5)과 비슷한 수준, 청취 피로 없음 |
| 정규화 방식 | **ffmpeg volume 필터** | 단순 gain, 정확히 타겟 도달 |

**주의사항:**
- 반드시 **_origin.mp3**(원본)에서 처리할 것. 이중 처리 금지
- Demucs 분리는 **Whisper STT용 보컬 추출에만 사용** (마스터링에는 사용 안 함)
- afftdn(노이즈 감소)은 Suno 곡에 사용 금지 — 인위적 느낌 유발
- noisereduce(spectral gating)도 사용 금지 — 소리가 좁아짐
- J-pop에 EQ 부스트 금지 — 아티팩트 증폭 + 청취 피로

### C-2. Matchering — 레퍼런스 매칭 (선택 옵션)

프로 곡을 레퍼런스로 지정하면 RMS/EQ/스테레오를 자동 매칭.

**설치:** `pip install matchering`

```python
import matchering as mg
mg.process(
    target="suno_output.mp3",
    reference="pro_reference.mp3",
    results=[mg.pcm16("mastered.wav")]
)
```

**주의:** 레퍼런스 곡과 다른 장르면 어울리지 않음. 같은 장르의 프로 곡 필요.

### C-3. 스테레오 와이드닝 — 사용 금지

`extrastereo` 테스트 결과: **Suno 곡에 적용하면 즉시 찢어짐**. 
Suno가 이미 스테레오로 생성하므로 추가 와이드닝 불필요.
EQ의 고음 부스트(3.5k, 5.5k, 10k)만으로 체감 공간감 충분.

### D. 주의사항

| 항목 | 상세 |
|------|------|
| **amix 볼륨** | `amix`는 입력 수만큼 볼륨을 나눔. `normalize=0,volume=N`으로 보정 필수 |
| **loudnorm 타겟** | 장르마다 다름. J-pop: -13 LUFS, EDM: -10 LUFS |
| **afftdn** | 보컬에만 사용. 전체 믹스에 쓰면 반주가 손상됨 |
| **Demucs 잔향** | 분리→합성 시 약간의 잔향 발생 가능. 허용 범위 내 |
| **볼륨 검증** | 처리 후 반드시 `volumedetect`로 원본과 비교 |

### E. 볼륨 검증 명령어

```bash
# 원본과 처리 결과 볼륨 비교
ffmpeg -i original.mp3 -af "volumedetect" -f null /dev/null 2>&1 | grep "mean_volume"
ffmpeg -i processed.mp3 -af "volumedetect" -f null /dev/null 2>&1 | grep "mean_volume"
# mean_volume 차이가 ±1dB 이내면 OK
```

---

## 4. Whisper STT

### 4-1. 설치

```bash
pip install faster-whisper

# 모델 다운로드 (첫 실행 시 자동)
# medium: 일본어/한국어/영어 균형, ~1.5GB
# large-v3: 더 정확하지만 느림 + 일본어 이상 발생 가능
```

### 4-2. 실행 전략 (mp3 + vocals 이중 처리)

**핵심**: 원본 mp3 + vocals.wav 둘 다 Whisper 돌려서 결과 보완.

```python
from faster_whisper import WhisperModel

def run_whisper(audio_path: str, language: str = "ja"):
    model = WhisperModel("medium", device="auto", compute_type="auto")
    segments, info = model.transcribe(
        audio_path,
        language=language,
        word_timestamps=True,    # 단어 단위 타임스탬프 (카라오케용)
        vad_filter=True,         # Voice Activity Detection
        beam_size=5,
    )

    result = {
        "segments": [],
        "words": [],
    }
    for seg in segments:
        seg_data = {
            "start": seg.start,
            "end": seg.end,
            "text": seg.text.strip(),
            "words": [
                {"word": w.word, "start": w.start, "end": w.end}
                for w in seg.words or []
            ]
        }
        result["segments"].append(seg_data)
        result["words"].extend(seg_data["words"])

    return result


def whisper_with_vocals_fallback(mp3: str, vocals: str):
    """mp3 + vocals 결합 처리"""
    # 1. 원본 mp3 전체 처리
    mp3_result = run_whisper(mp3)

    # 2. 시작 갭 보완 (vocals로 재처리)
    vocals_result = run_whisper(vocals)

    # 3. VAD 기반 병합
    # - 원본에서 놓친 첫 줄 → vocals 결과로 보완
    # - 극소 words (<0.1초) → vocals 결과로 교체
    merged = merge_whisper_results(mp3_result, vocals_result)

    return merged
```

### 4-3. Whisper 한계 (코드로 해결 불가)

| 한계 | 설명 | 대응 |
|------|------|------|
| 줄 누락 | 특정 곡 일부 줄 못 잡음 | txt 매칭으로 보간 |
| 간주 배치 | 정확한 시작점 모름 | RMS + Demucs VAD 보정 |
| 극소 words | <0.1초짜리 단어 | vocals Whisper로 교체 |
| Mega-segment | 여러 줄이 한 덩어리로 합쳐짐 | txt 순서로 분할 |
| 비결정성 | 매 실행마다 결과 변동 | **검증 통과 캐시 절대 삭제 금지** |

---

## 5. txt 매칭 파이프라인 (핵심)

### 5-1. 입력
- Whisper 결과 (segments + words)
- Suno 가사 txt (`01_songs/01_ヒーローになれなくても.txt`)

### 5-2. 매칭 알고리즘

```python
def match_lyrics_to_whisper(whisper_segments, txt_lines):
    """순서 제약 + mega-segment 분할 + 카타카나 정규화"""

    # 1. 카타카나 → 로마자 정규화
    def normalize(s):
        s = katakana_to_romaji(s)
        s = s.lower()
        s = re.sub(r'\s+', '', s)
        return s

    # 2. 순서 제약 매칭
    matches = []
    last_matched_idx = -1
    for txt_line in txt_lines:
        normalized = normalize(txt_line)
        best_idx = -1
        best_score = 0
        # last_matched_idx 이후만 검색
        for i in range(last_matched_idx + 1, len(whisper_segments)):
            seg = whisper_segments[i]
            seg_norm = normalize(seg["text"])
            score = fuzz_ratio(normalized, seg_norm)
            if score > best_score:
                best_score = score
                best_idx = i
        if best_idx >= 0 and best_score > 60:
            matches.append((txt_line, whisper_segments[best_idx]))
            last_matched_idx = best_idx

    # 3. mega-segment 분할
    # 여러 txt 줄이 한 whisper 세그먼트에 매칭됐으면 → 분할

    # 4. Counter 잔여 제거
    # 같은 세그먼트가 여러 txt 줄에 매칭됐으면 → 위치 최적 선택

    # 5. 근접 중복 제거 (1초 이내)

    return matches
```

### 5-3. 보간 삽입

매칭 실패한 txt 줄은 **앞뒤 매칭된 줄 사이에 균등 배치**하되, RMS + Demucs VAD로 실제 보컬 시작점 찾기:

```python
def interpolate_missing(matches, whisper_result, audio_path, vocals_path):
    """매칭 실패 줄을 앞뒤 사이에 보간"""
    result = []
    for i, (txt, seg) in enumerate(matches):
        if seg is None:
            # 앞뒤 매칭된 entry 사이 균등 배치
            prev = find_prev_matched(matches, i)
            next_match = find_next_matched(matches, i)
            estimated_start = (prev.end + next_match.start) / 2

            # RMS로 보컬 시작 정밀화
            precise_start = find_vocal_onset(vocals_path, estimated_start)

            # max 4초 back-align
            if precise_start < estimated_start - 4:
                precise_start = estimated_start - 4

            result.append({"text": txt, "start": precise_start, ...})
    return result
```

### 5-4. 첫 줄 강제 삽입

Whisper가 첫 줄을 자주 놓친다 → 반드시 강제 삽입:

```python
def force_first_line(matches, txt_lines, vocals_path):
    """첫 줄이 매칭 안 됐으면 강제로 추가"""
    if matches[0].text != txt_lines[0]:
        # vocals Whisper에서 첫 음성 위치 찾기
        first_vocal_time = find_first_vocal(vocals_path)
        matches.insert(0, {
            "text": txt_lines[0],
            "start": first_vocal_time,
            "end": first_vocal_time + 3.0,
        })
    return matches
```

---

## 6. 검증 (8항목)

**모든 곡은 렌더 전 반드시 검증**. 하나라도 실패하면 수동 조정.

| # | 항목 | 체크 |
|:-:|------|------|
| 1 | **txt 순서** | 매칭된 순서가 원본 txt와 일치 |
| 2 | **중복** | 같은 줄이 두 번 이상 안 나오는지 |
| 3 | **겹침** | entry[i].end > entry[i+1].start 없는지 |
| 4 | **words 보유** | 각 entry에 words 배열 존재 |
| 5 | **words 범위** | words[0].start >= entry.start, words[-1].end <= entry.end |
| 6 | **극소 words** | <0.1초 단어 없는지 |
| 7 | **갭** | 6초 이상 비정상 갭 없는지 |
| 8 | **첫 줄** | 첫 줄이 txt[0]과 일치 |

### 검증 스크립트
```bash
# 전체 곡 검증
python ~/Projects/clones/suno-video/scripts/verify_all.py \
    ~/Music/suno-channels/jpop/260402_hero-vol1
```

### 실패 대응
- **재렌더 금지** (Whisper 비결정성으로 더 나빠질 수 있음)
- 검증 통과 곡의 `.lyrics.json`은 **절대 삭제 금지**
- 실패 곡만 수동 조정 or 수용

---

## 7. 번역 (Vertex AI Gemini)

### 7-1. 인증
```python
import os
os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = \
    "/Users/nicejames/Projects/claude/test/niche_bending/infra/credentials_account3.json"

from google.cloud import aiplatform
aiplatform.init(project="project-3f3c3c98-9d26-4383-bcb", location="us-central1")
```

### 7-2. 번역 스크립트

```python
import json
import requests
import subprocess

def translate_lyrics(lyrics_json_path: str, target_langs: list = ["ko", "en"]):
    """lyrics.json의 각 줄을 다국어 번역"""
    with open(lyrics_json_path) as f:
        data = json.load(f)

    lines = [entry["text"] for entry in data["lines"]]
    all_text = "\n".join(lines)

    token = subprocess.run(
        ["gcloud", "auth", "print-access-token"],
        capture_output=True, text=True
    ).stdout.strip()

    url = "https://us-central1-aiplatform.googleapis.com/v1/projects/project-3f3c3c98-9d26-4383-bcb/locations/us-central1/publishers/google/models/gemini-2.5-flash:generateContent"

    for lang in target_langs:
        prompt = f"""
다음 일본어 가사를 {lang}로 번역하세요.
- 줄 순서를 반드시 유지 (입력과 같은 줄 수로 출력)
- 자연스러운 {lang} 표현 사용
- 감정/뉘앙스 보존

가사:
{all_text}

번역 (줄 수는 입력과 동일):
"""
        resp = requests.post(url,
            headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
            json={
                "contents": [{"parts": [{"text": prompt}]}],
                "generationConfig": {"temperature": 0.3}
            }
        )
        translated_lines = resp.json()["candidates"][0]["content"]["parts"][0]["text"].strip().split("\n")

        # lyrics.json에 병합
        if len(translated_lines) == len(data["lines"]):
            for i, entry in enumerate(data["lines"]):
                entry[f"translation_{lang}"] = translated_lines[i]
        else:
            print(f"⚠️ 줄 수 불일치: {len(translated_lines)} vs {len(data['lines'])}")

    # 저장
    with open(lyrics_json_path, "w") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
```

---

## 8. SRT 생성

### 8-1. 단일 언어 SRT

```python
def lyrics_to_srt(lyrics_json_path: str, lang: str, out_path: str):
    with open(lyrics_json_path) as f:
        data = json.load(f)

    with open(out_path, "w") as f:
        for i, entry in enumerate(data["lines"], 1):
            start = format_time(entry["start"])
            end = format_time(entry["end"])

            if lang == "original":
                text = entry["text"]
            else:
                text = entry.get(f"translation_{lang}", entry["text"])

            f.write(f"{i}\n{start} --> {end}\n{text}\n\n")

def format_time(seconds: float) -> str:
    hours = int(seconds // 3600)
    minutes = int((seconds % 3600) // 60)
    secs = seconds % 60
    return f"{hours:02d}:{minutes:02d}:{secs:06.3f}".replace(".", ",")
```

### 8-2. 다국어 일괄 생성

```python
def generate_all_srts(playlist_folder: str):
    srt_dir = Path(playlist_folder) / "03_subtitles"
    srt_dir.mkdir(exist_ok=True)

    langs = ["ko", "en", "ja", "zh", "es", "pt", "fr", "de", "ru", "id"]

    for song_dir in (Path(playlist_folder) / "01_songs").glob("*.lyrics.json"):
        base = song_dir.stem.replace(".lyrics", "")
        for lang in langs:
            out = srt_dir / f"{base}.{lang}.srt"
            lyrics_to_srt(str(song_dir), lang, str(out))
```

### 8-3. 풀영상용 플레이리스트 SRT

개별 곡 SRT를 plays 오프셋을 더해서 하나로 합침 (YouTube 업로드용):

```python
def merge_srts_for_playlist(playlist_folder: str, lang: str):
    """20곡 SRT를 하나로 합쳐서 풀영상에 맞춘 timestamps 생성"""
    songs = sorted((Path(playlist_folder) / "01_songs").glob("*.mp3"))
    offset = 0.0
    merged = []
    idx = 1

    for song in songs:
        duration = get_audio_duration(song)
        srt = Path(playlist_folder) / "03_subtitles" / f"{song.stem}.{lang}.srt"
        if not srt.exists():
            offset += duration + 1.5  # silence gap
            continue

        with open(srt) as f:
            entries = parse_srt(f.read())

        for e in entries:
            merged.append({
                "idx": idx,
                "start": e["start"] + offset,
                "end": e["end"] + offset,
                "text": e["text"]
            })
            idx += 1

        offset += duration + 1.5  # 1.5초 silence

    # 저장
    out = Path(playlist_folder) / "03_subtitles" / f"_playlist.{lang}.srt"
    with open(out, "w") as f:
        for e in merged:
            f.write(f"{e['idx']}\n{format_time(e['start'])} --> {format_time(e['end'])}\n{e['text']}\n\n")
```

---

## 9. 에너지 분석 (쇼츠 하이라이트용)

```python
import librosa
import numpy as np
import json

def analyze_energy(playlist_folder: str):
    """각 곡의 에너지 피크 구간 찾기 (쇼츠용 40초)"""
    results = {}
    for mp3 in Path(playlist_folder, "01_songs").glob("*.mp3"):
        y, sr = librosa.load(mp3, sr=22050)

        # RMS 에너지
        hop = 512
        rms = librosa.feature.rms(y=y, frame_length=2048, hop_length=hop)[0]
        times = librosa.frames_to_time(np.arange(len(rms)), sr=sr, hop_length=hop)

        # 40초 슬라이딩 윈도우의 평균 에너지 최대 구간
        window = 40  # 초
        window_frames = int(window * sr / hop)
        if len(rms) < window_frames:
            best_start = 0
        else:
            cumsum = np.cumsum(rms)
            avg = (cumsum[window_frames:] - cumsum[:-window_frames]) / window_frames
            best_start_frame = int(np.argmax(avg))
            best_start = times[best_start_frame]

        results[mp3.stem] = {
            "highlight_start": float(best_start),
            "highlight_end": float(best_start + window),
            "duration": float(len(y) / sr),
        }

    out = Path(playlist_folder) / "01_songs" / "_energy_analysis.json"
    with open(out, "w") as f:
        json.dump(results, f, indent=2)
    return results
```

---

## 10. 자동화 함수 시그니처

```python
def process_audio_playlist(
    playlist_folder: str,
    demucs_device: str = "mps",
    whisper_model: str = "medium",
    translate_langs: list[str] = ["ko", "en"],
    skip_cached: bool = True,      # 검증 통과 캐시 재사용
) -> dict:
    """
    1. Demucs 보컬 분리 (MPS)
    2. Whisper STT (mp3 + vocals)
    3. txt 매칭 파이프라인
    4. 8항목 검증
    5. Gemini 번역
    6. SRT 생성
    7. 에너지 분석

    Returns:
        {
            "processed": 20,
            "validated": 18,
            "failed": ["track_12", "track_16"],
            "srt_files": [...],
        }
    """
    ...


def verify_lyrics_json(lyrics_json_path: str) -> dict:
    """8항목 검증"""
    ...


def translate_lyrics(
    lyrics_json_path: str,
    target_langs: list[str],
) -> None:
    """번역을 lyrics.json에 inline 추가"""
    ...
```

---

## 11. 트러블슈팅

### 11-1. Demucs torchcodec 에러
**증상**: `ModuleNotFoundError: torchcodec`
**해결**: demucs/audio.py, wav.py 패치 (ta.save → soundfile.write)

### 11-2. MPS OOM
**증상**: `mps backend out of memory`
**해결**: 병렬을 1로 낮추거나, 곡 길이 30분 초과 시 청크 분할

### 11-3. Whisper 일본어 이상 문자
**증상**: 카타카나 대신 한자로 인식
**해결**: `language="ja"` 강제, medium 모델 사용 (large는 이상 발생 가능)

### 11-4. 가사 미스매치 (MP3와 txt 불일치)
**증상**: MP3에 있는 가사가 txt에 없음 (Suno가 다른 버전 생성)
**해결**: Suno API `get?ids={id}` → lyric 필드로 txt 재생성

### 11-5. SAFE_STEM 파일명 충돌
**증상**: 일본어 파일명이 MD5로 변환되면서 다른 곡과 충돌
**해결**: render.sh SAFE_STEM 로직에 숫자만 남을 경우 MD5 해시 8자 추가 (이미 적용됨)

### 11-6. 번역 줄 수 불일치
**증상**: Gemini가 번역 시 줄 수가 원본과 다름
**해결**: 프롬프트에 "줄 수 반드시 유지" 강조, 실패 시 줄 단위로 개별 번역

---

## 12. 체크리스트

### 처리 전
- [ ] Suno에서 받은 mp3 + jpeg + txt 3종 세트 확인
- [ ] Demucs 패치 적용 확인 (torchcodec 우회)
- [ ] GCP account3 크레딧 확인

### 처리 중
- [ ] Demucs MPS 병렬 2 이하
- [ ] Whisper는 mp3 + vocals 둘 다 처리
- [ ] txt 매칭 4단계 전부 실행 (순서/mega/중복/보간)
- [ ] 검증 통과 캐시 보존

### 처리 후
- [ ] 8항목 검증 통과 확인
- [ ] 번역 줄 수 일치 확인
- [ ] SRT 생성 (10개 언어)
- [ ] 에너지 분석 JSON 생성
- [ ] 다음 단계 (STEP 3: 영상 제작)로 이동

---

## 13. 다음 단계

`step3_video_production_guide.md`로 이동:
- Remotion DynamicVisualizer 렌더
- 플레이리스트 머지 (1.5초 무음)
- 쇼츠 생성 (가로→세로 변환)
