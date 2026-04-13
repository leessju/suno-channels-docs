# YouTube 업로드 가이드 (풀영상 + 쇼츠)

> 2026-04-07 확정. browser-use + YouTube API 하이브리드 방식.
> 모든 채널/플레이리스트에 적용 가능한 범용 가이드.

---

## 1. 핵심 전략

### 도구 분담 (하이브리드 방식)

| 작업 | 도구 | 이유 |
|------|------|------|
| 영상 파일 업로드 | **browser-use** | quota 0, 대용량 가능 |
| 썸네일 업로드 | **browser-use** | upload 명령 안정적 |
| Not made for kids | **browser-use** | 필수 필드, 기본 설정 |
| Private 저장 | **browser-use** | 업로드 완료 처리용 |
| **제목/설명** | **YouTube API** | browser-use 텍스트 입력 불안정 (한글/다국어/특수문자) |
| **공개 설정** | **YouTube API** | 일괄 처리 |
| **태그/카테고리** | **YouTube API** | snippet.update |
| **댓글** | **YouTube API** | browser-use 댓글 불안정 (Polymer submit 문제) |

### 채널 유형별 주의사항

| 채널 유형 | browser-use | API | 비고 |
|----------|:---:|:---:|------|
| **기본 채널** (예: nicejames) | ✅ 가능 | ✅ 가능 | 모두 사용 가능 |
| **브랜드 채널** (예: 季節のプレイリスト) | ⚠️ 직접 URL로 접근 필요 | ✅ 가능 | browser-use 채널 전환 불가, URL 직접 이동 |

**브랜드 채널 URL 접근:**
```
https://studio.youtube.com/channel/{CHANNEL_ID}
```

---

## 2. 사전 준비

### 2-1. 인증 토큰
채널별 OAuth 토큰 필요:
```
~/.claude/youtube_token_{channel}.json
```

### 2-2. 영상/썸네일/메타데이터
```
{playlist_folder}/
├── 04_final/{playlist_name}_final.mp4   ← 업로드 대상 (※ _full.mp4 아님, _final.mp4)
├── thumbnails/{playlist_name}.png        ← 1280x720
└── youtube_meta.json                     ← 제목/설명/태그
```

### 2-3. youtube_meta.json 포맷
```json
{
  "title": "한번 틀면 끝까지 듣게 되는 감성 J-POP 20곡 [가사/歌詞/Lyrics] | 51분 플레이리스트",
  "description": "🎵 채널명\n\n...설명...\n\n━━━━━━━━━━━━━━━━━━━\n📋 Tracklist\n━━━━━━━━━━━━━━━━━━━\n0:00 곡1\n...\n\n#해시태그",
  "tags": ["J-POP", "playlist", "감성JPOP", "..."],
  "category": "10",
  "privacy": "public"
}
```

### 2-4. 제목 전략 (VR 분석 기반)
- **검색 최적화형** 사용 (설명형, 정보 명시)
- 후킹 문구는 **썸네일**에만 사용 (역할 분담)
- 필수 포함: `[가사/歌詞/Lyrics]`, `{N}분 플레이리스트`
- **금지**: Vol 번호, AI 언급, 제목-썸네일 중복

### 2-5. 설명 템플릿
```
🎵 {채널명}

{한 줄 훅}
{상황 묘사: 공부, 작업, 퇴근길에 틀어놓으세요.}

━━━━━━━━━━━━━━━━━━━
📋 Tracklist
━━━━━━━━━━━━━━━━━━━
0:00 {곡1}
3:25 {곡2}
...

━━━━━━━━━━━━━━━━━━━
🎧 Channel: {채널명}
🔔 Subscribe for more {장르} playlists!

#{해시태그들}
```

---

## 3. 업로드 파이프라인

### Phase 1: browser-use로 파일 업로드

