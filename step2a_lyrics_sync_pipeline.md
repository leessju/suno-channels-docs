# STEP 2a: 가사 싱크 파이프라인 상세 가이드

> step2의 `extract_lyrics.py` 핵심 로직 상세 문서.
> 전처리 단계에서 싱크 품질이 결정됨 — Remotion 렌더러는 이 JSON을 그대로 표시.
> 최종 업데이트: 2026-04-13

---

## 1. 핵심 원칙

| 원칙 | 내용 |
|------|------|
| **txt 가사 = 절대 정답** | 화면에 표시되는 텍스트는 항상 txt 파일 내용 |
| **Whisper = 타이밍만** | Whisper STT는 타임스탬프 추출에만 사용, 텍스트 판단 권한 없음 |
| **엉뚱한 가사 = 크리티컬 버그** | txt와 다른 텍스트가 표시되면 즉시 수정 필요 |
| **txt의 모든 줄 = 영상에 표시** | 누락 줄 없음 |

---

## 2. 전체 파이프라인 흐름

```
mp3 (원본)
  ↓ Demucs (보컬 분리)
vocals.wav
  ↓ Whisper STT (타임스탬프 추출)
whisper_segments (word-level timing)
  ↓ extract_lyrics.py (핵심 전처리)
    ├── txt 가사와 STT 결과 매칭
    ├── mega-segment 분할
    ├── カタカナ/romaji 정규화
    ├── fuzzy word 매칭
    ├── zero-duration 보간
    └── 크레딧 줄 제거
  ↓
lyrics.json (Remotion 입력)
```

---

## 3. extract_lyrics.py 핵심 로직

### 3-1. mega-segment 분할

**문제**: Whisper가 여러 줄을 하나의 긴 segment로 묶을 때 발생.

```
Whisper output: "あの日の君の横顔を もう一度だけ見たくて" (1 segment, 0~8s)
txt:            あの日の君の横顔を      ← 줄 1
                もう一度だけ見たくて   ← 줄 2
```

**해결**: `words` 배열에서 txt 줄 경계를 찾아 분할.

```python
# mega-segment 감지: txt_norm in seg_norm 비교 (katakana 정규화 포함)
if txt_norm in seg_norm:
    # words 기반으로 해당 줄의 시작/끝 타임스탬프 추출
    split_by_words(segment, txt_line)
```

### 3-2. カタカナ/romaji 정규화

**문제**: Whisper가 "ラララ"로 인식하고 txt에 "la la la"가 있을 때 매칭 실패.

**해결**: `_katakana_to_romaji()` 함수 — カタカナ→ローマ字 변환 후 비교.

```python
def _katakana_to_romaji(text: str) -> str:
    """カタカナ를 romaji로 변환 (r/l 통일)"""
    # ラ→ra, リ→ri ... (r/l 구분 없이 통일)
    ...

# 적용 위치
# 1) mega-segment 감지: txt_norm in seg_norm 비교
# 2) _filter_words_to_text(): 단어 매칭 시 katakana 정규화
```

**결과**: 곡 06 — 16줄/3누락/6순서오류 → 19줄/0누락/1순서(중복줄)

### 3-3. fuzzy word 매칭

**문제**: Whisper 단어와 txt 단어가 완전 일치하지 않을 때 (발음 차이, 받아쓰기 오류).

**해결**: `_filter_words_to_text()`에서 katakana 정규화 + 2문자 이상 fuzzy fallback.

```python
def _filter_words_to_text(words, line_text):
    # 1차: 완전 매칭
    # 2차: katakana 정규화 후 매칭
    # 3차: 2문자 이상 fuzzy fallback
    ...
```

### 3-4. zero-duration 보간

**문제**: Whisper가 단어에 `start == end` (zero-duration)를 할당하는 경우.

**해결**: 정상 단어 보존, zero-duration만 주변 단어 기반으로 보간.

```python
def _fix_zero_duration_words(words):
    # 기존: 전체 균등분배 (정상 단어까지 덮어씀)
    # 수정: zero-duration만 보간, 정상 단어 타이밍 유지
    for w in words:
        if w['start'] == w['end']:
            interpolate_from_neighbors(w)
```

### 3-5. 반복 가사 `_txt_idx` 기반 매칭

**문제**: 동일한 가사가 반복될 때 (후렴구) 위치 오류.

```
txt: 사랑해  ← 2절 (50초)
     ...
     사랑해  ← 3절 (100초)

잘못된 매칭: 3절 "사랑해"에 2절 타임스탬프 부여
```

**해결**: `_txt_idx` (txt 파일 내 절대 줄 번호)로 순서 보장.

```python
# 단순 텍스트 매칭이 아닌 인덱스 기반 매칭
entry['_txt_idx'] = line_number  # txt 파일 줄 번호
# → 같은 텍스트라도 순서대로 매핑
```

