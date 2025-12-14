# Gateway Server 연동 가이드 (사내 Intranet)

이 문서는 **Codex의 OAuth issuer 주소를 Gateway로 변경**하여, `codex login` 실행 시 **브라우저에서 Gateway 로그인 페이지가 열리고** 로그인 성공 후 Codex가 토큰을 저장해 이후 요청을 Gateway로 보내도록 연동하는 가이드입니다.  
현재 Codex는 **브라우저 로그인 issuer가 `https://auth.openai.com`으로 고정**돼 있어, 이 방식은 **Codex 소스 수정이 필요**합니다.

Codex 쪽 패치(issuer/client_id override, 사내 브랜딩/한글화 등)는 `docs/codex-localizations.md`에 분리해 정리합니다.

전제:

- Gateway가 **OAuth2 Authorization Code + PKCE issuer**로 동작한다.
- Codex 브라우저 로그인에서 사용할 **issuer / client_id를 Gateway로 변경**한다(소스 수정 필요).
- 모델 호출은 Gateway가 OpenAI 호환(Responses API)으로 프록시한다.

---

## 1) 로그인 연동 (Issuer = Gateway)

### 1.1 로그인 흐름(요약)

1. 사용자가 `codex login` 실행
2. Codex가 로컬 콜백 서버를 띄움: `http://localhost:1455/auth/callback`
3. 브라우저가 아래 URL로 열림: `<issuer>/oauth/authorize?...`
4. Gateway에서 로그인/회원가입 완료 후 `redirect_uri`로 `302` 리다이렉트:
   - `Location: http://localhost:1455/auth/callback?code=<auth_code>&state=<state>`
5. Codex가 `POST <issuer>/oauth/token`으로 authorization code를 교환(Authorization Code + PKCE)
6. Codex가 같은 `/oauth/token`으로 token-exchange 호출을 한 번 더 수행하여 **Codex 전용 API Key**(Gateway 토큰)를 발급받음
7. Codex가 발급받은 토큰을 `auth.json`에 저장
8. 이후 Codex → Gateway 요청에는 `Authorization: Bearer <codex_api_key>`가 포함됨

### 1.2 Codex 소스 수정(필수)

Codex는 기본 OAuth issuer를 `https://auth.openai.com`으로 사용하므로, Gateway 로그인 페이지를 띄우려면 **브라우저 로그인 경로에서 issuer/client_id를 Gateway 값으로 override**하도록 Codex를 패치해야 합니다.

- Codex 변경 위치/방법: `docs/codex-localizations.md`의 “Gateway 로그인(issuer/client_id) 오버라이드 패치” 참고

### 1.3 Codex 설정 예시

`$CODEX_HOME/config.toml` 예시입니다.

```toml
# (권장) 로그인 방식을 Gateway issuer 기반으로 고정
forced_login_method = "chatgpt"

# 사용량 UI는 Gateway에서 제공
chatgpt_base_url = "https://gateway.company.local"

# 모델 호출은 Gateway로
model_provider = "company-gateway"
model = "gpt-5.1"

[model_providers.company-gateway]
name = "Company Gateway"
base_url = "https://gateway.company.local/v1"
wire_api = "responses"
requires_openai_auth = true
```

### 1.4 Gateway OAuth(issuer) API 명세

issuer base URL을 `https://gateway.company.local`이라고 가정합니다.

#### 1.4.1 `GET /oauth/authorize`

Codex가 브라우저로 여는 URL입니다.

Request (Query Parameters)

필수:

- `response_type=code`
- `client_id=<string>`
- `redirect_uri=http://localhost:<port>/auth/callback`
- `scope=openid profile email offline_access` (문자열; Gateway는 파싱만 하고 무시해도 됨)
- `code_challenge=<base64url>` (PKCE S256)
- `code_challenge_method=S256`
- `state=<random>`

옵션(존재할 수 있음; Gateway는 무시 가능):

- `originator=<string>` (예: `codex_cli_rs`)
- `id_token_add_organizations=true`
- `codex_cli_simplified_flow=true`
- `allowed_workspace_id=<uuid>`

Behavior

