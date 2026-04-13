# STEP 9: 미래 작업 (별건)

> ⚠️ **주의**: 이 문서는 **아직 구현되지 않은 개선 사항**들을 기록한 것입니다.
> step0~step5의 가이드는 실제 작업 경험 기반이지만, 이 문서의 항목들은 **향후 자동화 완성을 위해 필요한 추가 작업**입니다.
> 착수 전 사용자 확인 필수.

---

## 분류

| 우선순위 | 항목 수 | 설명 |
|:-:|:-:|------|
| 🔴 중요 | 4 | 자동화 완성에 필수. 에러 복구, 상태 관리, 모니터링, 성과 측정 |
| 🟡 중간 | 4 | 품질 개선. QA, 리소스, 백업, 기획 |
| 🟢 낮음 | 4 | 선택적. A/B 테스트, 채널 확장, 커뮤니티, AI 감지 |
| 🔵 특별 | 2 | 메모리 통합, 버전 관리 |

---

## 🔴 중요 (자동화 필수)

### 1. 에러 복구 / 재시도 전략

#### 문제
현재는 각 단계(step1~5)에서 실패하면 어디서부터 재시작할지 불명확. 중간 체크포인트 규칙이 분산되어 있음.

#### 필요한 것
- **체크포인트 파일** 표준화: `{playlist_folder}/.state.json`
  ```json
  {
    "stage": "step2_audio",
    "substage": "whisper_done",
    "completed_tracks": ["01", "02", "03"],
    "failed_tracks": [],
    "last_updated": "2026-04-07T12:34:56",
    "retry_count": {"04": 2}
  }
  ```
- **재시작 기준**:
  - STEP 1 실패: 실패한 곡만 재생성 (DB 기반)
  - STEP 2 실패: Demucs 캐시 활용, Whisper는 실패 곡만
  - STEP 3 실패: 렌더 캐시 활용 (이미 구현됨 — render_playlist.sh 건너뛰기)
  - STEP 4 실패: 이미지 생성 쿼터 회복 대기 후 재시도
  - STEP 5 실패: Private으로 저장된 상태 → API로 메타만 재업데이트
- **재시도 한도**: 각 곡당 3회, 초과 시 human review 요청
- **dead letter queue**: 3회 실패한 곡은 별도 `_failed/` 폴더로 이동

#### 작성 대상
`step9_a_error_recovery.md` (신규)

---

### 2. 로그 / 모니터링

#### 문제
파이프라인 실행 중 어디서 멈췄는지, 얼마나 진행됐는지 추적 불가.

#### 필요한 것
- **통합 로그 경로**: `{playlist_folder}/.logs/`
  ```
  .logs/
  ├── step1_suno.log
  ├── step2_demucs.log
  ├── step2_whisper.log
  ├── step3_render.log
  ├── step3_merge.log
  ├── step4_thumbnail.log
  ├── step5a_upload.log
  └── step5b_shorts.log
  ```
- **로그 포맷** (구조화):
  ```
  [2026-04-07 12:34:56] [step2] [track:01] [demucs] [success] duration=22.3s
  [2026-04-07 12:35:10] [step2] [track:02] [whisper] [error] reason="timeout"
  ```
- **진행률 대시보드** (선택):
  - 로컬 HTML: `{playlist_folder}/dashboard.html` (이미 일부 구현: `upload_dashboard.html`)
  - Telegram 알림: 주요 이벤트 (단계 완료, 에러 발생)
- **실시간 모니터링**: `tail -f .logs/*.log` 패턴

#### 작성 대상
`step9_b_logging.md` (신규)

---

### 3. 상태 관리 (DB status 필드)

#### 문제
`gems.db`의 `suno_tracks.status`, `suno_playlists.status` 필드 전이 규칙이 문서에 없음.

#### 필요한 것
- **트랙 상태 전이도**:
  ```
  planning
    ↓ step1
  generating
    ↓
  downloaded
    ↓ step2
  audio_processing
    ↓
  audio_done
    ↓ step3
  rendering
    ↓
  rendered
    ↓ step5
  uploaded
    ↓
  published
  ```
- **플레이리스트 상태 전이도**:
  ```
  planning → generating → audio_done → rendered → merged
    → thumbnails_ready → uploaded → public
  ```
