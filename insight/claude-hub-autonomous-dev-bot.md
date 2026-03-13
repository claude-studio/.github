# claude-hub — GitHub 자율 개발 봇 인사이트

<!-- 0. 메타 정보 블록 -->

| 항목 | 내용 |
|------|------|
| **저장소** | [claude-did-this/claude-hub](https://github.com/claude-did-this/claude-hub) |
| **라이선스** | MIT |
| **스택** | TypeScript, Express, Docker, Claude Code CLI |
| **추천 레벨** | 🔵 TRIAL |
| **분석일** | 2026-03-13 |

> 🔵 **TRIAL** — 개인/팀 사이드 프로젝트에서 실용적. 프로덕션 전 운영 부담 검토 필요

---

## 1. TL;DR

- **무엇**: GitHub 이슈/PR 코멘트에서 `@봇이름` 멘션을 받아 Claude Code CLI가 코드를 직접 작성·커밋·PR까지 만드는 자율 개발 봇 서버
- **어떻게**: Docker 컨테이너 안에서 `claude --print` 실행 → git clone → 코드 작성 → PR 생성까지 무인 수행
- **왜**: Claude Max 구독 세션을 재사용해 토큰 과금 없이 자율 AI 개발 워크플로우 구축 가능

---

## 2. 왜 주목해야 하는가

**기존 문제**: GitHub Actions + LLM API 조합은 토큰 비용이 크고, 컨텍스트가 단일 요청으로 제한되며, 코드 작성 → 커밋 → PR 생성까지 직접 연결하기 어렵다.

**이 프로젝트의 접근**: 웹훅 서버가 GitHub 이벤트를 수신 → Docker 컨테이너에서 Claude Code CLI를 전체 툴 권한으로 실행 → Claude가 저장소를 분석하고 작업을 완전 자율 수행.

**차별점**:
- Claude Max 구독(월정액) 세션을 그대로 재사용 → **추가 API 비용 없음**
- 컨테이너당 최대 2시간 실행 → 복잡한 멀티스텝 태스크 완료 가능
- 모듈형 웹훅 레지스트리로 GitHub 외 다른 provider 확장 가능

---

## 3. 아키텍처

```
[GitHub 이벤트 (이슈/PR 코멘트)]
    │ webhook POST
    ▼
[Express 서버 :3002]
    │ HMAC-SHA256 서명 검증
    │ Rate limit: 50req / 5min
    ▼
[WebhookProcessor]
    │
    ├── /api/webhooks/github     → [githubController] (레거시)
    └── /api/webhooks/:provider  → [WebhookRegistry] (모듈형)
                                       │
                         ┌─────────────┴──────────────┐
                         │                            │
              [GitHubWebhookProvider]     [ClaudeWebhookProvider]
                    [IssueHandler]           [SessionHandler]
                         │
              [claudeService.processCommand()]
                         │
              [docker run claudecode:latest]
                         │
              [git clone → claude --print → gh CLI]
```

| 컴포넌트 | 역할 |
|---------|------|
| Express 서버 | 웹훅 수신, 서명 검증, 라우팅 |
| WebhookRegistry | 플러그인 방식 provider 등록, 이벤트 매칭 |
| claudeService | Docker 실행 명령 구성, 프롬프트 조립, 응답 수집 |
| claudecode 컨테이너 | Claude Code CLI 격리 실행 환경 |
| SessionHandler | 세션 의존성 체인 (`분석 → 구현 → 테스트`) |

---

## 4. 동작 흐름

**시나리오**: 이슈 코멘트로 기능 구현 요청

```
1. 사용자: "@MyBot OAuth 로그인 구현해줘"
   (GitHub 이슈 #42 코멘트)

2. GitHub → POST /api/webhooks/github
   헤더: x-github-event: issue_comment
         x-hub-signature-256: sha256=...

3. Express 서버: HMAC-SHA256 서명 검증 (timing-safe 비교)

4. githubController:
   - comment.body에 '@MyBot' 포함 여부 확인
   - sender가 AUTHORIZED_USERS 목록에 있는지 확인
   - "@MyBot " 이후 텍스트 추출: "OAuth 로그인 구현해줘"

5. docker run --rm \
     --memory 2g --cpu-shares 512 --pids-limit 200 \
     -e REPO_FULL_NAME=org/repo \
     -e ISSUE_NUMBER=42 \
     -e COMMAND="<전체 프롬프트>" \
     -v ~/.claude-hub:/home/node/.claude \
     claudecode:latest

6. 컨테이너 내부 (claudecode-entrypoint.sh):
   a. gh auth login --with-token <<< $GITHUB_TOKEN
   b. git clone → git checkout -b feat/oauth-login
   c. claude --allowedTools "Read,Write,Edit,Bash,..." --print "<프롬프트>"
   d. git add/commit/push → gh pr create

7. 컨테이너 종료 → stdout 수집 → 이슈 #42에 최종 코멘트
```

---

## 5. 핵심 기능 분석

| 기능 | 레벨 | 메모 |
|------|------|------|
| @멘션 트리거 자율 구현 | 🟢 ADOPT | 검증된 핵심 기능, 즉시 실용 가능 |
| Claude Max 세션 재사용 | 🟢 ADOPT | 비용 절감 효과 명확, 구독자에게 바로 적용 가능 |
| 신규 이슈 자동 라벨링 | 🟢 ADOPT | 읽기 전용 툴만 사용, 안전하고 실용적 |
| CI 통과 후 자동 PR 리뷰 | 🔵 TRIAL | `PR_REVIEW_WAIT_FOR_ALL_CHECKS=true`로 활성화, 실사용 검토 필요 |
| SessionHandler 의존성 체인 | 🔵 TRIAL | `분석→구현→테스트` 파이프라인, 아직 실험적 |
| AWS Bedrock 인증 | 🟡 ASSESS | 엔터프라이즈 환경 한정, 일반 사용 불필요 |
| WebhookRegistry 커스텀 provider | 🟡 ASSESS | 확장성 좋으나 문서 미흡 |

---

## 6. 보안 & 제약사항

**보안 모델**:

| 레이어 | 방법 |
|--------|------|
| 웹훅 인증 | HMAC-SHA256 서명 검증 (timing-safe 비교) |
| 유저 권한 | `AUTHORIZED_USERS` 화이트리스트 |
| 코드 인젝션 방지 | Docker 명령어 배열 구성 (shell 미경유) |
| 컨테이너 격리 | 메모리 2GB / CPU shares 512 / PID 200 제한 |
| 무한루프 방지 | 응답에서 봇 @멘션 자동 제거 |
| 로그 보안 | 민감 env var 자동 redact |

**알려진 한계**:

- 봇 전용 GitHub 계정을 직접 생성해야 함 (GitHub App 미지원, 향후 계획)
- 컨테이너 최대 실행시간 2시간 (`CONTAINER_LIFETIME_MS`) — 초과 작업 불가
- 동시 컨테이너 수 제한 없음 → 트래픽 급증 시 호스트 리소스 주의
- Claude Max 세션 방식은 단일 사용자 인증 전제 → 팀 공용 서버에서 세션 공유 불가

---

## 7. 빠른 시작

**전제 조건**: Docker, Claude Max 구독(5x 이상), GitHub 봇 계정 및 PAT

```bash
# 1. 클론 및 설정
git clone https://github.com/claude-did-this/claude-hub.git
cd claude-hub
cp .env.quickstart .env

# 2. .env 필수 값 입력
# BOT_USERNAME=@YourBotName
# GITHUB_TOKEN=ghp_...          (봇 계정 PAT)
# GITHUB_WEBHOOK_SECRET=$(openssl rand -hex 32)
# AUTHORIZED_USERS=your-github-username

# 3. Claude Max 세션 캡처 (브라우저 인증 1회)
./scripts/setup/setup-claude-interactive.sh

# 4. 서비스 시작
docker compose up -d

# 5. 외부 노출 (Cloudflare Tunnel 사용 시)
cloudflared tunnel --url http://localhost:3002
```

**GitHub 웹훅 등록**: Repository → Settings → Webhooks → Add webhook
- Payload URL: `https://<터널URL>/api/webhooks/github`
- Content type: `application/json`
- Secret: `GITHUB_WEBHOOK_SECRET` 값
- Events: Issues, Issue comments, Pull requests, Check suites

**검증**:

```bash
# 헬스체크
curl http://localhost:3002/health

# 이슈 코멘트에서 테스트
# "@YourBotName hello, list the top-level files in this repo"
```

---

## 8. 도입 판단

**이럴 때 써라**:

- 반복적인 GitHub 이슈를 AI가 직접 처리하게 하고 싶을 때
- Claude Max 구독이 있고 API 추가 비용 없이 자동화하고 싶을 때
- 소규모 팀 / 개인 프로젝트에서 PR 리뷰·라벨링 자동화가 필요할 때
- 홈서버나 소형 VPS에서 self-hosted 봇을 운영하려는 경우

**이럴 때 쓰지 마라**:

- 공개 저장소에 `AUTHORIZED_USERS` 검증 없이 노출하는 경우 (보안 위험)
- Claude Max 구독 없이 API 비용 최소화가 최우선인 경우
- 컨테이너 오케스트레이션(Kubernetes 등) 없이 고트래픽 환경에서 운영하는 경우
- 엄격한 감사 로그·컴플라이언스가 필요한 엔터프라이즈 환경 (GitHub App 아님)

**대안**:

| 대안 | 비교 |
|------|------|
| GitHub Copilot Workspace | 관리형 SaaS, 설정 불필요. 자유도 낮음 |
| Devin | 전문 AI 개발 에이전트. 고가, 클라우드 의존 |
| GitHub Actions + OpenAI | 범용 구성. Claude Code CLI 수준의 자율성 없음 |

---

## 9. 구현 아이디어 & 참고 링크

**구현 아이디어**:

- **Slack 트리거 봇**: 동일한 웹훅 아키텍처에 Slack Event API provider를 추가해 `@bot 기능 구현해줘` 슬랙 멘션으로 동일한 자율 개발 루프 실행
- **사내 코드 리뷰 자동화 서비스**: PR 생성 시 자동으로 보안/성능/스타일 관점의 멀티 리뷰어 봇 실행. SessionHandler 체인으로 `리뷰→수정→재리뷰` 파이프라인 구성
- **이슈 트리아지 봇**: 신규 이슈 자동 라벨링 + 담당자 배정 + 유사 이슈 연결 코멘트까지 자동화. 읽기 전용 툴만 사용하므로 안전하게 적용 가능
- **GitHub Actions 대체 CI/CD 파이프라인**: 테스트 실패 이벤트(`check_suite.completed`) 수신 → Claude가 실패 원인 분석 + 수정 PR 자동 생성
- **문서 자동 동기화 봇**: 코드 변경 PR 머지 시 관련 README/API 문서 자동 업데이트 PR 생성

**참고 링크**:

- [공식 저장소](https://github.com/claude-did-this/claude-hub)
- [Quick Start Guide](https://github.com/claude-did-this/claude-hub/blob/main/QUICKSTART.md)
- [인증 가이드](https://github.com/claude-did-this/claude-hub/blob/main/docs/claude-authentication-guide.md)
- [Claude Code CLI](https://github.com/anthropics/claude-code)
- [GitHub Webhooks 문서](https://docs.github.com/en/webhooks)