```python
import subprocess
import time

PROFILE = "nicejames"
CHANNEL_ID = "UC2ofS0Y6ynIjRSVwexUUWQg"  # 季節のプレイリスト

def bu(*args, timeout=60):
    cmd = ["browser-use", "--headed", "--profile", PROFILE] + list(args)
    return subprocess.run(cmd, capture_output=True, text=True, timeout=timeout).stdout.strip()

def upload_video_file(video_path, thumb_path):
    # 1. 채널 대시보드 이동
    bu("open", f"https://studio.youtube.com/channel/{CHANNEL_ID}")
    time.sleep(4)

    # 2. Upload 버튼 클릭 (아리아 레이블 셀렉터 안정적)
    bu("eval", 'document.querySelector("[aria-label=\\"Upload videos\\"]")?.click(); "clicked"')
    time.sleep(3)

    # 3. file input 인덱스 동적 찾기
    state = bu("state")
    file_idx = None
    for line in state.split("\n"):
        if "type=file" in line and "name=Filedata" in line:
            idx_match = line.split("[")[1].split("]")[0] if "[" in line else None
            if idx_match:
                file_idx = idx_match
                break

    # 4. 영상 파일 업로드
    bu("upload", file_idx, video_path, timeout=120)
    time.sleep(10)  # 업로드 시작 대기

    # 5. 제목 임시 설정 (textContent 직접, 길이 제한 회피)
    bu("eval", '''
        const t = document.querySelector("#title-textarea #textbox");
        t.focus(); t.textContent = "temp_upload";
        t.dispatchEvent(new Event("input", {bubbles: true}));
    ''')
    time.sleep(1)

    # 6. 썸네일 업로드
    bu("scroll", "up")
    time.sleep(1)
    state = bu("state")
    thumb_idx = None
    for line in state.split("\n"):
        if "type=file" in line and "accept=image" in line:
            idx_match = line.split("[")[1].split("]")[0]
            thumb_idx = idx_match
            break
    bu("upload", thumb_idx, thumb_path, timeout=30)
    time.sleep(2)

    # 7. Not made for kids 선택
    bu("scroll", "down")
    time.sleep(1)
    state = bu("state")
    for line in state.split("\n"):
        if "NOT_MFK" in line:
            idx = line.split("[")[1].split("]")[0]
            bu("click", idx)
            break
    time.sleep(1)

    # 8. 업로드 처리 완료 대기 (Checks 완료까지)
    for i in range(30):
        progress = bu("eval", 'document.querySelector(".progress-label")?.textContent || "waiting"')
        if "complete" in progress.lower() or "no issues" in progress.lower():
            break
        time.sleep(15)

    # 9. Next 3번 (Details → Elements → Checks → Visibility)
    state = bu("state")
    next_idx = None
    for line in state.split("\n"):
        if "aria-label=Next" in line:
            next_idx = line.split("[")[1].split("]")[0]
            break
    for _ in range(3):
        bu("click", next_idx)
        time.sleep(2)

    # 10. Private 선택 (API로 나중에 Public 전환)
    state = bu("state")
    for line in state.split("\n"):
        if "name=PRIVATE" in line and "radio-button" in line:
            idx = line.split("[")[1].split("]")[0]
            bu("click", idx)
            break
    time.sleep(1)

    # 11. Save 클릭
    state = bu("state")
    for line in state.split("\n"):
        if "aria-label=Save" in line:
            idx = line.split("[")[1].split("]")[0]
            bu("click", idx)
            break
    time.sleep(5)

    return True
```

### Phase 2: 업로드된 영상 ID 수집 (API)

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

creds = Credentials.from_authorized_user_file(token_path)
youtube = build("youtube", "v3", credentials=creds)

# uploads playlist로 모든 영상 조회 (Private 포함)
ch = youtube.channels().list(part="contentDetails", id=channel_id).execute()
uploads_id = ch["items"][0]["contentDetails"]["relatedPlaylists"]["uploads"]

req = youtube.playlistItems().list(
    part="snippet,status",
    playlistId=uploads_id,
    maxResults=50
)
resp = req.execute()

for item in resp.get("items", []):
    vid = item["snippet"]["resourceId"]["videoId"]
    title = item["snippet"]["title"]
    privacy = item["status"]["privacyStatus"]
    print(f"{vid} | {privacy} | {title}")
```

### Phase 3: API로 메타데이터 일괄 업데이트

```python
def update_video_meta(video_id, meta_json_path, privacy="public"):
    with open(meta_json_path) as f:
        meta = json.load(f)

    body = {
        "id": video_id,
        "snippet": {
            "title": meta["title"],
            "description": meta["description"],
            "tags": meta.get("tags", []),
            "categoryId": meta.get("category", "10"),
        },
        "status": {
            "privacyStatus": privacy,
            "selfDeclaredMadeForKids": False,
        },
    }

    youtube.videos().update(part="snippet,status", body=body).execute()
```

### Phase 4: 댓글 달기 (API)

```python
def post_comment(video_id, hook_text, full_video_id):
    """쇼츠에 풀영상 링크 댓글 달기"""
    comment = (
        f"{hook_text}, 모았습니다\n\n"
        f"🎵아래 플레이리스트로 끝까지 들어보세요. 감사합니다.\n\n"
        f"👉 https://youtu.be/{full_video_id}"
    )

    youtube.commentThreads().insert(
        part="snippet",
        body={
            "snippet": {
                "videoId": video_id,
                "topLevelComment": {
                    "snippet": {"textOriginal": comment}
                }
            }
        }
    ).execute()
