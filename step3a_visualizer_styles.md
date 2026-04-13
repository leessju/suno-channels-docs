# STEP 3a: Remotion 비주얼라이저 스타일 가이드

> Remotion LyricsBox 스타일별 구현 상세.
> 핵심 파일: `src/Visualizer/LyricsBox.tsx`
> 최종 업데이트: 2026-04-13

---

## 1. 스타일 목록

| style 값 | 이름 | 주요 특징 |
|----------|------|----------|
| `classic` | 기본 | 스크롤 가사 + 하이라이트 |
| `emotional-v2` | Emotional V2 | 중앙 하단 3줄 그리드 + Noto Serif JP 명조체 |
| `emotional-v3` | Emotional V3 | emotional-v2 기반 + 추가 조정 |
| `phonk-v2` | Phonk V2 | Syncopate 폰트 + Cyan 네온 + 간주 INSTRUMENTAL BREAK |
| `phonk` | Phonk (레거시) | 불꽃 카라오케 |
| `hardcore` | Hardcore | 스트로브/글리치/네온레드 |

---

## 2. 공통 구조 (모든 스타일)

### 2-1. currentIndex 계산

```tsx
const currentIndex = useMemo(() => {
  const t = frame / fps;
  let best = -1;
  for (let i = 0; i < lyrics.length; i++) {
    const effectiveStart = lyrics[i].words?.[0]?.start ?? lyrics[i].start;
    if (t >= effectiveStart) best = i;
  }
  return best;
}, [frame, fps, lyrics]);
```

- `words[0].start`를 우선 사용 (word-level 싱크 정확도 향상)
- words 없으면 `line.start` 폴백

### 2-2. 베이스 강도 (bassIntensity)

```tsx
// 0~64개 FFT 샘플 중 저주파 6개 평균 × 3.5 (0~1 클램프)
bassIntensity = Math.min(1, values.slice(0, 6).reduce(...) / 6 * 3.5);
```

- 비트 반응 애니메이션의 공통 입력값
- 모든 스타일에서 사용 가능

---

## 3. emotional-v2 / phonk-v2 공유 블록

`line 162`: `if (style === "emotional-v2" || style === "emotional-v3" || style === "phonk-v2")`

### 3-1. 3줄 슬롯 그리드

```
┌─────────────────┐
│  prev (0.25 투명) │  ← 이전 줄
│  active (불투명)  │  ← 현재 활성 줄
│  next (0.25 투명) │  ← 다음 줄
└─────────────────┘
```

- **전환 애니메이션**: 활성 줄 변경 시 10프레임(~0.33초) 동안 슬롯 이동
  - prev: 42px → 42px, opacity 0.25 유지
  - active: 42px → 72px 확대, opacity 0.5 → 1.0
  - next: 42px → 42px, opacity 0 → 0.25

### 3-2. renderKaraoke 검증 로직 (공유)

```tsx
// words 데이터 불충분 시 전체 텍스트로 폴백
if (
  !line.words || line.words.length === 0
  || !line.words.some(w => line.text.includes(w.word.trim()))
  || line.words.reduce((sum, w) => sum + w.word.trim().length, 0)
     < line.text.replace(/\s/g, "").length * 0.5
) {
  return <span>{isPhonkV2 ? line.text.toUpperCase() : line.text}</span>;
}
```

- words 커버리지 < 50% 이면 word-level 카라오케 비활성화
- 일어 등 문자 단위 STT 결과에 대한 안전망

---

## 4. phonk-v2 전용 기능

### 4-1. 단어 카라오케 효과

```
비활성: rgba(255,255,255,0.58) + WebkitTextStroke
활성:   흰색→cyan 그라디언트 + 강한 glow (텍스트 섀도우 110px)
완료:   #00FFFF (solid cyan)
```

**활성화 순간 플래시 (4프레임)**:
```tsx
const wordScale = 1.25 - 0.25 * activationT;       // 1.25 → 1.0
const wordBrightness = 2.8 - 1.8 * activationT;    // 2.8 → 1.0
```

**강한 비트 진동**:
```tsx
const shakeX = isWordActive && bassIntensity > 0.65
  ? Math.sin(frame * 19 + wi * 7) * (bassIntensity - 0.65) * 7
  : 0;
```

### 4-2. 간주(INSTRUMENTAL BREAK) 감지

#### 사전 계산 (마운트 시 1회, useMemo)

