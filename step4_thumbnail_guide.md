# 썸네일 제작 가이드

> 모든 채널/플레이리스트에 범용 적용. 향후 자동화 파이프라인의 입력 스펙.

---

## 0. 현재 운영 방식 (자동화 완료)

### 0-1. 배경 이미지 할당 (assign_background_images.py)

각 vol의 곡별 배경 이미지를 `assets/back_image/{genre}/` 풀에서 균등 배분합니다.

```
assets/back_image/
├── jpop/
│   ├── j_001.png ~ j_NNN.png    ← 02번~ 곡에 사용
│   └── cover/
│       └── j_001.png ~ ...      ← 01번 곡 전용 (썸네일 겸용)
└── phonk/
    ├── p_001.png ~ p_NNN.png
    └── cover/
        └── p_001.png ~ ...
```

**규칙:**
- 01번 곡 → `cover/` 풀만 사용
- 02번~ 곡 → `cover/` + 일반 풀 전체 사용
- vol 내 중복 금지 (stem 기준 — `cover/j_001.png` == `j_001.png` 동일 취급)
- 풀 소진 시 least-used 이미지 재사용 허용
- `gems.db back_image_usage` 테이블에 사용 횟수 기록

```bash
python3 scripts/assign_background_images.py              # 전체 vol 처리
python3 scripts/assign_background_images.py --dry-run   # 미리보기
python3 scripts/assign_background_images.py jpop        # 특정 장르
python3 scripts/assign_background_images.py --vol 260402_hero-vol1
python3 scripts/assign_background_images.py --stats     # 사용 현황
```

### 0-2. 썸네일 자동 생성 (gen_vol_thumbnails.py)

`vol_song_images` DB에서 **1번 곡 배경이미지**를 조회해 Pillow로 텍스트를 합성합니다.

**썸네일 후킹 전략:**
- **인물 사진 자체가 후킹** — 자연스러운 실사 일본 거리 인물로 클릭 유도
- 텍스트는 장르 정보만 (최소화), 과장된 효과 없음
- "꾸밈없는 자연스러움" — 경쟁 채널이 텍스트 도배할 때 차별화 포인트

**스타일 (확정):**
- 배경 다크닝 없음 (이미지 원본 밝기 유지)
- **하단 좌/우** 동적 배치: 좌/우 열(column) 전체 표준편차 비교 → 인물 없는 쪽 자동 선택
- 두 박스(일어+영어) 가로 너비 통일 — 영어 박스 기준 고정, 일어 폰트 자동 축소하여 맞춤
- 일어 박스 텍스트 세로 중앙 정렬 (top offset 보정)
- 박스 스타일: **Glassmorphism 강화** — GaussianBlur(radius=25) + rgba(255,255,255,0.06) 극투명 오버레이 + 흰색 실선(alpha=100, width=1)
- 박스 너비: 자간 포함 실제 영어 텍스트 너비 기준으로 계산 (letter_spacing=3 포함)
- 폰트: `BlackHanSans.ttf` (영어 size=50, letter_spacing=3), `DelaGothicOne-Regular.ttf` (일어 — YouTube 썸네일 최적, Google Fonts OFL)

**색상 (확정):**
| 요소 | 색상 | 근거 |
|------|------|------|
| 일어 메인 텍스트 | `#FFD700` 골드 `(255,215,0)` | 5,000개 영상 분석 + IKXE 벤치마크 |
| `\|` 구분자 + カラオケ歌詞 | `#FFFFFF` 흰색 | 골드와 충돌 없이 조화 |
| 영어 메인 텍스트 | `#FFFFFF` 흰색 | |
| 보라색 | 금지 | AI 의심 트리거 |

**vol별 일어 서브타이틀:**
| vol | 일어 |
|-----|------|
| 260402_hero-vol1 | 一度聴いたら止まらない |
| 260402_hero-vol2 | 一日中頭の中に流れる |
| 260402_hero-vol3 | 聴いた瞬間に気分が上がる |
| 260402_ref-vol1  | 一度は聴いたことがある名曲 |
| 260402_ref-vol2  | 仕事中に流したら時間が溶ける |
| 260402_phonk-vol1 | The hardest Phonk tracks |

**저장 위치:** `{vol_dir}/thumbnails/{short_name}.png`