1. 사용자가 **미로그인 상태**면 로그인/회원가입 UI를 표시하고, 성공 시 원래 authorize 요청으로 복귀합니다.
2. 사용자 로그인 완료 후, (선택) 권한동의 화면을 거친 다음 authorization code를 생성합니다.
3. 브라우저를 아래로 302 리다이렉트합니다:

`Location: <redirect_uri>?code=<auth_code>&state=<state>`

Response

- 성공: `302 Found` + `Location` 헤더
- 실패(권장): `400 Bad Request` + HTML/텍스트 에러 페이지
  - Codex는 OAuth 에러 query 파라미터(`error`, `error_description`)를 처리하지 않으므로, 실패 시 리다이렉트보다 에러 페이지가 디버깅에 유리합니다.

Authorization Code 저장 규칙(권장)

- code는 난수(opaque)로 발급(예: 32~64 bytes)
- DB에는 code를 **해시**로 저장(원문 저장 금지)
- TTL: 5분
- 1회 사용 후 즉시 폐기
- 아래 값을 함께 저장:
  - `client_id`
  - `redirect_uri`
  - `code_challenge`, `code_challenge_method`
  - `user_id`

---

#### 1.4.2 `POST /oauth/token`

Codex가 authorization code 교환 및 token-exchange에 사용합니다.

> 주의: Codex는 grant에 따라 Content-Type을 다르게 보낼 수 있으므로, 최소한 `application/x-www-form-urlencoded`는 반드시 지원해야 합니다.

(A) Authorization Code 교환

- Content-Type: `application/x-www-form-urlencoded`

Request form fields:

- `grant_type=authorization_code`
- `code=<auth_code>`
- `redirect_uri=<redirect_uri>` (authorize 요청과 동일해야 함)
- `client_id=<client_id>`
- `code_verifier=<pkce_verifier>`

Validation:

- code 존재/미사용/미만료
- `client_id`, `redirect_uri` 매칭
- PKCE S256 검증: `BASE64URL(SHA256(code_verifier)) == code_challenge`

Response: `200 application/json`

```json
{
  "id_token": "<jwt>",
  "access_token": "<opaque_or_jwt>",
  "refresh_token": "<opaque>",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

Codex는 최소 `id_token`, `access_token`, `refresh_token` 키를 기대합니다.

(B) Token Exchange (Codex용 API Key 발급)

Codex는 로그인 후 추가로 token-exchange를 호출해 “OpenAI API key처럼 쓸 토큰”을 얻습니다. Gateway에서 이 토큰을 **Codex 전용 API Key(=Gateway 토큰)** 로 발급하세요.

- Content-Type: `application/x-www-form-urlencoded`

Request form fields (Codex 고정 값):

- `grant_type=urn:ietf:params:oauth:grant-type:token-exchange`
- `client_id=<client_id>`
- `requested_token=openai-api-key`
- `subject_token=<id_token>`
- `subject_token_type=urn:ietf:params:oauth:token-type:id_token`

Response: `200 application/json`

```json
{
  "access_token": "<codex_api_key>",
  "token_type": "Bearer"
}
```

Codex는 이 `access_token`을 저장하고, 이후 **모델 호출/usage 호출**에 `Authorization: Bearer <codex_api_key>`로 전송합니다.

권장:

- `<codex_api_key>`는 **opaque 랜덤 토큰**으로 발급(예: prefix `cgk_` + 32~64 bytes)
- DB에는 토큰 원문을 저장하지 말고 **해시만 저장**
- 토큰에 `user_id`, `created_at`, `revoked_at`, `last_used_at` 메타데이터를 관리

(C) Refresh Token (선택 구현)

요청/응답 형태는 구현 자유도가 있으나, 아래 JSON 형태를 지원하면 Codex 호환에 유리합니다.

Request JSON:

```json
{
  "client_id": "<client_id>",
  "grant_type": "refresh_token",
  "refresh_token": "<refresh_token>",
  "scope": "openid profile email"
}
```

Response JSON:

```json
{
  "id_token": "<jwt>",
  "access_token": "<opaque_or_jwt>",
  "refresh_token": "<opaque>"
}
```

---

### 1.5 토큰(JWT) 클레임 권장 스펙

Gateway는 자체 검증/감사/추적을 위해 서명된 JWT를 권장합니다.

#### 1.5.1 `id_token` (JWT)

필수(권장):

- `iss`: Gateway issuer (예: `https://gateway.company.local`)
- `aud`: `client_id`
- `sub`: Gateway user id
- `iat`, `exp`

