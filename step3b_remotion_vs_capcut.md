# STEP 3b: Remotion 활용 범위 — CapCut 비교

> Remotion으로 할 수 있는 것과 없는 것. CapCut과의 역할 비교.
> 최종 업데이트: 2026-04-15

---

## 1. 한 줄 요약

**CapCut** = 손으로 한 번 만드는 도구  
**Remotion** = 코드로 찍어내는 공장

---

## 2. Remotion이 CapCut을 압도하는 것

| 기능 | 설명 |
|------|------|
| **배치 자동화** | 곡 100개 → 영상 100개 무인 렌더링 |
| **데이터 연동** | JSON/DB 값으로 텍스트·색상·타이밍 자동 변경 |
| **카라오케 싱크** | 단어 단위 타이밍 — CapCut 수동으로 불가능한 수준 |
| **음악 반응 비주얼** | FFT/베이스 연동 파티클, EQ, 파형 실시간 계산 |
| **파이프라인 통합** | 렌더 → 자막 생성 → YouTube 업로드 전자동화 |
| **버전 관리** | git으로 영상 템플릿 코드 관리 가능 |

---

## 3. CapCut 기능 중 Remotion에서도 되는 것

| CapCut 기능 | Remotion 구현 방법 |
|------------|------------------|
| 자막 추가 | `<Sequence>` + 텍스트 컴포넌트 |
| 페이드인/아웃 | `interpolate()` + opacity |
| 트랜지션 효과 | `interpolate()` + CSS transform |
| 영상 위에 영상 오버레이 | `<OffthreadVideo>` 레이어 |
| 배경음악 + 영상 합성 | `<Audio>` + `<OffthreadVideo>` |
| 영상 재생 속도 조절 | `<Video playbackRate={n}>` |
| 텍스트 애니메이션 | `spring()` + CSS |
| 인트로/아웃트로 삽입 | `<Sequence from durationInFrames>` |

---

## 4. Remotion이 약한 것

| 기능 | 비고 |
|------|------|
| 타임라인 드래그 편집 | UI 없음 — 코드로만 제어 |
| 색보정 (LUT, 컬러그레이딩) | 제한적 |
| 영상 트리밍/자르기 실시간 | ffmpeg 별도 필요 |
| 손떨림 보정, AI 배경제거 | 미지원 |
| 한 번짜리 빠른 편집 | CapCut이 압도적으로 빠름 |

---

## 5. 실제 활용 전략

### Remotion을 쓰는 경우
- 동일 템플릿으로 **반복 생산**이 필요할 때
- 음악/데이터에 반응하는 **동적 비주얼**이 필요할 때
- **파이프라인 자동화** (렌더→업로드) 를 구축할 때

### CapCut을 쓰는 경우
- **한 번짜리** 특별 편집이 필요할 때
- 트렌디한 **CapCut 템플릿 효과**를 빠르게 적용할 때
- 색보정·AI 기능이 필요할 때

### 혼합 전략 (권장)
```
Remotion → 본편 영상 자동 생성
CapCut   → 인트로/특수 효과 영상 제작
ffmpeg   → 두 영상 합성 (concat)
```

---

## 6. Remotion 핵심 컴포넌트 참고

| 컴포넌트 | 역할 |
|---------|------|
| `<Sequence from durationInFrames>` | 타임라인 특정 구간에만 렌더 |
| `<OffthreadVideo>` | 외부 mp4 영상 레이어 삽입 |
| `<Audio>` | 오디오 재생 + 타이밍 제어 |
| `<AbsoluteFill>` | 전체 화면 레이어 |
| `interpolate()` | 프레임 기반 수치 보간 (페이드, 이동 등) |
| `spring()` | 물리 기반 스프링 애니메이션 |
| `useCurrentFrame()` | 현재 프레임 번호 (모든 애니메이션의 기준) |
| `useVideoConfig()` | fps, durationInFrames, width, height |

---

## 7. mlx-whisper 전환 완료 (2026-04-13)

전처리 파이프라인이 `faster-whisper` → `mlx-whisper`로 전환 완료.

| 항목 | faster-whisper | mlx-whisper |
|------|---------------|-------------|
| 곡당 시간 | ~42초 | ~9초 |
| 20곡 총 시간 | ~14분 | ~3분 |
| 속도 향상 | 기준 | **4.5배 빠름** |

- Apple Silicon Metal GPU 직접 활용
- `extract_lyrics.py` — mlx-whisper 우선, faster-whisper fallback 구조
- 상세: `step2a_lyrics_sync_pipeline.md` 참고