- **상태 쿼리 헬퍼**:
  ```python
  def get_next_stage(playlist_id: int) -> str:
      """현재 상태에서 다음 단계 반환"""
      ...

  def get_failed_tracks(playlist_id: int) -> list:
      """실패 상태 트랙 목록"""
      ...
  ```
- **상태 변경 로그**: 별도 테이블 `track_status_history`

#### 작성 대상
`step9_c_state_management.md` (신규)

---

### 4. 성과 측정 (YouTube Analytics)

#### 문제
업로드 후 조회수, VR(조회수/구독자 비율), 클릭률 등 성과 데이터를 수집하지 않음.
데이터 없이는 "어떤 썸네일/제목이 잘 먹혔는지" 판단 불가.

#### 필요한 것
- **YouTube Analytics API 연동**:
  - 조회수, 시청 시간, 평균 시청률, CTR, 노출수
  - 일/주/월 단위 추적
- **DB 테이블**:
  ```sql
  CREATE TABLE video_stats (
    video_id TEXT,
    date DATE,
    views INTEGER,
    watch_time_seconds INTEGER,
    ctr REAL,
    impressions INTEGER,
    avg_view_duration REAL,
    subscribers_gained INTEGER,
    PRIMARY KEY (video_id, date)
  );
  ```
- **일일 수집 크론**: 매일 아침 전일 데이터 수집
- **성과 대시보드**:
  - 플레이리스트별 VR 순위
  - 썸네일 A/B 성과 비교
  - 제목 패턴별 CTR
- **피드백 루프**: 성과 데이터 기반으로 다음 플레이리스트 제목/썸네일 자동 최적화

#### 작성 대상
`step9_d_analytics.md` (신규)

---

## 🟡 중간 (품질)

### 5. QA 체크리스트 통합

#### 문제
각 단계(step1~5) 말미에 체크리스트가 있지만 파편화. "다음 단계로 넘어가도 되는가?" 판정 기준 없음.

#### 필요한 것
- **gatekeeper 스크립트**: 각 단계 완료 후 자동 검증
  ```bash
  python qa.py --stage step2 --playlist 260402_hero-vol1
  # 출력: ✅ 18/20 통과, ❌ 2곡 실패 (목록)
  ```
- **정량 기준**:
  - STEP 1: 20곡 DB에 v1+v2 suno_id 존재 + mp3/jpeg/txt 3종 세트
  - STEP 2: 18/20 이상 8항목 검증 통과 (90% 기준)
  - STEP 3: 20개 mp4 존재 + 오디오 볼륨 정상 + duration 일치
  - STEP 4: 썸네일 5장 존재 + 1280×720 + 파일 크기 > 500KB
  - STEP 5: 영상 ID 수집 + 메타 업데이트 성공 + 공개 설정 일치

#### 작성 대상
`step9_e_qa_gates.md` (신규)

---

### 6. 리소스 / 쿼터 모니터링

#### 문제
GCP 크레딧, YouTube quota, 2Captcha 잔액 등을 수동 체크. 중요한 순간에 고갈 가능.

#### 필요한 것
- **일일 자동 체크 스크립트**:
  ```python
  def check_all_resources() -> dict:
      return {
          "gcp_credit": get_gcp_balance(),          # 크레딧 잔액
          "gcp_credit_expires": "2026-06-27",
          "youtube_quota_used": get_quota_used(),   # 일일 사용량
          "youtube_quota_limit": 10000,
          "captcha_balance": get_captcha_balance(),
          "suno_cookie_valid": check_suno_auth(),
      }
  ```
- **경고 임계치**:
  - GCP 크레딧 < ₩50K: 경고
  - YouTube quota > 8,000: 경고 (80% 사용)
  - Captcha < $5: 경고
- **알림**: Telegram 또는 데스크톱 알림
- **기록**: 일일 사용량 DB 저장

#### 작성 대상
`step9_f_resource_monitoring.md` (신규)

---

### 7. 백업 정책

#### 문제
완성된 영상/메타데이터가 로컬 + YouTube에만 있음. 로컬 디스크 손상 시 영상 유실.

#### 필요한 것
- **백업 대상**:
  - 01_songs/ (mp3, jpeg, lyrics.json) — 원본 소스
  - 04_final/ (풀영상) — 렌더 결과
  - 05_shorts/ (쇼츠) — 렌더 결과
  - youtube_meta.json — 메타데이터
  - thumbnails/ — 썸네일
- **백업 위치**:
  - 1차: 외장 SSD (로컬)
  - 2차: Google Drive (원본 + 썸네일만, 영상은 YouTube가 원본)
  - 3차: gems.db 일일 덤프