호환/편의(권장):

- `email`: 사용자 이메일
- `chatgpt_account_id`: 사용자 식별자(문자열)
- `https://api.openai.com/auth`: 객체로 아래를 넣으면 plan type 표시에 활용 가능
  - `chatgpt_plan_type`: `"free" | "plus" | "pro" | "team" | "business" | "enterprise" | "edu"`

#### 1.5.2 `<codex_api_key>` (token-exchange `access_token`)

실제로는 “Codex가 매 요청에 보내는 API 키”입니다.

- 만료 정책: (권장) 장기 토큰 + 서버측 revoke 지원
- 로테이션: 사용자 UI에서 재발급/폐기 기능 제공 권장

---

### 1.6 구현 완료 기준(체크리스트)

- `codex login` 실행 시 브라우저가 `https://gateway.company.local/oauth/authorize?...`로 열림
- Gateway 로그인 성공 후 `http://localhost:1455/auth/callback?code=...&state=...`로 리다이렉트됨
- Gateway가 `/oauth/token`에 대해 (A) authorization_code 교환 + (B) token-exchange를 정상 처리함
- 이후 Codex 요청에 `Authorization: Bearer <codex_api_key>`가 포함되어 Gateway가 사용자 식별이 가능함

---

## 2) Codex ↔ Gateway Request/Response 명세

### 2.1 모델 호출: `POST /v1/responses` (SSE 스트리밍)

#### 요청

- Method: `POST`
- Path: `<model_provider.base_url>/responses`
  - 예: `base_url="https://gateway.company.local/v1"`이면 `POST /v1/responses`
- Content-Type: `application/json`

주요 요청 헤더:

- `Authorization: Bearer <codex_api_key>`
- `originator: codex_cli_rs` (기본값; 내부 override 가능)
- `User-Agent: codex_cli_rs/<version> (...) ...`
- `conversation_id: <uuid>` (Codex가 세션 단위로 부여)
- `session_id: <uuid>` (conversation_id와 동일 값으로 설정됨)
- `x-openai-subagent: <string>` (서브 에이전트 문맥일 때만)
- (옵션) W3C trace 헤더(`traceparent` 등)가 추가될 수 있음

요청 바디(요약)

Codex는 OpenAI **Responses API** 호환 JSON 스키마를 사용합니다.

```json
{
  "model": "gpt-5.1",
  "instructions": "…",
  "input": [ /* ResponseItem[] */ ],
  "tools": [ /* function tool defs */ ],
  "tool_choice": "auto",
  "parallel_tool_calls": false,
  "store": false,
  "stream": true,
  "include": [],
  "prompt_cache_key": "…",
  "text": { "verbosity": "high" }
}
```

`input`은 “대화/툴 호출/툴 결과”를 모두 포함하는 배열이며, 대표적인 항목은 아래와 같습니다.

- 사용자 메시지
  - `{ "type": "message", "role": "user", "content": [{ "type": "input_text", "text": "..." }] }`
- 모델이 요청한 함수 호출(스트리밍 응답에서 옴)
  - `{ "type": "function_call", "call_id": "...", "name": "shell_command", "arguments": "{\"command\":\"ls\"}" }`
- 함수 호출 결과(다음 턴 요청의 input에 포함되어 서버로 전송됨)
  - `{ "type": "function_call_output", "call_id": "...", "output": "stdout..." }`

> 구현 팁(PII 마스킹): Gateway에서 마스킹하려면 `input[].type == "message"`의 `content[].text`, 그리고 `function_call_output.output`(툴 출력)에 민감정보가 섞일 수 있음을 전제로 처리하는 것이 안전합니다.

#### 응답 (SSE: `text/event-stream`)

Codex는 SSE 이벤트를 파싱하며, 대표 이벤트는 아래와 같습니다.

