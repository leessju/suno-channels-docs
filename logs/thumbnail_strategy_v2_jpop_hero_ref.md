# 썸네일 전략 V2 (2026-04-07)

> 다른 세션에서 이 문서만 읽으면 바로 작업 가능하도록 모든 결정 사항을 기록.

---

## 1. 디자인 컨셉 (확정)

### 비주얼 스타일
- **1990s 레트로 애니 일러스트** (cel shading, 따뜻한 머트 톤)
- retro_C_01.png (레코드샵 소녀)가 원본 레퍼런스
- 경쟁 채널 lofitokyo의 "TOKYO LO-FI" 썸네일 스타일 참조

### 인물 규칙
- **단발 보브컷 소녀** — 채널 아이덴티티 (모든 영상 동일 캐릭터)
- **이어폰/헤드폰 필수** — 음악 채널임을 즉시 인지
- **화면의 60% 이상 차지** — 상반신 클로즈업 (배경에 묻히지 않도록)
- **큰 눈 + 디테일한 얼굴** — "beautiful anime girl, large expressive eyes, detailed face close-up, character taking up 60 percent of frame, upper body portrait"
- **장면마다 다른 표정/의상/소품** — 같은 캐릭터인데 다른 무드

### 배경 규칙
- 장소별 소품 디테일 가득 (레코드, 커피, 악기, 책, 포스터 등)
- 인물 뒤로 밀리되 분위기 전달 역할
- 따뜻한 조명 (램프, 석양, 네온 등)

### 이미지 생성 도구
- **현재**: Vertex AI Imagen 3.0 API (`imagen-3.0-generate-002`)
- **GCP 프로젝트**: `project-3f3c3c98-9d26-4383-bcb`
- **인증**: `~/Projects/claude/test/niche_bending/infra/credentials_account3.json` (authorized_user)
- **토큰**: `gcloud auth print-access-token`
- **최적 설정**: `sampleCount: 4`, `aspectRatio: "16:9"`, `safetyFilterLevel: "block_few"`
- **병렬 호출 가능**: 2~3개 동시 OK, 4개 이상은 429 에러 (분당 쿼터)
- **쿼터 회피**: 배치 간 20~60초 대기
- **향후**: Google Flow 웹 (Nano Banana 2) CDP 방식 전환 검토 (퀄리티 더 좋음)

### 이미지 생성 프롬프트 템플릿
```
공통 스타일:
1990s retro anime illustration, warm muted color palette, detailed background,
cozy slice of life, cel shading, beautiful anime girl with short bob hair,
large expressive eyes, detailed face close-up, character taking up 60 percent
of frame, upper body portrait, wearing earbuds listening to music, 16:9 widescreen

장면별 추가 (예시):
+ resting chin on hand at vintage cafe, dreamy half-closed eyes, steaming coffee cup,
  golden evening light through window, autumn leaves outside, melancholic mood
```

### 주의사항
- "anime character" (성별 중립) → 남자가 나옴. 반드시 **"anime girl"** 명시
- 인물 묘사 없이 장소만 → 배경만 나옴. 반드시 인물 행동/외모 포함
- **보라색 사용 금지** (AI 생성 의심 트리거)

---

## 2. 후킹 제목 전략 (데이터 기반, 확정)

### 분석 근거
- 제목 훅 패턴 5,000개 영상 분석 (`niche_bending/docs/title_thumbnail_channel_report.md`)
- 바이럴 165개 훅×공유동기 교차 분석 (`niche_bending/docs/pattern-analysis-report.md`)
- 음악 VR 1,539개 키워드 분석 (`niche_bending/data/reports/music_vr_analysis.md`)
- IKXE 49개 영상 제목 패턴 (메모리: `project_ikxe_benchmark.md`)