- **복구 절차**: 어느 단계부터 복구 가능한지 명세

#### 작성 대상
`step9_g_backup.md` (신규)

---

### 8. 플레이리스트 기획 단계 (step0 이전)

#### 문제
현재 step1~5는 "플레이리스트 곡 목록이 주어졌다" 전제. 실제로는 **어떤 곡 20개를 뽑을지** 기획 필요.

#### 필요한 것
- **기획 단계**:
  1. **테마 결정** — 무드, 타겟, 계절감
  2. **프롬프트 다양화** — 20곡이 단조롭지 않도록 BPM/키/악기 변주
  3. **가사 작성** — 직접 작성 or LLM 생성 + 수정
  4. **트랙 순서** — librosa 에너지 분석 → 후킹 시작 곡 → 드라마틱 빌드업 → 해방 엔딩
  5. **제목/설명 초안** — 데이터 기반 VR 예측
- **트랙 순서 공식**:
  ```python
  def optimize_track_order(tracks: list) -> list:
      """
      1곡: 가장 후킹 강한 곡 (처음 30초 임팩트)
      2-5곡: 점진적 빌드업
      6-10곡: 피크 (에너지 최고)
      11-15곡: 변주 (다른 톤)
      16-19곡: 감정 클라이맥스
      20곡: 여운 있는 엔딩
      """
      # librosa 에너지 + BPM + 키 기반 정렬
      ...
  ```

#### 작성 대상
`step0a_playlist_planning.md` (신규, step1 이전)

---

## 🟢 낮음 (선택)

### 9. A/B 테스트 프로세스

#### 필요한 것
- 썸네일 2종 제작 → 한 쪽으로 24시간 테스트 → 성과 좋은 쪽으로 교체
- YouTube는 공식 A/B 테스트 기능 없음 → 수동 비교
- 제목도 동일하게 테스트
- DB에 실험 결과 기록 → 다음 썸네일 자동 최적화

### 10. 채널별 특화 가이드

#### 필요한 것
- **Hazy Roast (cafe)**:
  - 인스트루멘탈 전용 파이프라인 (Whisper STT 불필요)
  - 커버아트 스타일: 로파이 애니, 따뜻한 톤
  - 제목: "틀어놓으면 카페 사장님이 뭐냐고 물어봄"
- **ZERO MERCY BEATS (phonk)**:
  - 에너지 높은 곡 전용
  - 커버아트: 사이버펑크, 네온
  - 제목: "이게 왜 좋지??💀" IKXE 패턴
  - 60~90초 쇼츠 최적

### 11. 커뮤니티 상호작용

#### 필요한 것
- 댓글 응답 (커뮤니티 지침 위반 없는 선에서)
- 커뮤니티 탭 활용 (이미지 투표, 근황 등)
- 구독자 Q&A 영상
- 100 구독자 달성 시 감사 쇼츠

### 12. AI 감지 / 저작권

#### 필요한 것
- mippia.com 등 AI 감지 서비스 모니터링 (이미 연구 중)
- 후처리 공식 (이미 일부 메모리에 기록):
  - 스템분리+FLAC재인코딩+가벼운후처리 → 69% 감지율
  - 스템분리+보컬피치복원(+4/-4반음) → 52~53% (최저)
- Content ID 클레임 대응 절차
- 저작권 신고 예방

---

## 🔵 특별

### 13. feedback_*.md 메모리 → docs 통합

#### 문제
사용자 피드백이 메모리(`~/.claude/projects/.../memory/feedback_*.md`)에만 있고 docs에 일부만 반영됨.

#### 통합해야 할 피드백
| 메모리 | 관련 docs | 반영 상태 |
|--------|----------|:---:|
| `feedback_hooking_titles.md` | step4_thumbnail | ✅ 반영됨 |
| `feedback_destructive_actions.md` | 모든 단계 | ⚠️ 언급만 |
| `feedback_playlist_merge.md` | step3_video | ✅ 반영됨 |
| `feedback_save_all_data.md` | step1_suno | ✅ 반영됨 |
| `feedback_no_purple_ai.md` | step4_thumbnail | ✅ 반영됨 |
| `feedback_demucs_mps.md` | step2_audio | ✅ 반영됨 |
| `feedback_verify_before_render.md` | step3_video | ⚠️ 일부 |
| `feedback_suno_aligned_lyrics.md` | step2_audio | ⚠️ 일부 |
| `feedback_api_quota_preserve.md` | step5a_upload | ✅ 반영됨 |
| `feedback_browser_use_brand_channel.md` | step5a_upload | ✅ 반영됨 |
| `feedback_manual_headless.md` | step5a_upload | ✅ 반영됨 |
| `feedback_fix_root_cause.md` | (전반) | ❌ 미반영 |
| `feedback_independent_videos.md` | step5a_upload | ⚠️ 일부 |
| `feedback_use_team.md` | (전반) | ❌ 미반영 |
| `feedback_formal_korean.md` | (커뮤니케이션) | N/A |