- `response.created`: `data.response.id` 포함
- `response.output_item.added`: `data.item`에 메시지/리즈닝/함수호출 등 “아이템”이 추가됨
- `response.output_text.delta`: `data.delta`에 출력 텍스트의 증분이 옴
- `response.output_item.done`: `data.item`이 완료됨(예: function_call이 여기로 옴)
- `response.reasoning_summary_text.delta`, `response.reasoning_text.delta`: 리즈닝 요약/본문 델타
- `response.completed`: `data.response.usage.total_tokens` 등 토큰 사용량 포함(가능한 경우)
- `response.failed`: `data.response.error.code`, `data.response.error.message`

Gateway가 **사용량 집계**를 하려면 `response.completed`의 `usage.total_tokens`를 저장하는 방식이 가장 단순합니다.

한도 초과 처리(권장): `429` + 아래 JSON 바디

```json
{
  "error": {
    "type": "usage_limit_reached",
    "plan_type": "team",
    "resets_at": 1735722000
  }
}
```

---

### 2.2 자동 컴팩션: `POST /v1/responses/compact` (비스트리밍)

Codex는 대화가 길어질 때(설정/상황에 따라) 컴팩션 API를 호출할 수 있습니다.

- Method: `POST`
- Path: `<base_url>/responses/compact`
- Content-Type: `application/json`

요청 예:

```json
{
  "model": "gpt-5.1",
  "instructions": "…",
  "input": [ /* ResponseItem[] */ ]
}
```

응답 예:

```json
{
  "output": [ /* ResponseItem[] (압축된 히스토리) */ ]
}
```

---

### 2.3 사용량/한도 UI: `GET /api/codex/usage`

Codex TUI는 `chatgpt_base_url`로 지정된 서버에 요청을 보내 **상태바/카드 형태로 사용량(%)** 을 표시합니다.

- Method: `GET`
- Path:
  - `chatgpt_base_url`에 `/backend-api`가 **포함되지 않으면**: `GET <chatgpt_base_url>/api/codex/usage`
  - 포함되면: `GET <chatgpt_base_url>/wham/usage`

Gateway에서는 보통 전자를 권장합니다(예: `chatgpt_base_url="https://gateway.company.local"`).

요청 헤더:

- `Authorization: Bearer <codex_api_key>`
- `User-Agent: ...`

응답 바디(필수/권장 필드)

`plan_type`과 `rate_limit`(윈도우 1~2개)이 핵심입니다.

```json
{
  "plan_type": "pro",
  "rate_limit": {
    "allowed": true,
    "limit_reached": false,
    "primary_window": {
      "used_percent": 42,
      "limit_window_seconds": 3600,
      "reset_after_seconds": 120,
      "reset_at": 1735689720
    },
    "secondary_window": {
      "used_percent": 5,
      "limit_window_seconds": 86400,
      "reset_after_seconds": 43200,
      "reset_at": 1735722000
    }
  },
  "credits": null
}
```

- `used_percent`는 **0~100 정수**입니다(표시용).
- `reset_at`은 epoch seconds(UTC)입니다.

사용량/한도 계산 권장(예시)

- 1차(Primary) 윈도우: “시간당/분당” 한도(예: TPM/TPH)
  - `used_percent = floor(used / limit * 100)`
  - `limit_window_seconds = 3600`(시간당) 또는 `60`(분당) 등
  - `reset_at = next_window_boundary_epoch_seconds`
- 2차(Secondary) 윈도우: “일/주 단위” 한도(예: 일간 토큰 예산)
  - `limit_window_seconds = 86400`(일간) 또는 `604800`(주간)

---

### 2.4 (옵션) 모델 응답 헤더로 사용량 스냅샷 전달

Codex는 모델 호출(`POST /responses`)의 **HTTP 응답 헤더**에서 아래 커스텀 헤더를 읽어 “rate limit snapshot” 이벤트를 만들 수 있습니다.

- `x-codex-primary-used-percent`
- `x-codex-primary-window-minutes`
- `x-codex-primary-reset-at`
- `x-codex-secondary-used-percent`
- `x-codex-secondary-window-minutes`
- `x-codex-secondary-reset-at`
- `x-codex-credits-has-credits`
- `x-codex-credits-unlimited`
- `x-codex-credits-balance`

Gateway가 `/api/codex/usage` 외에도 “즉시 반영”을 원하면 함께 고려할 수 있습니다.