### 핵심 규칙
1. **호기심 유발이 왕** — 47% 압도적 1위. "왜?", "어떻게?" 유발
2. **호기심 갭 공식**: `[알려진 것] + [궁금한 것]` → 클릭
3. **감정 트리거 필수**: 설명형 금지 → 의문문/감정 폭발/상황 체험 예고
4. **락발라드 톤 반영**: 우리 곡은 "emotional rock ballad, acoustic pop" — 벅차오르는/소름/울컥 (chill/귀여운 톤 금지)
5. **AI 언급 절대 금지** (역효과)
6. **"진짜" 남발 주의** — 1~2개만
7. **이모지 최소화**

### 제목 패턴 (검증된 것)
- 의문문: "이 노래 왜 아무도 안 알려줬어?" (호기심 갭)
- 체험 예고: "후렴에서 터지는 그 느낌 아는 사람?" (공감 + 호기심)
- 짧은 충격: "듣는 순간 봄이 와버림" (결과 제시)
- 공감형: "한 번쯤은 들어봤을 진짜 명곡만 모음" (이거 나야)
- 상황형: "틀어놓으면 시간이 진짜 순삭됨" (유용하다)

### VR 높은 제목 키워드 (참고)
- S등급: motivation (VR 36,590)
- A등급: playlist (7,197), phonk/trap (8,686), workout (8,058)
- B등급: 감성 (5,158), rain/비 (4,918), chill (6,678)
- 고VR 제목 공식: `[𝐏𝐥𝐚𝐲𝐥𝐢𝐬𝐭] {상황/감정} | {부제}` (유니코드 스타일링)

---

## 3. 최종 매칭 (확정)

| 영상 | 이미지 파일 | 무드 | 제목 | 서브 텍스트 |
|------|------------|------|------|-----------|
| hero-vol1 | `cafe_v5.png` | 옥상 석양 + 이어폰 + 희망 | 이 노래 왜 아무도 안 알려줬어? | 감성 J-POP 20곡 · 1시간 플레이리스트 |
| hero-vol2 | `pretty_02_01.png` | 비오는 카페 + 이어폰 + 몽환 | 후렴에서 터지는 그 느낌 아는 사람? | 감성 J-POP 20곡 · 1시간 플레이리스트 |
| hero-vol3 | `cafe_v1.png` | 가을 저녁 카페 + 단풍 + 멜랑콜리 | 듣는 순간 봄이 와버림 | 감성 J-POP 20곡 · 1시간 플레이리스트 |
| ref-vol1 | `cafe_v2.png` | 밤 재즈카페 + 네온 + 세련 | 한 번쯤은 들어봤을 진짜 명곡만 모음 | J-POP 명곡 20곡 · 1시간 플레이리스트 |
| ref-vol2 | `cafe_v4.png` | 겨울 눈 + 따뜻한 음료 + 평화 | 틀어놓으면 시간이 진짜 순삭됨 | J-POP 명곡 20곡 · 1시간 플레이리스트 |

---

## 4. 파일 위치

### 이미지 원본
```
~/Music/suno-channels/_raw/design_research/v3_anime/
├── cafe_v1.png          ← hero-vol3 (가을 저녁)
├── cafe_v2.png          ← ref-vol1 (밤 재즈카페)
├── cafe_v4.png          ← ref-vol2 (겨울 눈)
├── cafe_v5.png          ← hero-vol1 (옥상 석양)
├── pretty_02_01.png     ← hero-vol2 (비오는 카페)
├── (기타 후보 이미지 ~60장)
```

### 이전 썸네일 (V1, 교체 대상)
```
~/Music/suno-channels/jpop/260402_hero-vol1/thumbnails/
├── hero-vol1.png    ← 기존 (듀오톤 그라데이션 스타일, 교체)
├── hero-vol2.png    ← 기존 (교체)
├── hero-vol3.png    ← 기존 (교체)
├── ref-vol1.png     ← 기존 (교체)
├── ref-vol2.png     ← 기존 (교체)
```

### 최종 썸네일 출력 위치
```
~/Music/suno-channels/jpop/260402_hero-vol1/thumbnails/v2/
```

