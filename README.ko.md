[English](README.md) | 한국어

# insane-search

> **포기를 모르는 미친 스크레이퍼.**

`403`. WAF. CAPTCHA. 빈 SPA. 로그인 벽. 일반 도구가 다 포기할 때, insane-search는 그제서야 시작합니다. 5단계 프로브. TLS 임퍼소네이션 자동 설치. 진짜 브라우저로 숨은 API 발견. "차단됐다"는 사이트도 결국 길이 나옵니다.

API 키 없음. 가입 없음. 설정 없음. 설치하고, Claude Code가 더 이상 포기하지 않는 걸 보세요.

[빠른 시작](#빠른-시작) • [동작 방식](#동작-방식) • [인덱스](#인덱스에-있는-것) • [레퍼런스](#레퍼런스) • [요구사항](#요구사항)

---

## 빠른 시작

### 1. 마켓플레이스 등록

```
/plugin marketplace add https://github.com/fivetaku/gptaku_plugins.git
```

### 2. 플러그인 설치

```
/plugin install insane-search
```

### 3. Claude Code 재시작

끝. 설정도, API 키도, 환경변수도 없습니다.

### 4. 평소처럼 말하기

차단된 사이트는 자동으로 뚫립니다.

```
"r/LocalLLaMA에서 뜨는 거 뭐야?"
"openclaw가 X에 최근에 뭐라고 했어?"
"이 유튜브 영상 요약해줘"
"쿠팡에서 10만원 미만 키보드 찾아줘"
"이 네이버 블로그 글 읽어줘"
```

---

## 왜 insane-search?

- **"차단"이라는 단어를 모름** — "이 사이트는 접근 불가" 같은 사전 판정이 없습니다. 모든 사이트가 전체 체인을 탑니다. 쿠팡? 뚫립니다. LinkedIn? JSON-LD 추출됩니다. 요즘IT? Chrome UA로 끝
- **자기 무기를 스스로 설치** — TLS 핑거프린트 우회용 `curl_cffi` 없음? 설치합니다. `feedparser` 없음? 설치합니다. `yt-dlp` 없음? 설치합니다. 눈치챌 새도 없이
- **5단계 프로브, 단 하나 아님** — WebFetch → Jina → curl UA/URL 변형 → TLS 임퍼소네이션 (safari/chrome/firefox) → 실제 브라우저. 이전 단계가 벽에 부딪혀야만 다음으로 에스컬레이션
- **숨은 API까지 발견** — Phase 3은 페이지만 렌더링하는 게 아닙니다. 브라우저 네트워크를 감시해서 사이트가 내부적으로 쓰는 실제 JSON API를 잡아내고, 재활용할 수 있게 돌려줍니다
- **설정 부담 제로** — API 키도, OAuth도, 개발자 포털도 없음. 전부 공개 엔드포인트와 자동 설치 라이브러리로 작동

---

## 동작 방식

Claude Code가 URL 접근이 필요할 때, insane-search는 4단계 적응형 스케줄러를 실행합니다. 각 Phase는 이전 Phase가 실패하거나 특정 차단 신호를 감지할 때만 실행됩니다.

```
Phase 0: 특수 엔드포인트 인덱스
  ↓ 인덱스에 없거나 실패
Phase 1: 경량 프로브 (병렬)
  • WebFetch + Jina Reader
  • curl Chrome / 모바일 / Googlebot UA
  • URL 변형: m.{domain}, .json, /rss, /feed
  • 사이드카: AMP 캐시, archive.today, Wayback (low-trust)
  ↓ 403/429/WAF 헤더/챌린지 본문 감지
Phase 2: TLS 임퍼소네이션
  • curl_cffi safari → chrome → firefox
  • 미설치면 자동 설치: pip install curl_cffi
  ↓ TLS 우회 실패 또는 JS 챌린지 감지
Phase 3: 실제 브라우저
  • Playwright MCP (browser_navigate → snapshot → evaluate)
  • 숨은 API 발견도 함 (network_requests)
  ↓ login/paywall 감지
종료: "인증 필요" — 더 이상 Phase 진행해도 소용없음
```

**핵심 원칙**: 어떤 방법도 미리 제외하지 않는다. 의존성이 없다고 건너뛰지 말고 설치하고 시도한다. "이 사이트는 어렵다"고 건너뛰지 말고 — 사이트도 변하고 방법도 지금은 먹힐 수 있다.

모든 HTML 응답에서 OGP 태그와 JSON-LD 구조화 데이터도 같이 추출합니다. 본문 전체를 못 가져와도 제목, 요약, 가격, 프로필은 확보됩니다.

---

## 인덱스에 있는 것

범용 체인이 자동으로 발견할 수 없는 특수 엔드포인트만 인덱스에 둡니다. 나머지 — 네이버 블로그, 쿠팡, LinkedIn, Medium, 한국 뉴스 사이트, Substack, 대부분의 커뮤니티 — 는 인덱스에 없어도 적응형 스케줄러가 알아서 처리합니다.

### 플랫폼 전용 API

| 플랫폼 | 방법 | 레퍼런스 |
|--------|------|----------|
| X/Twitter | `syndication.twitter.com/srv/timeline-profile/...` + oEmbed | `twitter.md` |
| Reddit | URL + `.json` + Mobile UA | `json-api.md` |
| Bluesky | AT Protocol (`public.api.bsky.app/xrpc/...`) | `public-api.md` |
| Mastodon | 인스턴스별 공개 API | `public-api.md` |
| Hacker News | Firebase API | `json-api.md` |
| Stack Overflow | SE API v2.3 | `public-api.md` |
| Lobste.rs / V2EX / dev.to | 공개 JSON API | `json-api.md` |

### 미디어 (CLI 도구 필수)

| 플랫폼 | 방법 | 레퍼런스 |
|--------|------|----------|
| YouTube / Vimeo / Twitch / TikTok / SoundCloud + 1,853개 | `yt-dlp --dump-json` | `media.md` |

### 학술 & 레지스트리

| 플랫폼 | 방법 | 레퍼런스 |
|--------|------|----------|
| arXiv | Atom API | `public-api.md` |
| CrossRef | REST API | `public-api.md` |
| Wikipedia | REST API | `json-api.md` |
| OpenLibrary | JSON API | `public-api.md` |
| GitHub | `gh` CLI / REST API | `public-api.md` |
| npm / PyPI | Registry API | `json-api.md` |
| Wayback Machine | CDX API | `public-api.md` |

### 한국 전용

| 플랫폼 | 방법 | 레퍼런스 |
|--------|------|----------|
| 네이버 금융 시세 | `api.finance.naver.com/siseJson.naver` (비공식, 무인증) | `naver.md` |

**그 외 전부 Phase 1~3이 자동 처리** — 쿠팡 (curl_cffi safari), LinkedIn (JSON-LD 추출), Medium (Jina), 대부분의 한국 커뮤니티 (Jina 또는 curl), `/rss`·`/feed` 엔드포인트가 있는 모든 사이트.

---

## 레퍼런스

스킬은 기법별 레퍼런스 파일로 구성됩니다.

| 파일 | 내용 |
|------|------|
| `fallback.md` | Phase 0→3 적응형 스케줄러, 에스컬레이션 신호, 응답 검증 |
| `jina.md` | Jina Reader (`r.jina.ai`, 무인증) |
| `json-api.md` | 공개 JSON API (Reddit, HN, dev.to, Wikipedia, npm, PyPI 등) |
| `public-api.md` | Bluesky, Mastodon, Stack Exchange, arXiv, CrossRef, OpenLibrary, GitHub, Wayback |
| `media.md` | 1,858개 미디어 사이트용 yt-dlp |
| `twitter.md` | Twitter Syndication API + oEmbed |
| `naver.md` | 네이버 블로그 모바일 URL, 네이버 금융 JSON API |
| `rss.md` | 한국 언론 RSS 9개, Google News RSS, feedparser, SearXNG |
| `tls-impersonate.md` | curl_cffi 다중 타겟 (safari/chrome/firefox) + 자동 설치 |
| `playwright.md` | Playwright MCP 풀 도구 (snapshot, evaluate, network_requests) |
| `cache-archive.md` | Google AMP 캐시, archive.today, Wayback Machine |
| `metadata.md` | OGP, JSON-LD, Schema.org, Next.js RSC 페이로드 추출 |

---

## 요구사항

**필수:** Claude Code만 있으면 됩니다.

**자동 설치됨** (첫 사용 시 스킬이 알아서 설치):

```bash
pip install curl_cffi    # WAF 차단 사이트용 TLS 임퍼소네이션
pip install feedparser   # RSS/Atom 파싱
pip install yt-dlp       # 1,858개 미디어 사이트
```

**선택적, 커버리지 향상:**

```bash
brew install gh                      # GitHub (REST API보다 빠름)
claude mcp add playwright -- npx @playwright/mcp@latest   # JS 렌더링 사이트
```

의존성이 없다고 방법을 건너뛰지 않습니다 — 설치하고 시도합니다.

---

## insane-search는 이런 게 아닙니다

- **스크레이퍼가 아님** — 방법 선택 레이어. 공개 API와 표준 기법을 조합할 뿐
- **API 키 기반 아님** — 전부 무인증 공개 엔드포인트 또는 URL 변형
- **수작업 정답지 아님** — 인덱스는 최소(~15 그룹). 나머지는 적응형 스케줄러가 발견
- **바이어스 형성 안 함** — "접근 거부" 목록 없음. 뚫릴 수 있는 사이트라면 체인이 길을 찾음

---

## 사용 예시

명령어 없습니다. 평소처럼 말하면 됩니다. URL이 차단되거나 특수 처리가 필요한 플랫폼 접근 시 자동 트리거.

```
"지금 해커뉴스 프론트페이지 뭐 있어?"
→ Firebase API로 탑 스토리 + 점수 + 댓글

"이번 주 arXiv AI 논문 찾아줘"
→ arXiv Atom API, 날짜 필터

"쿠팡에서 100만원 이하 노트북 뽑아줘"
→ Phase 2: curl_cffi safari → JSON-LD ItemList

"이 Medium 기사 요약해줘"
→ Phase 1: Jina Reader → 클린 마크다운

"Claude Code 레딧 반응 확인해줘"
→ Reddit JSON API + Mobile UA → 포스트 + 댓글
```

---

## 라이선스

MIT

---

<div align="center">

**웹에 있으면, insane-search는 들어갑니다.**

</div>