#### 작성 대상
미반영/일부 반영 피드백을 해당 step 문서에 통합

---

### 14. docs 버전 관리 (CHANGELOG)

#### 필요한 것
- `CHANGELOG.md` 추가
- 각 문서 상단에 `version: 1.0.0` 및 `last_updated: 2026-04-07` 추가
- 주요 변경 사항 기록:
  ```markdown
  ## [1.0.0] - 2026-04-07
  ### Added
  - Initial release: step0~step5 + step9
  - Suno generation guide (step1)
  - Audio processing guide (step2)

  ## [Unreleased]
  - Analytics integration (step9_d)
  - A/B testing process (step9)
  ```

---

## 15. 착수 우선순위 (제안)

### Phase 1 — 다음 세션
1. **🔴 #1 에러 복구** — 체크포인트 파일 표준화 → 어떤 단계 실패해도 재시작 가능
2. **🔴 #3 상태 관리** — DB status 전이 규칙 → 파이프라인 자동화 기반

### Phase 2 — 10 플레이리스트 업로드 후
3. **🔴 #4 Analytics** — 성과 데이터 수집 → 피드백 루프
4. **🟡 #5 QA Gates** — 자동 검증

### Phase 3 — 100 구독자 달성 후
5. **🟡 #8 기획 단계** — librosa 트랙 순서 최적화
6. **🟢 #9 A/B 테스트** — 썸네일/제목 최적화
7. **🟢 #10 채널 확장** — Hazy Roast, ZERO MERCY BEATS

### 백그라운드 (언제든지)
- **🟡 #6 리소스 모니터링** — 매일 자동 체크
- **🟡 #7 백업 정책** — 외장 SSD 세팅
- **🔵 #13 메모리 통합** — docs 보완
- **🔵 #14 버전 관리** — CHANGELOG

---

## 16. 문서 작성 계획 (이 문서 이후)

미래 작업으로 만들어야 할 **새 문서 목록**:

| 파일 | 내용 | 우선순위 |
|------|------|:---:|
| `step0a_playlist_planning.md` | 기획 단계 (곡 선정, 트랙 순서) | 🟡 |
| `step9_a_error_recovery.md` | 체크포인트 + 재시도 전략 | 🔴 |
| `step9_b_logging.md` | 통합 로그 + 대시보드 | 🔴 |
| `step9_c_state_management.md` | DB status 전이 규칙 | 🔴 |
| `step9_d_analytics.md` | YouTube Analytics 연동 | 🔴 |
| `step9_e_qa_gates.md` | 단계별 자동 검증 | 🟡 |
| `step9_f_resource_monitoring.md` | 크레딧/쿼터 추적 | 🟡 |
| `step9_g_backup.md` | 백업 정책 | 🟡 |
| `step9_h_ab_testing.md` | A/B 테스트 | 🟢 |
| `step9_i_channel_specifics.md` | cafe/phonk 특화 | 🟢 |
| `step9_j_community.md` | 커뮤니티 상호작용 | 🟢 |
| `step9_k_copyright.md` | AI 감지/저작권 | 🟢 |
| `CHANGELOG.md` | 버전 관리 | 🔵 |

---

## 17. 주의사항

⚠️ **이 문서의 모든 항목은 아직 구현되지 않은 작업입니다.**
- 실제 파이프라인 실행에는 **step0~step5만 사용**
- 이 문서의 항목들은 각 항목 착수 전 **사용자 확인 필수**
- 현재 파이프라인(step0~step5)은 수동 개입이 일부 필요하지만 동작 가능

⚠️ **먼저 step0~step5로 10개 플레이리스트 정도 돌려보고**, 실제로 문제가 되는 부분부터 이 문서의 항목을 구현하는 것을 권장합니다.
