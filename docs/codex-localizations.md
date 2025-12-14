# Codex 사내 커스터마이징 가이드 (한글화/브랜딩/패치 유지)

이 문서는 upstream Codex(openai/codex)를 **사내 배포용으로 로컬라이즈(한글화) / 브랜딩**하고, 신규 릴리즈가 나올 때 **패치를 반복 적용**하기 위한 유지보수 방법을 정리합니다.

> 이 문서의 내용은 사내 배포 목적이며 upstream에 PR/푸시하지 않는 전제를 둡니다.

---

## 0) TUI가 어디를 타는지(중요)

Codex는 빌드/설정에 따라 `tui2`(신규) 또는 `tui`(legacy)를 사용합니다.

- 선택 로직: `codex-rs/cli/src/main.rs`
  - `tui2`가 enabled면 `codex-rs/tui2` 경로의 문자열이 실제로 노출됩니다.
  - fallback 시 `codex-rs/tui` 경로의 문자열이 노출됩니다.

사내 배포에서 “항상” 한글이 보이게 하려면 **`tui2`와 `tui` 둘 다 패치**하는 것을 권장합니다.

---

## 1) 한글화/브랜딩 변경 지점(소스 코드)

### 1.1 처음 설치/첫 실행: 로그인 TUI

- Welcome 화면(상단 환영 문구)
  - `tui2`: `codex-rs/tui2/src/onboarding/welcome.rs`
  - `tui`: `codex-rs/tui/src/onboarding/welcome.rs`
- 로그인 선택/진행/성공/에러 문구(“Sign in with ChatGPT…”, “Finish signing in…” 등)
  - `tui2`: `codex-rs/tui2/src/onboarding/auth.rs`
  - `tui`: `codex-rs/tui/src/onboarding/auth.rs`

### 1.2 처음 설치/첫 실행: 프로젝트 신뢰 여부(Trust directory) TUI

- 신뢰 프롬프트/설명/선택지(“Allow…”, “Require approval…”, “Press Enter…” 등)
  - `tui2`: `codex-rs/tui2/src/onboarding/trust_directory.rs`
  - `tui`: `codex-rs/tui/src/onboarding/trust_directory.rs`

### 1.3 “OpenAI Codex” 표시를 “현대퓨처넷 HCodex([버전])”로 변경

Codex는 “버전”을 서로 다른 지점에서 렌더링합니다.

- 세션 헤더 박스(대화 상단)
  - `tui2`: `codex-rs/tui2/src/history_cell.rs`
  - `tui`: `codex-rs/tui/src/history_cell.rs`
  - 참고: 버전 값은 `self.version`을 사용합니다.
- 상태 카드(세션 정보/사용량 카드)
  - `tui2`: `codex-rs/tui2/src/status/card.rs`
  - `tui`: `codex-rs/tui/src/status/card.rs`
  - 참고: 버전 값은 `CODEX_CLI_VERSION`을 사용합니다.
- (TUI 밖) 사람 출력용 배너(로그/콘솔)
  - `codex-rs/exec/src/event_processor_with_human_output.rs`

### 1.4 “Visit https://…” 문구 변경(상태 카드 하단 안내)

상태 카드의 “Visit … for up-to-date …” 안내 문구는 아래에서 변경합니다.

- `tui2`: `codex-rs/tui2/src/status/card.rs`
- `tui`: `codex-rs/tui/src/status/card.rs`

> 비슷한 “Visit …” 문구가 에러 메시지(예: usage limit)에도 존재하므로, 상태 카드 외의 문구까지 바꾸려면 `rg "Visit https://"`로 전체 검색 후 필요한 곳을 추가로 패치하세요(예: `codex-rs/core/src/error.rs`).

---

## 2) Gateway 로그인(issuer/client_id) 오버라이드 패치

Gateway 연동 시(issuer를 사내 Gateway로 변경) Codex의 브라우저 로그인 경로에서 `ServerOptions`를 override 해야 합니다. 상세한 Gateway API/동작은 `docs/gateway-server.md`를 참고하세요.

### 2.1 변경 지점