### 리서치 자료
```
~/Music/suno-channels/_raw/design_research/
├── ours_*.jpg              (우리 현재 영상 프레임 3장)
├── ikxe_*.jpg              (경쟁사 IKXE 영상 프레임 3장)
├── *경쟁 썸네일*.jpg        (27장)
├── gemini_design_direction.md  (Gemini 디자인 컨설팅 - 3안 비교)
├── flow_mockups/           (레이아웃 목업 6장)
├── flow_models/            (실사 인물 이미지 10장)
├── new_thumbnails/         (V1 썸네일 5장 - 실사 배경, 교체 대상)
└── v3_anime/               (애니 이미지 ~60장, 최종 후보 여기서 선정)
```

---

## 5. 썸네일 제작 스펙

### 해상도
- **1280 × 720 px** (YouTube 표준)

### 텍스트 레이아웃
```
┌─────────────────────────────────────┐
│                                     │
│         (애니 인물 이미지)           │
│                                     │
│  ┌─────────────────────────────┐    │
│  │ 라인1: 흰색 (100-120px)     │    │
│  │ 라인2: 골드 강조 (110-130px) │    │
│  │ 서브: 연회색 (28px)          │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
- 텍스트 위치: 좌측 하단 (margin-left: 60px)
- y 시작: H - 280px
- 라인 간격: 110px (라인1→라인2), 120px (라인2→서브)
```

### 오버레이
- 하단 1/3 → 그라데이션 블랙 오버레이 (alpha 0→210)
- 좌측 절반 → 약한 블랙 오버레이 (alpha 0→80)
- 목적: 텍스트 가독성 확보

### 색상
| 요소 | 색상 | 비고 |
|------|------|------|
| 라인1 | `#FFFFFF` (흰색) | |
| 라인2 (강조) | `#FFD700` (골드) | 영상별 변경 가능 |
| 서브 텍스트 | `#C8C8C8` (연회색) | |
| 텍스트 그림자 | `#000000` alpha 180, 5px | 전방향 |
| **보라색 금지** | | AI 의심 트리거 |

### 폰트 (미확정 — 다음 작업)
현재 사용:
- `/System/Library/Fonts/AppleSDGothicNeo.ttc` index 8 (ExtraBold)
- 한글 지원되는 유일한 시스템 폰트

다운로드 후보 (무료, 상업 이용 가능):
| 폰트 | 특징 | 용도 |
|------|------|------|
| **Black Han Sans** | 두껍고 강렬, 제목용 최적 | 라인1, 라인2 |
| **Pretendard Bold/Black** | 모던 고딕, 가독성 최고 | 전체 |
| **Noto Sans KR Black** | 구글 기본, 안정적 | 전체 |
| **Jua** | 둥글고 친근한 | 서브 텍스트 |
| **Do Hyeon** | 제목용 강조체 | 라인2 강조 |

**→ 다음 세션에서 폰트 다운로드 후 5종 × 5영상 = 25개 변형 생성하여 최종 선택**

---

## 6. 썸네일 생성 코드

### Python (Pillow) 기본 구조
```python
from PIL import Image, ImageDraw, ImageFont

W, H = 1280, 720

# 1. 이미지 로드 + 16:9 크롭 + 리사이즈
img = Image.open("cafe_v5.png").convert("RGBA")
# crop to 16:9 center, resize to 1280x720

# 2. 그라데이션 오버레이 (하단 + 좌측)
overlay = Image.new("RGBA", (W, H), (0,0,0,0))
draw_ov = ImageDraw.Draw(overlay)
for y in range(H//3, H):
    alpha = int(210 * (y - H//3) / (H - H//3))
    draw_ov.rectangle([(0,y),(W,y+1)], fill=(0,0,0,alpha))
for x in range(0, W//2):
    alpha = int(80 * (1 - x/(W//2)))
    draw_ov.rectangle([(x,0),(x+1,H)], fill=(0,0,0,alpha))
img = Image.alpha_composite(img, overlay).convert("RGB")

# 3. 텍스트 (그림자 + 본문)
draw = ImageDraw.Draw(img)
font_main = ImageFont.truetype(FONT_PATH, 100, index=8)
font_accent = ImageFont.truetype(FONT_PATH, 110, index=8)
font_sub = ImageFont.truetype(FONT_PATH, 28, index=3)

# 그림자 함수: 5px 전방향 검정
def draw_shadow_text(draw, pos, text, font, fill, shadow=5):
    x, y = pos
    for dx in range(-shadow, shadow+1):
        for dy in range(-shadow, shadow+1):
            if dx==0 and dy==0: continue
            draw.text((x+dx,y+dy), text, font=font, fill=(0,0,0,180))
    draw.text((x,y), text, font=font, fill=fill)

margin = 60
y_start = H - 280
draw_shadow_text(draw, (margin, y_start), "이 노래 왜 아무도", font_main, (255,255,255))
draw_shadow_text(draw, (margin, y_start+110), "안 알려줬어?", font_accent, (255,215,0))
draw_shadow_text(draw, (margin, y_start+230), "감성 J-POP 20곡 · 1시간 플레이리스트", font_sub, (200,200,200), shadow=2)

img.save("hero-vol1.png", quality=95)
```