```

---

## 4. 댓글 규칙

### 4-1. 형식 (반드시 3줄, 빈 줄 포함)
```
{후킹 제목}, 모았습니다

🎵아래 플레이리스트로 끝까지 들어보세요. 감사합니다.

👉 https://youtu.be/{video_id}
```

### 4-2. 제약
- **Private 영상에 댓글 불가** → 반드시 Public 전환 후 댓글
- **쇼츠에 풀영상 링크 댓글** — 시청자를 풀영상으로 유도
- **자기 영상 self-comment OK** — 채널 소유자 본인 댓글

### 4-3. quota
- `commentThreads.insert` = **50 units/회**
- 일일 한도 10,000 units → 하루 최대 200개
- 100곡 쇼츠 댓글 = 5,000 units (일일 한도의 절반)

---

## 5. browser-use 핵심 패턴 (하드 확인된 것)

### 5-1. 세션 관리
```bash
# 기존 세션 충돌 시 닫기
browser-use close

# 항상 --headed + --profile로 시작
browser-use --headed --profile nicejames <command>
```

### 5-2. 요소 인덱스 동적 찾기 (하드코딩 금지)
```bash
# state로 요소 목록 조회
STATE=$(browser-use --headed --profile nicejames state)

# grep으로 인덱스 추출
IDX=$(echo "$STATE" | grep "aria-label=Next" | grep -o '\[[0-9]*\]' | tr -d '[]')
browser-use --headed --profile nicejames click $IDX
```

### 5-3. JavaScript eval 패턴
```bash
# 안정적: document.querySelector + optional chaining
browser-use --headed --profile nicejames eval '
    document.querySelector("[aria-label=\"Upload videos\"]")?.click();
    "clicked"
'

# ❌ 불안정: 복잡한 async/await, 한글 문자열
# ❌ eval 반환값은 항상 "result: ..." 형태로 나오는데
#    한글 반환 시 None으로 표시되기도 함 — 실행 자체는 될 수 있음
```

### 5-4. textContent 직접 설정 (텍스트 입력 안정화)
```javascript
const el = document.querySelector("#title-textarea #textbox");
el.focus();
el.textContent = "원하는 텍스트";
el.dispatchEvent(new Event("input", {bubbles: true}));
```

이 방법은 contenteditable 요소의 기존 내용을 **완전히 교체**합니다.
`type` 명령은 기존 내용에 **추가**되므로 주의.

### 5-5. 파일 업로드 (upload 명령)
```bash
# 반드시 인덱스 사용 (셀렉터 아님)
browser-use --headed --profile nicejames upload <INDEX> <file_path>

# state에서 인덱스 찾기
grep "type=file" state.txt | grep -o '\[[0-9]*\]'
```

---

## 6. 주의사항 체크리스트

### 업로드 전
- [ ] 영상 파일 존재 (`04_final/{vol}_full.mp4`)
- [ ] 썸네일 존재 (`thumbnails/{vol}.png`, 1280x720)
- [ ] `youtube_meta.json` 작성 완료
- [ ] 채널 OAuth 토큰 유효
- [ ] 15분 초과 영상이면 채널 전화번호 인증 확인

### 업로드 중 (browser-use)
- [ ] `--headed` 플래그 필수 (headless는 파일 다이얼로그 문제)
- [ ] `--profile` 플래그로 세션 유지
- [ ] 한 영상씩 순차 처리 (병렬 금지)
- [ ] 각 단계 후 screenshot으로 검증
- [ ] 제목은 `textContent` 직접 설정 (type 사용 금지 — 기존 내용에 추가됨)

### 업로드 후 (API)
- [ ] uploads playlist로 영상 ID 수집
- [ ] `videos.update`로 제목/설명/공개설정 업데이트
- [ ] Public 영상에 대해서만 댓글 가능
- [ ] 결과 저장 (`upload_results.json`)

### quota 관리
- [ ] 업로드는 browser-use (quota 0)
- [ ] 메타 업데이트: `videos.update` = 50 units/건
- [ ] 댓글: `commentThreads.insert` = 50 units/건
- [ ] 자막: `captions.insert` = 400 units/건
- [ ] 일일 한도: 10,000 units

---

## 7. 트러블슈팅

### "Your title is too long" 에러
**원인**: browser-use `type` 명령이 기존 제목에 추가해서 길이 초과
**해결**: `textContent` 직접 설정
```javascript
const t = document.querySelector("#title-textarea #textbox");
t.focus();
t.textContent = "새 제목";
t.dispatchEvent(new Event("input", {bubbles: true}));
```

### Next 버튼 disabled
**원인**: 필수 필드 누락 (제목 에러, Not made for kids 미선택 등)
**진단**:
```javascript
document.querySelectorAll("[invalid]")
    .forEach(e => console.log(e.textContent));