- `codex login`(CLI) 브라우저 로그인
  - `codex-rs/cli/src/login.rs`
  - `login_with_chatgpt()`에서 `ServerOptions::new(...)` 생성 후 `opts.issuer` / `opts.client_id`를 Gateway 값으로 설정하는 방식으로 패치합니다.
- 첫 실행 Onboarding(TUI)에서 ChatGPT 로그인
  - `tui2`: `codex-rs/tui2/src/onboarding/auth.rs`
  - `tui`: `codex-rs/tui/src/onboarding/auth.rs`
  - `start_chatgpt_login()`에서 `ServerOptions::new(...)` 생성 후 동일하게 override 합니다.
- 참고(기본값/URL 구성)
  - `codex-rs/login/src/server.rs` (기본 issuer/authorize URL/token URL 구성)

### 2.2 사내 배포에서 흔히 쓰는 override 값 소스(예시)

사내 배포는 “upstream config 스키마 변경”을 최소화하는 편이 안전합니다. 보통 아래 중 하나로 값을 주입합니다.

- (권장) 사내 배포 전용 config 키를 추가해 `Config`에서 읽기
- (간단) env var로 주입(내부 배포 스크립트/런처에서 설정)
- (CLI 친화) `codex login`에 `--issuer-base-url`, `--client-id` 같은 옵션 추가

어떤 방식을 택하든 목표는 동일합니다: `run_login_server(opts)` 호출 전에 `opts.issuer` / `opts.client_id`가 Gateway 값으로 설정되어야 합니다.

---

## 3) 신규 릴리즈 대응(패치 유지) 방법

### 3.1 패치 브랜치(커밋)로 유지하기

한 번의 대형 커밋보다 **기능별 커밋(예: 브랜딩 / 한글화 / Gateway issuer)** 으로 쪼개두면 충돌 해결이 쉬워집니다.

예시 워크플로우(업스트림 태그 기반):

```bash
# 1) upstream remote 추가(또는 사내 미러)
git remote add upstream https://github.com/openai/codex.git
git fetch upstream --tags

# 2) 사내 패치 브랜치로 이동
git checkout internal/hcodex

# 3) 신규 릴리즈 태그로 rebase
git rebase upstream/vX.Y.Z
```

이 방식의 장점은 “수정 지점 다시 찾기”가 아니라 **충돌난 곳만 고치면 끝**이라는 점입니다.

### 3.2 반복 충돌 자동화: `git rerere`

매 릴리즈마다 비슷한 충돌이 반복된다면 `rerere`가 시간을 크게 줄여줍니다.

```bash
git config rerere.enabled true
git config rerere.autoupdate true
```

- 한 번 충돌을 해결하면, 다음 rebase/merge에서 유사 충돌을 자동으로 적용합니다.

### 3.3 패치 파일로 보관하기(선택)

배포 파이프라인에서 “소스 트리 + patch” 형태로 관리하고 싶다면:

```bash
git format-patch -o internal/patches upstream/vX.Y.(Z-1)..internal/hcodex
git am -3 internal/patches/*.patch
```

---

## 4) 내부 CI/검증 자동화(추천)

### 4.1 원문 문자열 잔존 체크(브랜딩/한글화 누락 방지)

사내 CI(또는 배포 스크립트)에 아래 같은 검사를 넣어두면, 신규 릴리즈에서 문자열이 원복되었을 때 빠르게 감지할 수 있습니다.

```bash
rg -n "OpenAI Codex|Welcome to|Sign in with ChatGPT|Visit https://chatgpt.com/codex/settings/usage" codex-rs/tui2 codex-rs/tui codex-rs/exec
```

- 매칭 결과가 있으면 실패(exit 1)하도록 처리합니다.

### 4.2 스냅샷 테스트(문구 변경 시)

`codex-rs/tui`/`codex-rs/tui2`는 `insta` 스냅샷 테스트를 사용합니다. UI 문구를 바꾸면 스냅샷 업데이트가 필요할 수 있습니다.

- 예시(`tui2`):
  - `cargo test -p codex-tui2`
  - `cargo insta pending-snapshots -p codex-tui2`
  - 필요 시 `cargo insta accept -p codex-tui2`