```bash
python3 scripts/gen_vol_thumbnails.py              # 전체
python3 scripts/gen_vol_thumbnails.py jpop         # 특정 장르
python3 scripts/gen_vol_thumbnails.py --vol 260402_hero-vol1
```

> **전제 조건:** `assign_background_images.py`를 먼저 실행해야 DB에 1번 곡 이미지가 등록됩니다.

---

## 1. 출력 스펙

| 항목 | 값 |
|------|-----|
| 해상도 | **1280 × 720 px** (YouTube 표준) |
| 포맷 | PNG (quality 95) |
| 파일명 | `{playlist_name}.png` |
| 저장 위치 | `{playlist_folder}/thumbnails/` |

---

## 2. 레이아웃 구조

```
┌──────────────────────────────────────┐
│                                      │
│        (배경 이미지, 전체 채움)        │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  하단 그라데이션 오버레이       │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ 라인1 (흰색, 메인 폰트)  │  │  │
│  │  │ 라인2 (강조색, 메인 폰트) │  │  │
│  │  │ 서브 (연회색, 작은 폰트)  │  │  │
│  │  └──────────────────────────┘  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### 텍스트 배치
| 요소 | x | y | 폰트 크기 |
|------|---|---|----------|
| 라인1 | 60px | H - 280px | 100~120px Bold |
| 라인2 (강조) | 60px | H - 170px | 110~130px Bold |
| 서브 텍스트 | 60px | H - 50px | 28px Regular |

### 오버레이
```python
# 하단 그라데이션: H/3 → H, alpha 0→210
for y in range(H // 3, H):
    alpha = int(210 * (y - H // 3) / (H - H // 3))
    # fill (0, 0, 0, alpha)

# 좌측 그라데이션: 0 → W/2, alpha 80→0
for x in range(0, W // 2):
    alpha = int(80 * (1 - x / (W // 2)))
    # fill (0, 0, 0, alpha)
```

### 텍스트 그림자
```python
def draw_shadow_text(draw, pos, text, font, fill, shadow=5):
    x, y = pos
    for dx in range(-shadow, shadow + 1):
        for dy in range(-shadow, shadow + 1):
            if dx == 0 and dy == 0:
                continue
            draw.text((x + dx, y + dy), text, font=font, fill=(0, 0, 0, 180))
    draw.text((x, y), text, font=font, fill=fill)
```

---

## 3. 색상 시스템

| 요소 | 기본값 | 비고 |
|------|--------|------|
| 라인1 | `#FFFFFF` | 고정 |
| 라인2 (강조) | `#FFD700` (골드) | 채널/무드별 변경 가능 |
| 서브 텍스트 | `#C8C8C8` | 고정 |
| 그림자 | `#000000` alpha 180 | 고정 |

### 채널별 강조색 추천
| 채널 | 강조색 | 이유 |
|------|--------|------|
| 季節のプレイリスト (jpop) | `#FFD700` 골드 | 따뜻한 감성 |
| Hazy Roast (cafe) | `#D4A574` 카페 브라운 | 커피 무드 |
| ZERO MERCY BEATS (phonk) | `#00FFFF` 시안 | 네온/에너지 |

### 금지 색상
- **보라색 계열 금지** (#8B5CF6, #7C3AED 등) — AI 생성 의심 트리거

---

## 4. 폰트 시스템

### 현재 사용 가능
| 폰트 | 경로 | index | 한글 | 용도 |
|------|------|-------|------|------|
| AppleSDGothicNeo ExtraBold | `/System/Library/Fonts/AppleSDGothicNeo.ttc` | 8 | ✅ | 메인 |
| AppleSDGothicNeo Regular | 위와 동일 | 3 | ✅ | 서브 |

### 다운로드 추천 (무료, 상업 가능)
| 폰트 | 특징 | 추천 용도 | 다운로드 |
|------|------|----------|---------|
| Black Han Sans | 두껍고 강렬 | 후킹 제목 | Google Fonts |
| Pretendard Bold/Black | 모던 고딕 | 범용 | GitHub releases |
| Noto Sans KR Black | 구글 기본 | 범용 | Google Fonts |
| Jua | 둥글고 친근 | 카페/힐링 채널 | Google Fonts |
| Do Hyeon | 제목 강조체 | 강조 라인 | Google Fonts |

### 폰트 저장 위치
```
~/Music/suno-channels/assets/fonts/
```

---

## 5. 배경 이미지 생성

### 스타일 카테고리

| 카테고리 | 프롬프트 키워드 | 적합 채널 |
|---------|---------------|----------|
| 레트로 애니 | `1990s retro anime illustration, cel shading, warm muted palette` | jpop |
| 실사 일본 여성 | `cinematic portrait, beautiful Japanese woman, golden hour` | jpop |
| 로파이 애니 | `lofi anime illustration, cozy room, warm lamp` | cafe |
| 사이버펑크 | `cyberpunk anime, neon Tokyo, glowing headphones` | phonk |
| 수채화풍 | `watercolor anime, soft edges, pastel colors` | jpop, cafe |

### 인물 프롬프트 규칙
```
필수 포함:
- "beautiful anime girl" 또는 "beautiful Japanese woman" (성별 명시)
- "large expressive eyes, detailed face close-up" (얼굴 디테일)
- "character taking up 60 percent of frame" (인물 비중)
- "upper body portrait composition" (구도)
- 음악 요소: "wearing headphones/earbuds", "holding guitar", "microphone" 등

금지:
- "anime character" (성별 중립 → 남자 나올 수 있음)
- "purple, violet, lavender" (AI 의심)
```

### 프롬프트 템플릿
```
{스타일}, beautiful anime girl with {헤어스타일},
large expressive eyes, detailed face close-up,
character taking up 60 percent of frame, upper body portrait,
wearing {음악 요소}, {표정},
{장소} with {소품 디테일}, {조명/시간대}, {무드},
16:9 widescreen
```

### 변수 예시
| 변수 | 옵션 |
|------|------|
| 헤어스타일 | short bob hair, long flowing black hair, twin tails, ponytail, wavy brown hair |
| 음악 요소 | headphones, earbuds, holding vinyl record, playing guitar, microphone |
| 표정 | gentle smile, dreamy expression, peaceful closed eyes, passionate expression, wistful look |
| 장소 | vintage cafe, record store, rooftop, train window, live house, bookstore |
| 조명 | golden hour, warm lamp light, neon glow, moonlight, sunset |
| 무드 | melancholic, hopeful, cozy, energetic, nostalgic |

### 이미지 생성 API

```python
import requests, subprocess, base64

PROJECT_ID = "project-3f3c3c98-9d26-4383-bcb"
token = subprocess.run(
    ["gcloud", "auth", "print-access-token"],
    capture_output=True, text=True
).stdout.strip()

URL = f"https://us-central1-aiplatform.googleapis.com/v1/projects/{PROJECT_ID}/locations/us-central1/publishers/google/models/imagen-3.0-generate-002:predict"

headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json"
}

payload = {
    "instances": [{"prompt": "YOUR_PROMPT_HERE"}],
    "parameters": {
        "sampleCount": 4,        # 1~4, 한번에 여러 장
        "aspectRatio": "16:9",   # 썸네일용
        "safetyFilterLevel": "block_few"
    }
}

resp = requests.post(URL, headers=headers, json=payload, timeout=120)
for i, pred in enumerate(resp.json()["predictions"]):
    with open(f"output_{i}.png", "wb") as f:
        f.write(base64.b64decode(pred["bytesBase64Encoded"]))
```

### API 최적화 규칙
| 규칙 | 값 |
|------|-----|
| sampleCount | **4** (최대, 효율적) |
| 병렬 호출 | **2~3개** 동시 OK |
| 4개 이상 동시 | 429 에러 (분당 쿼터) |
| 배치 간 대기 | **20~60초** |
| 인증 파일 | `~/Projects/claude/test/niche_bending/infra/credentials_account3.json` |
| 크레딧 | ~₩385,000 (2026-06-27 만료) |
| 이미지당 비용 | ~₩55 (~$0.04) |

### 결과 없음 (빈 predictions) 대응
- 안전 필터에 걸린 것 → 프롬프트에서 인물 묘사 완화 또는 재시도
- "girl" → "character" 로 바꾸면 안전 필터 통과하지만 남자 나올 수 있음
- **권장**: "anime girl"은 유지하되, 의상/포즈 묘사를 간접적으로

---

## 6. 후킹 제목 작성 규칙

### 데이터 기반 원칙 (5,000개 영상 분석)

| 순위 | 훅 유형 | 비율 | 예시 |
|------|---------|------|------|
| 1 | **호기심** | 47% | "왜 아무도 안 알려줬어?" |
| 2 | 유머 | 10% | "이건 반칙 아니냐" |
| 3 | 공감 | 10% | "나만 이거 무한반복?" |
| 4 | 질문 | 8% | "이 느낌 아는 사람?" |
| 5 | 반전 | 5% | "듣는 순간 봄이 와버림" |

### 호기심 갭 공식
```
[알려진 것] + [궁금한 것] → 클릭

예: "이 노래" (알려진) + "왜 아무도 안 알려줬어?" (궁금)
예: "후렴에서 터지는 그 느낌" (알려진) + "아는 사람?" (궁금)
```

### 채널별 톤 가이드

| 채널 | 톤 | 키워드 | 예시 |
|------|-----|--------|------|
| jpop (감성 락발라드) | 벅차오르는, 소름, 울컥 | 후렴, 소름, 명곡 | "후렴에서 터지는 그 느낌 아는 사람?" |
| cafe (인스트루멘탈) | 아늑한, 편안한, 몰입 | 틀어놓으면, 순삭, 시간 | "틀어놓으면 카페 사장님이 뭐냐고 물어봄" |
| phonk (에너지) | 강렬, 미침, 떡상 | 💀, 미침, 터짐 | "이게 왜 좋지??💀" |

### 금지 패턴
- ❌ 설명형: "J-POP 감성 플레이리스트 20곡"
- ❌ AI 언급: "AI가 만든", "Suno"
- ❌ Vol 번호: "Vol.1", "Part 2"
- ❌ "진짜" 3회 이상 반복

### 서브 텍스트 템플릿
```
{장르 키워드} {곡수}곡 · {길이} 플레이리스트

예: 감성 J-POP 20곡 · 1시간 플레이리스트
예: Cafe Jazz 15곡 · 45분 플레이리스트
예: Phonk Mix 40곡 · 1시간 플레이리스트
```

---

## 7. 전체 썸네일 생성 파이프라인 (자동화용)

```
입력:
  - playlist_folder: 플레이리스트 폴더 경로
  - channel: jpop | cafe | phonk
  - title_line1: 후킹 제목 1줄
  - title_line2: 후킹 제목 2줄 (강조)
  - bg_image: 배경 이미지 경로 (없으면 자동 생성)
  - accent_color: 강조색 hex (없으면 채널 기본값)

처리:
  1. bg_image 없으면 → Imagen API로 생성 (채널 스타일 + 프롬프트 템플릿)
  2. 이미지 로드 → 1280x720 크롭/리사이즈
  3. 그라데이션 오버레이 적용
  4. 텍스트 렌더링 (그림자 + 본문)
  5. 저장: {playlist_folder}/thumbnails/{playlist_name}.png

출력:
  - 1280x720 PNG 썸네일
```

### 자동화 함수 시그니처 (향후 구현)
```python
def generate_thumbnail(
    playlist_folder: str,
    channel: str,              # "jpop" | "cafe" | "phonk"
    title_line1: str,          # "이 노래 왜 아무도"
    title_line2: str,          # "안 알려줬어?"
    sub_text: str = None,      # None이면 자동 생성
    bg_image: str = None,      # None이면 Imagen으로 생성
    bg_prompt: str = None,     # bg_image 없을 때 사용할 프롬프트
    accent_color: str = None,  # None이면 채널 기본값
    font_main: str = None,     # None이면 기본 폰트
    font_sub: str = None,
) -> str:  # 저장된 파일 경로 반환
    ...
```

---

## 8. 이미지/폰트 에셋 위치

```
~/Music/suno-channels/
├── assets/
│   ├── fonts/              ← 한글 폰트 (다운로드 후)
│   └── bg_templates/       ← 채널별 배경 이미지 템플릿
├── docs/
│   ├── thumbnail_guide.md  ← 이 문서
│   ├── shorts_guide.md     ← 쇼츠 가이드
│   └── logs/               ← 작업 이력 (특정 영상 매칭표, 업로드 ID 등)
```