```tsx
const interludeRanges = useMemo(() => {
  const ranges = [];
  for (let i = 1; i < lyrics.length - 1; i++) {
    // i=0: 전주 제외, i=last: 후주 제외
    const lineEnd = lyrics[i].end ?? lyrics[i].start;
    const gap = lyrics[i + 1].start - lineEnd;
    if (gap >= 8) ranges.push({ start: lineEnd, end: lyrics[i + 1].start });
  }
  return ranges;
}, [lyrics]);

// 매 프레임 조회
const showInterlude = interludeRanges.some(
  r => currentTime >= r.start && currentTime < r.end
);
```

#### 기준
- **8초 이상** 가사 공백 = 간주
- **전주 제외** (i=0): 노래 시작 전 긴 인트로를 간주로 오인 방지
- **후주 제외** (i=last): 곡 끝 아웃트로 제외

#### Variant Zap 실측 간주 구간 (3회)

| 구간 | start | end | 길이 |
|------|------:|----:|-----:|
| 1절→2절 | 52.98s | 102.70s | 49.72s |
| 2절→3절 | 113.00s | 143.44s | 30.44s |
| 3절→아웃트로 | 158.98s | 204.10s | 45.12s |

### 4-3. INSTRUMENTAL BREAK 고전압 효과

```
"INSTRUMENTAL BREAK"
    │
    ├── Voltage Flicker: random()으로 2프레임마다 opacity 0.4~1.0
    ├── White Flash: bass > 0.6 + 확률 12% → #FFFFFF 순간 번쩍
    ├── Neon Bloom: bass 연동 text-shadow 10px → 40px
    └── Micro Shake: bass > 0.5 시 X±12px / Y±5px
```

**폰트**: `SYNCOPATE_FONT_FAMILY` (간주 텍스트 전용)

**제거된 효과**: 스캔라인 — div/SVG 방식 모두 사각 박스로 보여 제거.

### 4-4. 활성줄 슬롯 숨김

```tsx
// 간주 중에는 active 슬롯 숨김 (INSTRUMENTAL BREAK 표시)
// isAfterActiveLine 완전 제거 (22.05초 버그 원인이었음)
```

**이전 버그 (22.05초)**: `isAfterActiveLine`이 active 슬롯을 숨겨서 prev+next 동시 표시.
**수정**: `isAfterActiveLine` 제거, active 슬롯은 `showInterlude` 일 때만 숨김.

---

## 5. emotional-v2 전용 기능

### 5-1. 폰트

- `SERIF_FONT_FAMILY` (Noto Serif JP 명조체)
- 일어 가사에 최적화

### 5-2. 글래스모피즘 박스

- 컨테이너는 투명
- 활성 줄에만 글래스모피즘 박스 적용

### 5-3. 진행바

```tsx
{/* 진행바 — 래퍼 기준 100% (emotional only) */}
```

- phonk-v2에는 진행바 없음

---

## 6. 스타일 추가 방법

1. `src/Visualizer/LyricsBox.tsx` line 162의 `if` 조건에 새 스타일 추가
2. `isPhonkV2`처럼 스타일 플래그 변수 선언
3. `renderKaraoke` 내에서 스타일별 분기 처리
4. `src/Visualizer/Main.tsx`에서 해당 스타일 레이아웃 설정
5. `src/Root.tsx`에 Composition 등록

---

## 7. 주요 파일

| 파일 | 역할 |
|------|------|
| `src/Visualizer/LyricsBox.tsx` | 가사 렌더링 핵심 (모든 스타일) |
| `src/Visualizer/Main.tsx` | 레이아웃 조합 + 스타일 라우팅 |
| `src/Visualizer/Background.tsx` | 부유 파티클 배경 |
| `src/helpers/font.ts` | 폰트 상수 (SYNCOPATE, PHONK, SERIF) |
| `src/helpers/schema.ts` | Zod 스키마 (LyricLine, LyricWord) |

---

## 8. 개발 시 주의사항

### 싱크 문제 발생 시

- 렌더러(LyricsBox) 코드가 아닌 **전처리(extract_lyrics.py)** 를 먼저 확인
- LyricsBox는 JSON을 표시만 함 → 싱크 = 파이프라인 품질

### 새 효과 추가 시

- `overflow: visible` 확인 — 부모 컨테이너가 클리핑하면 효과 잘림
- 스캔라인 효과는 div/SVG 방식 모두 실패 이력 있음 (사각 박스 문제)
- canvas 기반 대안 검토 필요

### 간주 임계값 (8초) 변경 시

- Variant Zap 기준 실측: 최소 간주 30.44초, 최대 49.72초
- 8초보다 낮추면 짧은 가사 공백도 간주로 감지될 수 있음
- 곡마다 다를 수 있으므로 변경 전 다른 곡에서도 검증 필요