### 3-6. 크레딧 줄 자동 제거

**문제**: Suno가 txt에 `作詞·作曲·編曲` 등 크레딧 줄을 포함.

```python
CREDIT_PATTERNS = [
    r'^作詞[：:·]', r'^作曲[：:·]', r'^編曲[：:·]',
    r'^Words\s*[&and]\s*Music', r'^Lyrics\s*by',
    ...
]

def _is_credit_line(text: str) -> bool:
    return any(re.match(p, text) for p in CREDIT_PATTERNS)
```

---

## 4. 스타일별 적용 범위

| 스타일 | 파이프라인 적용 | 비고 |
|--------|:--------------:|------|
| classic | ✅ | 공통 JSON |
| emotional-v2/v3 | ✅ | 공통 JSON |
| phonk-v2 | ✅ | 공통 JSON |
| hardcore | ✅ | 공통 JSON |

**결론**: 파이프라인 수정은 스타일과 무관하게 전체 적용됨. Remotion 렌더러는 JSON을 표시만 하므로 전처리 품질 = 싱크 품질.

---

## 5. 검증 도구

```bash
# 기본 검증 (6항목)
python scripts/verify_lyrics.py {곡_경로}

# 정밀 검증 (8항목)
python scripts/verify_all.py {곡_경로}

# 배치 검증
python scripts/batch_verify.py {플레이리스트_폴더}
```

### 8항목 검증 기준

| 항목 | 기준 |
|------|------|
| 누락 줄 | 0개 |
| 순서 오류 | 0개 |
| zero-duration 단어 | 0개 |
| 타임스탬프 역전 | 0개 |
| 크레딧 줄 포함 | 0개 |
| txt 텍스트 일치 | 100% |
| 단어 커버리지 | ≥ 50% (일어 특성상) |
| 전체 duration 범위 | 오디오 길이 이내 |

---

## 6. 트러블슈팅

### 일어 곡에서 가사 누락

1. Whisper가 カタカナ를 romaji로 인식했는지 확인
2. `_katakana_to_romaji()` 변환 후 재매칭
3. mega-segment 분할 실패 여부 확인 (`words` 배열 길이 vs txt 줄 수)

### 후렴구 타이밍 오류

1. `_txt_idx` 기반 매칭 동작 여부 확인
2. txt 파일의 후렴구 줄 번호와 JSON 매핑 비교

### zero-duration 단어 다수 발생

1. Demucs 보컬 분리 품질 확인 (instruments가 너무 강하면 STT 오류)
2. Whisper 모델 크기 확인 (`large-v3` 권장)
3. `_fix_zero_duration_words()` 보간 결과 확인

---

## 7. mlx-whisper 전환 (완료 — 2026-04-13)

**이전:** `faster-whisper` (CPU int8, 곡당 ~42초)
**현재:** `mlx-whisper` (Apple Silicon MLX, 곡당 ~9초, **4.5배 빠름**)

### 전환 내용
- `extract_lyrics.py`의 `extract_lyrics()` 함수: mlx-whisper 우선, faster-whisper fallback
- 모델: `mlx-community/whisper-medium` (HuggingFace MLX 변환 모델)
- VAD는 여전히 faster-whisper tiny 사용 (mlx-whisper에 `vad_filter` 옵션 없음)
- mlx-whisper 특유 노이즈 필터 추가: `^[-─—\s.…♪*]+$` 패턴 세그먼트 자동 제거

### 성능 비교 (20곡 배치, medium 모델, M1 Pro)
| 항목 | faster-whisper | mlx-whisper |
|------|---------------|-------------|
| 곡당 시간 | ~42초 | ~9초 |
| 20곡 총 시간 | ~840초 (14분) | ~180초 (3분) |
| 매칭 품질 | 기준 | 동등 |
| 노이즈 세그먼트 | 적음 | 많음 (필터 적용) |

### 주의사항
- mlx-whisper는 **비결정적 출력** — 동일 입력에 약간 다른 결과 가능
- temperature 기본 튜플 `(0.0, 0.2, 0.4, 0.6, 0.8, 1.0)`이 0.0 고정보다 품질 우수
- 일부 곡에서 첫 가사 타이밍이 수초 늦게 감지될 수 있음 (수동 조정 필요)

---

## 8. 관련 파일

| 파일 | 역할 |
|------|------|
| `scripts/extract_lyrics.py` | 핵심 전처리 (1322줄, mlx-whisper primary) |
| `scripts/generate_lyrics_txt.py` | txt 없을 때 Gemini STT로 자동 생성 |
| `scripts/verify_lyrics.py` | 기본 검증 |
| `scripts/verify_all.py` | 정밀 검증 |
| `scripts/batch_verify.py` | 배치 검증 |
| `public/lyrics_variant_*.json` | 렌더용 가사 JSON (출력물) |