### 기존 생성 스크립트
- `/tmp/gen_new_thumbs.py` — V1 썸네일 생성 (실사 배경, 교체 대상)
- 위 코드를 V2 이미지로 교체하면 됨

---

## 7. 다음 작업 (이 세션 또는 다음 세션)

1. **한글 폰트 다운로드** (Black Han Sans, Pretendard, Noto Sans KR 등)
2. **5종 폰트 × 5영상 = 25개 썸네일 변형 생성**
3. **최종 폰트 확정** (사용자 선택)
4. **YouTube 썸네일 교체** (browser-use 또는 API)
5. **Remotion 영상 레이아웃 재설계** (별도 세션 — `gemini_design_direction.md` 참조)
6. **100곡 재렌더링** (모바일 가사 가독성 개선 후)

---

## 8. YouTube 채널 정보 (참고)

| 채널 | ID | 토큰 |
|------|-----|------|
| 季節のプレイリスト (jpop) | UC2ofS0Y6ynIjRSVwexUUWQg | `~/.claude/youtube_token_jpop.json` |
| Hazy Roast (cafe) | UCSvzzpXpaXwRWi3G-grk_7A | `~/.claude/youtube_token_cafe.json` |
| ZERO MERCY BEATS (phonk) | UC0OSrx55lFo7ELCMo88mxUw | `~/.claude/youtube_token_phonk.json` |

### 현재 업로드 상태 (교체 대상)
- hero-vol1: https://youtu.be/ExLKDFLNONg (Public, quota 심사용)
- ref-vol1: https://youtu.be/4b2D86SNSNA (Public)
- ref-vol2: https://youtu.be/yCwsR4PXkLg (Public)
- hero-vol2, vol3: 모바일 가사 문제로 삭제됨

---

## 9. 의사결정 이력 (왜 이렇게 됐는지)

1. **왜 레트로 애니?** — 경쟁 채널(lofitokyo, nekonomaki) 썸네일 분석 → 애니 스타일이 J-POP 채널에 가장 적합. 실사 일본 여성도 시도했으나 "한국 사람 같다"는 피드백
2. **왜 단발 보브컷?** — pretty_02_01.png(카페 단발 소녀)가 사용자 선택. cafe_v1~v6 모두 이 캐릭터 기반으로 무드 변형
3. **왜 후킹 제목?** — 이전 V1 제목("퇴근길에 매일 듣게 되는 J-POP 플레이리스트")은 설명형이라 클릭 유인 약함. IKXE 벤치마크 + 5,000개 영상 분석에서 의문문/감정 트리거가 2배 이상 효과적
4. **왜 골드 강조?** — 따뜻한 애니 톤과 조화 + 시인성 높음. 보라색은 AI 의심 트리거로 금지
5. **왜 cafe_v5가 hero-vol1?** — 옥상 석양 = 희망/시작 무드 → "왜 아무도 안 알려줬어?" (발견의 느낌)과 매칭