```

### Close 버튼 disabled
**원인**: 업로드 처리 중 또는 필드 에러
**해결**: 에러 해소 후 자동 활성화, 또는 영상 처리 완료 대기

### 설명란에 제목이 들어감 (포커스 오류)
**원인**: `type` 명령 실행 시 포커스가 설명란이 아닌 제목란에 있음
**해결**: `type` 사용 금지, `textContent` + `insertLineBreak` 사용
**최선책**: 제목/설명은 API로 후처리

### Private 영상에 댓글 403 에러
**원인**: Private 영상은 댓글 불가
**해결**: Public 전환 후 댓글

### 브랜드 채널 업로드 실패
**원인**: browser-use가 브랜드 채널로 자동 전환 불가
**해결**: 채널 ID URL로 직접 이동
```
https://studio.youtube.com/channel/{CHANNEL_ID}
```

### 쇼츠 업로드 Close 다이얼로그 실패
**해결**: `offsetParent !== null` 필터 필수
```javascript
Array.from(document.querySelectorAll('button'))
    .find(b => b.getAttribute('aria-label') === 'Close' && b.offsetParent !== null)
    ?.click()
```

---

## 8. 전체 자동화 함수 시그니처 (향후 구현)

```python
def upload_playlist(
    playlist_folder: str,      # ~/Music/suno-channels/jpop/260402_hero-vol2
    channel: str,              # "jpop" | "cafe" | "phonk"
    privacy: str = "public",   # "public" | "private" | "unlisted"
    post_comment: bool = True, # 댓글 달기 여부
) -> dict:
    """
    전체 업로드 파이프라인:
    1. browser-use로 영상+썸네일 업로드, Private 저장
    2. API로 영상 ID 수집
    3. API로 메타데이터 업데이트 (제목/설명/공개설정)
    4. API로 댓글 달기 (Public인 경우)

    Returns:
        {
            "video_id": "...",
            "url": "https://youtu.be/...",
            "privacy": "public",
            "commented": True
        }
    """
    ...


def upload_playlist_batch(
    playlist_folders: list[str],
    channel: str,
    privacy: str = "public",
) -> dict:
    """여러 플레이리스트 일괄 업로드"""
    ...
```

---

## 9. YouTube 채널 정보

| 채널 | ID | 토큰 | 15분+ | 유형 |
|------|-----|------|:---:|---|
| 季節のプレイリスト | `UC2ofS0Y6ynIjRSVwexUUWQg` | `youtube_token_jpop.json` | ✅ | 기본 |
| Lucid White | `UCoHpJmMju00FPBogQr6D_Kw` | `youtube_token_lucidwhite.json` | ✅ | 기본 |
| Hazy Roast | `UCSvzzpXpaXwRWi3G-grk_7A` | `youtube_token_cafe.json` | ❌ | 브랜드 |
| ZERO MERCY BEATS | `UC0OSrx55lFo7ELCMo88mxUw` | `youtube_token_phonk.json` | ❌ | 브랜드 |

### 채널별 업로드 방식
| 채널 | 영상 업로드 | 메타 업데이트 | 댓글 |
|------|-----------|-------------|------|
| jpop (기본) | browser-use | API | API |
| Lucid White (기본) | browser-use | browser-use (제목 고정) | browser-use |
| cafe (브랜드) | browser-use (URL 이동) or API | API | API |
| phonk (브랜드) | browser-use (URL 이동) or API | API | API |

### Save vs Publish 버튼
| Privacy 설정 | 버튼 레이블 |
|-------------|-----------|
| PRIVATE / UNLISTED | **Save** |
| PUBLIC | **Publish** |

> ⚠️ Private 업로드 시 "Publish" 버튼을 기다리면 영원히 타임아웃됨. Privacy에 따라 버튼 레이블이 달라지므로 반드시 분기 처리.
> ```python
> BTN_LABEL = "Save" if PRIVACY in ("PRIVATE", "UNLISTED") else "Publish"
> ```

---

## 10. 오늘 배운 핵심 교훈

1. **browser-use 텍스트 입력은 불안정** → API로 후처리
2. **type vs textContent** — type은 기존 내용에 추가, textContent는 교체
3. **인덱스 하드코딩 금지** — state에서 동적으로 찾기
4. **병렬 업로드 금지** — 세션 충돌, 매번 순차 처리
5. **제목 길이 제한** — YouTube는 100자 제한, 초과 시 모든 버튼 disabled
6. **Private에는 댓글 불가** — Public 전환 후 댓글
7. **5개 영상 기준 소요 시간**: 머지 25분 + 업로드 30분 + 메타 1분 + 댓글 1분 ≈ **총 1시간**
