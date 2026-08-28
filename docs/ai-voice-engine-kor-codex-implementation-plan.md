# ai-voice-engine-kor 구현 작업지시서

## 1. 문서의 목적

이 문서는 Codex가 `ai-voice-engine-kor` 저장소를 처음부터 단계적으로 구현하기 위한 실행 지시서다.

이 프로젝트의 최종 목표는 **사용자가 웹 브라우저에서 마이크와 스피커를 켜고 한국어 AI 음성 에이전트와 샘플 상담을 진행한 뒤, 별도의 Supervisor Dashboard에서 그 통화의 발화, 상태 변화, 도구 실행, 지연시간과 최종 결과를 한눈에 확인할 수 있는 단순하지만 실제 작동하는 데모**를 만드는 것이다.

이번 구현은 콜센터 완제품, 자체 음성 기반모델, 실제 전화망 서비스가 아니다. 핵심은 다음 세 가지를 끝까지 연결해 증명하는 것이다.

1. 브라우저에서 실제 양방향 음성 대화를 할 수 있다.
2. 대화 중 발생한 주요 사건이 구조화되어 서버에 저장된다.
3. Supervisor Dashboard에서 통화 진행 상황과 결과를 관찰할 수 있다.

---

## 2. 최종 데모 시나리오

데모 사용자는 웹에서 `Caller Lab` 화면을 연다.

1. 사용자가 `상담 시작` 버튼을 누른다.
2. 브라우저가 마이크 권한을 요청한다.
3. 연결이 완료되면 AI가 한국어로 먼저 인사한다.
4. 사용자는 “이번 달 요금이 왜 이렇게 많이 나왔나요?”라고 질문한다.
5. AI는 샘플 고객 확인 절차를 진행한다.
6. AI가 mock billing tool을 호출해 청구 내역을 조회한다.
7. AI는 조회 결과를 근거로 요금 증가 원인을 설명한다.
8. 사용자가 AI 발화 중 끼어들면 AI는 가능한 한 빠르게 기존 발화를 중단한다.
9. 통화가 끝나면 세션 상태가 `completed`로 바뀐다.
10. Supervisor Dashboard에는 통화 목록, 전체 transcript, tool call, 주요 latency, 종료 결과가 표시된다.

Supervisor Dashboard는 화려할 필요가 없다. 다만 데모를 보는 사람이 다음 질문에 즉시 답할 수 있어야 한다.

- 지금 통화 중인가, 끝난 통화인가?
- 고객과 AI가 어떤 말을 주고받았는가?
- AI가 어떤 업무 도구를 왜 실행했는가?
- 도구 실행 결과는 무엇인가?
- 응답이 얼마나 느렸는가?
- 통화는 해결, 상담원 전환 필요, 실패 중 무엇으로 끝났는가?

---

## 3. 범위

### 반드시 구현한다

- React 19 기반 Caller Lab
- React 19 기반 Supervisor Dashboard
- 브라우저 마이크 입력과 스피커 출력
- LiveKit 기반 WebRTC 세션
- Python 기반 voice runtime
- 하나의 realtime voice model provider 연동
- 한국어 샘플 상담 프롬프트
- 사용자 끼어들기와 AI 응답 취소
- mock 고객 조회와 mock 청구 조회 도구
- 통화 세션, 발화, tool call, 주요 event 저장
- 진행 중인 통화 상태의 Dashboard 반영
- 종료된 통화 상세 조회
- 최소한의 자동화 테스트
- Docker Compose 기반 로컬 실행
- `.env.example`과 실행 방법 문서화

### 이번 구현에서는 제외한다

- 실제 PSTN/SIP 전화번호 연결
- Twilio 등 실제 전화사업자 연동
- Kubernetes
- Kafka
- Redis
- Neo4j
- 벡터 데이터베이스와 범용 RAG
- `contexture-bridge` 실제 연동
- 다중 에이전트 orchestration
- 실제 상담원의 음성 takeover
- 자체 STT/TTS/음성모델 학습
- 복잡한 인증·조직·권한 시스템
- 실제 고객 개인정보 사용
- 프로덕션 과금·멀티테넌시

제외 항목은 확장 지점으로 문서에만 기록하고 구현하지 않는다.

---

## 4. 기술 스택

### Frontend

- React 19
- TypeScript strict mode
- Vite
- React Router
- Zustand
- TanStack Query
- Tailwind CSS
- shadcn/ui 또는 최소한의 자체 컴포넌트
- LiveKit Client SDK
- Vitest
- React Testing Library
- Playwright

### Voice Runtime

- Python 3.12 이상
- LiveKit Agents
- 선택한 realtime voice model SDK
- Pydantic v2
- httpx
- pytest
- structlog 또는 Python JSON logging

### Tool API

- FastAPI
- Pydantic v2
- SQLAlchemy 2
- Alembic
- PostgreSQL
- pytest

### Infrastructure

- Docker Compose
- Caddy app gateway for manual AWS deployment
- LiveKit Server
- PostgreSQL
- 서비스별 Dockerfile
- health check

### 패키지 관리

- Frontend: `pnpm`
- Python: `uv`

---

## 5. 설계 원칙

1. **동작하는 수직 슬라이스를 우선한다.** 모든 계층을 조금씩 만든 뒤 마지막에 연결하지 말고, 가장 작은 통화 흐름부터 끝까지 연결한다.
2. **모델 SDK를 domain/application 코드에 흩뿌리지 않는다.** provider adapter 내부에 격리한다.
3. **LLM이 데이터베이스를 직접 조회하거나 수정하지 않는다.** 명시적으로 등록된 tool만 호출한다.
4. **PostgreSQL을 영속 기록의 authoritative source로 사용한다.**
5. **실시간 audio frame을 PostgreSQL에 저장하지 않는다.** 저장 대상은 transcript와 구조화 event다.
6. **원본 음성 녹음은 기본적으로 저장하지 않는다.** 이번 MVP에서는 녹음 저장을 생략해도 된다.
7. **모든 외부 입력은 검증한다.** 모델 tool argument도 신뢰하지 않는다.
8. **모든 중요한 event에는 `call_id`, `event_id`, `occurred_at`을 포함한다.**
9. **시간은 서버에서 UTC로 저장하고 UI에서 지역 시간으로 표현한다.**
10. **과도한 추상화를 피하되 provider 교체 경계는 명확히 둔다.**
11. **PostgreSQL은 기본 persistence adapter다.** application code는 repository
    port와 unit-of-work 경계에 의존하고, PostgreSQL/SQLAlchemy 세부사항은
    adapter 내부에 둔다.
12. **고객사가 다른 RDBMS를 요구할 수 있음을 전제로 한다.** 핵심 application
    logic에 PostgreSQL 전용 SQL, JSONB 연산, dialect-specific behavior를
    흩뿌리지 않는다.
13. **운영 로그와 domain audit event를 구분한다.** `call_events`는 Supervisor와
    감사용 영속 event이고, service log는 장애 분석용 structured stdout log다.
14. **모든 서비스는 초기부터 structured JSON log를 남긴다.** 가능한 모든 로그에
    `service`, `event`, `request_id`, `call_id`, `room_name`, `duration_ms`,
    `error_code` 같은 correlation field를 포함한다.
15. **CloudWatch, Azure Monitor, Google Cloud Logging 같은 특정 cloud vendor
    고유 logging/observability service에 직접 결합하지 않는다.** 기본 출력은
    vendor-neutral stdout JSON log와 DB에 저장되는 domain event다.
16. **각 Phase는 독립적으로 실행·검증된 뒤 다음 Phase로 넘어간다.**
17. **Phase 완료 시 테스트와 문서를 함께 갱신한다.**

---

## 6. 목표 아키텍처

```mermaid
flowchart TD
    C["Caller Lab"] --> L["LiveKit / WebRTC"]
    L --> V["Voice Runtime"]
    V --> M["Realtime Model"]
    V --> A["Tool API"]
    A --> P["PostgreSQL"]
    S["Supervisor Dashboard"] --> A
    V --> A
```

### 서비스 책임

#### `apps/console`

- Caller Lab 제공
- 마이크 권한과 LiveKit room 연결
- AI audio 재생
- 연결 상태와 실시간 transcript 표시
- Supervisor Dashboard 제공
- 통화 목록과 상세 event timeline 표시

#### `services/voice-runtime`

- LiveKit room 참가
- realtime model session 관리
- 사용자와 AI의 turn 관리
- interruption/barge-in 처리
- tool call 수신과 Tool API 호출
- transcript와 runtime event를 Tool API로 전달
- 통화 시작·종료 상태 보고

#### `services/tool-api`

- 브라우저용 LiveKit access token 발급
- 통화 세션 생성·조회
- transcript와 event 저장
- mock customer/billing tool 제공
- Supervisor Dashboard 조회 API 제공
- 통화 결과 계산 및 저장

#### `postgres`

- 통화 및 감사 데이터 영속 저장

#### `livekit`

- 브라우저와 voice runtime 사이의 실시간 미디어 전달

---

## 7. 권장 저장소 구조

```text
ai-voice-engine-kor/
├── apps/
│   └── console/
│       ├── src/
│       │   ├── app/
│       │   ├── features/
│       │   │   ├── caller/
│       │   │   ├── transcript/
│       │   │   ├── supervisor/
│       │   │   └── call-detail/
│       │   ├── components/
│       │   ├── hooks/
│       │   ├── stores/
│       │   └── lib/
│       ├── tests/
│       └── package.json
├── services/
│   ├── voice-runtime/
│   │   ├── src/voice_engine/
│   │   │   ├── application/
│   │   │   ├── domain/
│   │   │   ├── ports/
│   │   │   ├── adapters/
│   │   │   └── entrypoint.py
│   │   ├── tests/
│   │   └── pyproject.toml
│   └── tool-api/
│       ├── src/tool_api/
│       │   ├── api/
│       │   ├── calls/
│       │   ├── tools/
│       │   ├── events/
│       │   ├── persistence/
│       │   └── main.py
│       ├── migrations/
│       ├── tests/
│       └── pyproject.toml
├── contracts/
│   ├── openapi/
│   ├── json-schema/
│   └── events/
├── prompts/
│   ├── system/
│   └── scenarios/
├── evals/
│   ├── scenarios/
│   ├── fixtures/
│   └── reports/
├── infra/
│   ├── livekit/
│   └── postgres/
├── docs/
│   ├── architecture/
│   ├── adr/
│   └── runbooks/
├── compose.yaml
├── Makefile
├── .env.example
└── README.md
```

---

## 8. 핵심 데이터 모델

### `call_sessions`

- `id`: UUID
- `room_name`: string, unique
- `scenario_code`: string
- `status`: `created | connecting | active | completed | failed`
- `outcome`: `resolved | handoff_required | abandoned | failed | unknown`
- `model_provider`: string
- `started_at`: timestamp nullable
- `ended_at`: timestamp nullable
- `created_at`: timestamp

### `utterances`

- `id`: UUID
- `call_id`: UUID FK
- `speaker`: `customer | agent | supervisor | system`
- `text`: text
- `is_final`: boolean
- `started_at`: timestamp nullable
- `ended_at`: timestamp nullable
- `sequence_number`: integer
- `created_at`: timestamp

### `call_events`

- `id`: UUID
- `call_id`: UUID FK
- `event_type`: string
- `payload`: JSONB
- `occurred_at`: timestamp

초기 event type 예:

- `call.created`
- `call.connected`
- `call.ended`
- `participant.joined`
- `participant.left`
- `customer.speech_started`
- `customer.speech_ended`
- `agent.response_started`
- `agent.response_cancelled`
- `agent.response_completed`
- `tool.requested`
- `tool.completed`
- `tool.failed`
- `handoff.requested`

### `tool_executions`

- `id`: UUID
- `call_id`: UUID FK
- `tool_name`: string
- `arguments`: JSONB
- `result`: JSONB nullable
- `status`: `requested | completed | failed`
- `duration_ms`: integer nullable
- `created_at`: timestamp
- `completed_at`: timestamp nullable

### `call_metrics`

- `call_id`: UUID FK
- `first_response_ms`: integer nullable
- `average_response_ms`: integer nullable
- `tool_average_ms`: integer nullable
- `interruption_count`: integer
- `utterance_count`: integer
- `updated_at`: timestamp

---

## 9. API 최소 계약

### 세션 및 LiveKit

- `POST /api/calls`
  - 새 통화 세션을 생성한다.
  - `call_id`, `room_name`을 반환한다.
- `POST /api/calls/{call_id}/token`
  - Caller Lab이 사용할 LiveKit token을 반환한다.
- `POST /api/calls/{call_id}/start`
- `POST /api/calls/{call_id}/end`

### runtime event

- `POST /api/calls/{call_id}/utterances`
- `POST /api/calls/{call_id}/events`
- `POST /api/calls/{call_id}/metrics`

### mock tools

- `POST /api/tools/verify-customer`
- `POST /api/tools/get-billing-summary`

### Supervisor Dashboard

- `GET /api/supervisor/calls`
- `GET /api/supervisor/calls/{call_id}`
- `GET /api/supervisor/calls/{call_id}/timeline`
- `GET /api/supervisor/calls/{call_id}/stream`

마지막 endpoint는 MVP에서 Server-Sent Events를 사용한다. WebSocket 서버를 별도로 만들지 않는다.

---

## 10. Phase별 구현 계획

## Phase 0 — 저장소 생성과 범위 고정

### 작업

1. `ai-voice-engine-kor` 저장소를 생성한다.
2. 루트 README에 목적, 범위, 제외 범위를 기록한다.
3. `.gitignore`, `.editorconfig`, `.env.example`을 만든다.
4. `docs/adr/0001-mvp-scope.md`를 작성한다.
5. `main` 브랜치가 빈 상태에서도 CI를 통과하도록 기본 workflow를 만든다.

### 완료 조건

- README만 읽어도 “브라우저 음성 상담과 Supervisor 관찰 데모”라는 목표가 분명하다.
- 실제 전화망, Kafka, Neo4j 등이 MVP 범위에서 제외되었음이 명시돼 있다.
- 비밀키가 저장소에 들어가지 않는다.

### 검증

```bash
git status --short
git grep -nE '(API_KEY|SECRET|PASSWORD)=' -- ':!*.example'
```

---

## Phase 1 — 모노레포와 개발 도구 스캐폴딩

### 작업

1. `apps/console`에 React 19 + TypeScript + Vite 프로젝트를 만든다.
2. `services/tool-api`에 FastAPI 프로젝트를 만든다.
3. `services/voice-runtime`에 Python 패키지를 만든다.
4. `pnpm-workspace.yaml`, Python `pyproject.toml`, Makefile을 구성한다.
5. lint, type check, unit test 명령을 통일한다.
6. 모든 service가 structured JSON stdout logging을 사용하도록 logging 기본값을
   만든다.
7. cloud vendor 고유 logging SDK나 CloudWatch 전용 logger를 application code에
   직접 추가하지 않는다.

### 필수 명령

```bash
make install
make lint
make typecheck
make test
```

### 완료 조건

- React 초기 화면이 로컬에서 열린다.
- 두 Python 서비스가 import 오류 없이 시작한다.
- 모든 테스트 명령이 실제 테스트를 최소 한 개 이상 실행한다.
- placeholder 명령으로 성공을 위장하지 않는다.
- Tool API와 voice-runtime이 service 이름과 event 이름을 포함한 JSON log를
  stdout에 남긴다.

---

## Phase 2 — PostgreSQL과 Tool API 기초

### 작업

1. PostgreSQL Compose 서비스를 추가한다.
2. SQLAlchemy model과 Alembic migration을 만든다.
3. `call_sessions`, `utterances`, `call_events`, `tool_executions`, `call_metrics`를 구현한다.
4. Tool API health endpoint를 구현한다.
5. 통화 생성·목록·상세 조회 API를 구현한다.
6. API 오류 형식을 하나로 통일한다.
7. persistence repository port와 PostgreSQL adapter 경계를 만든다.
8. API handler가 SQLAlchemy query나 PostgreSQL dialect detail에 직접 의존하지
   않도록 한다.

### 완료 조건

- 빈 DB에서 migration이 성공한다.
- 통화 생성 후 목록과 상세 조회가 가능하다.
- 존재하지 않는 통화는 일관된 404 응답을 반환한다.
- 테스트마다 별도 test database 또는 transaction isolation을 사용한다.
- application/service layer가 PostgreSQL adapter 교체 가능 경계를 가진다.
- event payload는 JSON으로 확장 가능하게 두되, 자주 조회할 status, outcome,
  speaker, tool_name, occurred_at 같은 값은 column으로 유지한다.

### 검증

```bash
docker compose up -d postgres
make db-migrate
make test-api
curl -fsS http://localhost:8000/health
```

---

## Phase 3 — Mock 업무 도구 구현

### 작업

1. 가상의 고객 2~3명과 청구 데이터 fixture를 만든다.
2. `verify-customer` tool을 구현한다.
3. `get-billing-summary` tool을 구현한다.
4. tool 실행 전후를 `tool_executions`와 `call_events`에 기록한다.
5. 허용되지 않은 고객 ID, 잘못된 argument, 중복 요청을 테스트한다.

### 샘플 청구 사유

- 기본요금 55,000원
- 데이터 추가 사용 12,000원
- 해외 로밍 18,000원
- 전월 대비 30,000원 증가

### 완료 조건

- 동일 fixture 입력은 결정적인 결과를 반환한다.
- 모든 tool execution은 성공과 실패 모두 기록된다.
- tool 결과에는 사람이 읽을 수 있는 설명과 구조화된 금액 데이터가 함께 포함된다.

---

## Phase 4 — Supervisor Dashboard 정적 골격

### 작업

1. `/supervisor`에 통화 목록 화면을 만든다.
2. `/supervisor/calls/:callId`에 상세 화면을 만든다.
3. 상태 badge, 시작 시각, duration, outcome을 표시한다.
4. transcript와 event timeline을 구분해 표시한다.
5. tool execution card를 만든다.
6. 빈 상태, loading, 오류 상태를 구현한다.

### 화면 배치

- 좌측: 통화 목록
- 중앙: 시간순 transcript
- 우측: 상태, metrics, tool execution

작은 화면에서는 세 영역을 탭으로 전환해도 된다.

### 완료 조건

- Phase 2와 3에서 생성한 DB 데이터를 UI에서 볼 수 있다.
- 서버 fixture가 아니라 실제 API를 호출한다.
- 통화 상세 URL을 새로고침해도 동일 상세 화면이 열린다.
- UI는 단순하되 정보 계층이 명확하다.

### 검증

```bash
make test-web
make test-e2e-supervisor
```

---

## Phase 5 — LiveKit 로컬 미디어 인프라

### 작업

1. LiveKit Server를 Compose에 추가한다.
2. 개발용 key/secret은 `.env.example`에 placeholder로만 제공한다.
3. Tool API에 caller token 발급 endpoint를 구현한다.
4. Voice Runtime용 agent identity와 token 정책을 구분한다.
5. Caller Lab에서 room 연결·해제를 구현한다.
6. 마이크 enable/disable과 audio output 상태를 표시한다.

### 완료 조건

- 브라우저에서 권한을 허용하면 room에 참가한다.
- 마이크 트랙이 publish된다.
- 연결 상태가 `idle → connecting → connected → disconnected`로 표시된다.
- 브라우저에 LiveKit secret을 노출하지 않는다.
- 새로고침이나 연결 해제 후 트랙과 room resource가 정리된다.

### 검증

```bash
docker compose up --build
docker compose ps
make test-e2e-media
```

Playwright의 가상 마이크 또는 fake media stream을 사용해 최소한의 연결 테스트를 자동화한다. 실제 음질 평가는 수동으로 한다.

---

## Phase 6 — Voice Runtime과 Echo 수직 슬라이스

### 작업

1. Voice Runtime이 지정된 LiveKit room에 agent로 참가하도록 한다.
2. participant join/leave를 감지한다.
3. 실제 모델을 연결하기 전에 echo 또는 고정 음성 응답으로 media round trip을 검증한다.
4. call 상태와 participant event를 Tool API에 기록한다.
5. 종료 signal과 graceful shutdown을 처리한다.

### 완료 조건

- 브라우저가 보낸 음성이 Voice Runtime에 도달한다.
- 테스트 응답 음성이 브라우저 스피커로 재생된다.
- Dashboard에서 room 접속과 종료 event가 보인다.
- 프로세스를 재시작해도 고아 세션을 가능한 범위에서 정리한다.

이 Phase가 통과하기 전에는 realtime model을 연결하지 않는다.

---

## Phase 7 — Realtime Voice Model 연결

### 작업

1. `RealtimeVoiceProvider` port를 정의한다.
2. 하나의 provider adapter만 먼저 구현한다.
3. API key와 model name을 환경변수로 받는다.
4. 한국어 시스템 프롬프트를 연결한다.
5. AI가 먼저 짧게 인사하도록 한다.
6. input/output transcript event를 수집한다.
7. 최종 transcript를 Tool API에 저장한다.
8. provider 오류를 call failure event로 변환한다.

### 최소 인터페이스

```python
class RealtimeVoiceProvider(Protocol):
    async def connect(self, config: SessionConfig) -> None: ...
    async def send_audio(self, frame: AudioFrame) -> None: ...
    async def cancel_response(self) -> None: ...
    async def send_tool_result(self, result: ToolResult) -> None: ...
    def events(self) -> AsyncIterator[VoiceEvent]: ...
```

### 완료 조건

- 사용자가 한국어로 말하면 AI의 한국어 음성 답변을 실제로 듣는다.
- 사용자와 AI의 최종 transcript가 DB에 순서대로 저장된다.
- provider-specific event가 application 바깥으로 직접 새지 않는다.
- API key가 로그, 브라우저, DB에 기록되지 않는다.

---

## Phase 8 — 한국어 상담 프롬프트와 Tool Calling

### 작업

1. 청구 문의 전용 시스템 프롬프트를 작성한다.
2. AI가 고객 확인 전에는 청구 정보를 제공하지 않도록 한다.
3. `verify_customer`와 `get_billing_summary` tool schema를 provider에 등록한다.
4. model tool call을 검증해 Tool API로 전달한다.
5. tool result를 model session으로 되돌린다.
6. AI가 금액과 증가 사유를 한국어로 설명하도록 한다.
7. tool call의 시작, 성공, 실패와 duration을 저장한다.

### 프롬프트 행동 규칙

- 기본적으로 존댓말을 사용한다.
- 모르는 정보는 추측하지 않는다.
- 고객 확인 전 개인정보와 청구 상세를 말하지 않는다.
- 금액은 통화 단위와 함께 명확히 읽는다.
- tool 실패 시 실패 사실을 숨기지 않는다.
- 해결할 수 없으면 상담원 연결 필요 상태로 종료한다.
- 답변은 음성 대화에 맞게 짧게 나눈다.

### 완료 조건

- 최종 데모의 청구 문의가 처음부터 끝까지 수행된다.
- transcript만 보아도 tool call 전후 맥락을 이해할 수 있다.
- Dashboard에서 tool argument, result, duration을 확인할 수 있다.
- 잘못된 tool argument는 실행되지 않고 validation failure로 기록된다.

---

## Phase 9 — 끼어들기와 응답 취소

### 작업

1. 고객 speech start event를 감지한다.
2. AI가 발화 중이면 provider response cancel을 호출한다.
3. 이미 전송된 audio playback을 가능한 범위에서 중단한다.
4. `agent.response_cancelled` event를 기록한다.
5. interruption count를 갱신한다.
6. 취소 이후 다음 고객 발화가 정상 처리되는지 검증한다.

### 완료 조건

- 사용자가 AI 발화 중 말하면 AI 음성이 실제로 중단된다.
- 중단된 응답을 완료된 발화처럼 transcript에 기록하지 않는다.
- Dashboard에서 interruption 시점과 횟수를 확인할 수 있다.
- 끼어들기 이후 대화 세션이 깨지지 않는다.

### 수동 검증

1. AI에게 청구 상세를 설명하게 한다.
2. AI가 두 번째 문장을 말할 때 “잠깐만요”라고 끼어든다.
3. AI 음성이 멈추는지 확인한다.
4. 새로운 질문에 답하는지 확인한다.

---

## Phase 10 — 실시간 Supervisor 갱신

### 작업

1. `GET /api/supervisor/calls/{call_id}/stream` SSE를 구현한다.
2. 새 utterance, event, tool execution, metric을 순서대로 전송한다.
3. React에서 reconnect와 last event handling을 구현한다.
4. Supervisor 상세 화면이 polling 없이 주요 정보를 갱신하도록 한다.
5. SSE가 끊겨도 REST 상세 조회로 상태를 복구한다.

### 완료 조건

- Caller Lab에서 말한 내용이 Supervisor 화면에 거의 실시간으로 나타난다.
- tool 실행 상태가 `requested → completed/failed`로 바뀐다.
- 브라우저 재접속 후 누락 없이 현재 상태를 복구한다.
- 동일 event가 UI에 중복 표시되지 않는다.

---

## Phase 11 — 통화 종료와 결과 요약

### 작업

1. 사용자가 `상담 종료` 버튼을 누를 수 있게 한다.
2. participant 종료와 runtime 종료를 idempotent하게 처리한다.
3. 통화 outcome을 계산한다.
4. duration, utterance count, interruption count, response latency를 집계한다.
5. Supervisor 상세 상단에 결과 summary card를 표시한다.

### MVP outcome 규칙

- billing tool 성공 후 설명 완료: `resolved`
- 상담원 요청 또는 tool로 해결 불가: `handoff_required`
- 대화 시작 후 즉시 종료: `abandoned`
- provider 또는 시스템 오류: `failed`
- 그 밖의 경우: `unknown`

### 완료 조건

- 여러 번 종료 요청해도 상태가 깨지지 않는다.
- 종료된 통화는 다시 `active`로 돌아가지 않는다.
- Dashboard에서 결과와 기본 metrics를 한눈에 확인할 수 있다.

---

## Phase 12 — 데모 UX 정리

### Caller Lab 필수 요소

- 시나리오 제목과 한 줄 설명
- 상담 시작/종료 버튼
- 마이크 on/off
- 연결 상태
- AI speaking / customer speaking 표시
- 실시간 transcript
- 개인정보를 입력하지 말라는 안내

### Supervisor Dashboard 필수 요소

- 진행 중/종료 통화 목록
- 선택한 통화의 상태와 outcome
- 화자별 transcript
- tool execution 카드
- event timeline
- first response, 평균 응답, tool duration
- interruption count

### 완료 조건

- 처음 보는 사람이 별도 설명 없이 상담을 시작할 수 있다.
- 오류가 발생하면 개발자 console을 열지 않아도 화면에서 알 수 있다.
- 모바일 완전 대응은 필요 없지만 1280px 화면에서 깨지지 않는다.
- 화려한 animation보다 상태와 정보 전달을 우선한다.

---

## Phase 13 — 통합 테스트와 실패 복구

### 자동화할 흐름

1. 통화 생성
2. LiveKit token 발급
3. mock transcript/event 저장
4. tool 실행
5. Supervisor 상세 조회
6. 통화 종료
7. outcome 및 metrics 확인

### 수동 검증할 흐름

1. 실제 브라우저 마이크 권한 승인
2. 실제 한국어 음성 대화
3. AI 음성 출력
4. 끼어들기
5. 실시간 Supervisor 반영
6. 통화 종료 후 결과 확인

### 실패 시나리오

- 마이크 권한 거부
- LiveKit 연결 실패
- model API key 누락
- provider timeout
- tool validation 실패
- Tool API 일시 중단
- Supervisor SSE 재연결

### 완료 조건

- 핵심 happy path E2E test가 CI에서 통과한다.
- 외부 model이 필요한 테스트와 필요 없는 테스트를 분리한다.
- 외부 비용이 발생하는 테스트는 기본 CI에서 실행하지 않는다.
- 실패 시 session이 무한히 `connecting` 또는 `active`로 남지 않는다.

---

## Phase 14 — Docker Compose 완성

### 로컬 개발 최종 서비스

```yaml
services:
  console:
  tool-api:
  voice-runtime:
  postgres:
  livekit:
```

`console`은 로컬 개발에서 Vite 개발 서버로 실행한다. AWS 수동 배포에서는
React 정적 빌드를 Caddy가 직접 서빙하므로 별도 `console` 런타임 컨테이너를
필수로 두지 않는다.

### AWS 수동 배포 서비스

```yaml
services:
  caddy:
  tool-api:
  voice-runtime:
  postgres:
  livekit:
```

Caddy는 AWS 배포에서 app gateway 역할을 맡는다.

- React 정적 파일 서빙
- SPA fallback 처리
- `/api/*` 요청을 `tool-api`로 reverse proxy
- Supervisor SSE endpoint의 proxy buffering 방지
- ALB health check용 HTTP endpoint 제공

FastAPI는 API와 SSE에 집중한다. FastAPI가 React static을 직접 서빙하는 전략은
가능하지만, 이 프로젝트에서는 장기 구조 변경을 줄이기 위해 AWS 배포의 HTTP
진입점을 Caddy로 분리한다.

### 작업

1. 모든 서비스에 multi-stage Dockerfile을 적용한다.
2. health check와 `depends_on.condition`을 적절히 사용한다.
3. named volume은 PostgreSQL에만 필수로 사용한다.
4. source bind mount는 개발 profile에서만 사용한다.
5. 종료 signal이 각 서비스에 전달되도록 한다.
6. API key와 secret은 Compose 파일에 하드코딩하지 않는다.
7. Caddy 배포 image 또는 stage는 `apps/console`의 production build 결과만
   포함한다.
8. Docker build context에서 `.fordeploy/`, env 파일, credential 파일, image
   tarball이 제외되도록 한다.

### 완료 조건

```bash
cp .env.example .env
# 사용자가 필요한 외부 API key를 입력
docker compose up --build
```

위 과정 후 다음이 가능해야 한다.

- Caller Lab 접속
- Supervisor Dashboard 접속
- 실제 음성 대화
- 통화 결과 확인

---

## Phase 14B — AWS 수동 배포 전략

이 Phase는 MVP의 로컬 Compose 검증을 대체하지 않는다. 로컬 Compose는 개발과
테스트의 기준 경로이고, AWS 수동 배포는 같은 애플리케이션을 장기적으로 일관된
구조로 운영하기 위한 선택 경로다.

### 목표 구조

```text
Internet
  -> AWS ALB :443, ACM TLS
      -> app target group
          -> caddy container
              -> React static
              -> /api/* -> tool-api
              -> /api/supervisor/*/stream -> tool-api SSE

  -> LiveKit public endpoint
      -> livekit container
          -> WebSocket signaling
          -> WebRTC media ports
```

AWS 배포에서 권장 컨테이너 수는 5개다.

1. `caddy`
2. `tool-api`
3. `voice-runtime`
4. `postgres`
5. `livekit`

### ALB와 LiveKit 경계

- ALB는 `app` HTTP traffic의 기본 진입점이다.
- ALB와 ACM이 public HTTPS 인증서를 담당한다.
- Caddy는 ALB 뒤에서 HTTP app gateway로 동작한다. Caddy의 자동 HTTPS 기능은
  직접 public TLS를 맡길 때 유용하지만, 이 프로젝트에서는 Caddy 구축 경험과
  명확한 gateway 분리를 위해 사용한다.
- 비용 절감형 demo 배포에서는 기존 public `aws-bastion` EC2를 runtime host로
  사용하는 전략을 허용한다. 이 경우에도 ALB는 계속 사용하되, 추가 NLB와 NAT
  gateway는 기본적으로 만들지 않는다.
- LiveKit은 app HTTP traffic과 분리된 public hostname을 사용한다.
- LiveKit signaling은 `wss://` endpoint로 제공한다.
- WebRTC media port는 ALB만으로 모두 해결된다고 가정하지 않는다. 단일 public
  EC2 수동 배포에서는 LiveKit에 필요한 media port를 해당 EC2 security group과
  host port에서 직접 허용하는 방식을 우선 검토한다.
- NLB, NAT gateway, 별도 TURN infrastructure는 기본 범위가 아니다. 실제
  네트워크 검증에서 직접 port exposure로 해결되지 않을 때만 추가 검토한다.
- 브라우저에 노출되는 값은 public URL뿐이어야 한다. LiveKit API secret과
  provider API key는 서버와 runtime에만 존재한다.
- `t3a.micro`는 비용 절감형 단일 사용자 demo host로 시도할 수 있다. 다만
  Caddy, Tool API, voice-runtime, PostgreSQL, LiveKit을 모두 같은 host에서
  실행하므로 memory pressure나 swap 사용, 음성 latency 문제가 보이면 그 시점에
  즉시 `t3a.small`로 상향한다.

예상 hostname:

```text
app.example.com      -> ALB -> caddy
livekit.example.com  -> LiveKit signaling/media policy
```

### 수동 배포 방식

- AWS 배포는 항상 수동으로 수행한다.
- CI/CD, hosted build pipeline, registry push, Kubernetes, 자동 배포 hook은
  기본 범위가 아니다.
- 기본 demo target은 기존 public `aws-bastion` host다. 이 host는 bastion
  역할도 겸하므로 보안상 이상적인 분리는 아니지만, 비용 절감을 위한 demo
  runtime host로 허용한다.
- `aws-bastion`에는 Docker, Docker Compose, AWS CLI가 설치되어 있다고 가정하고,
  script는 가능하면 이 도구들을 재사용한다.
- `.fordeploy/` 아래에 배포 script와 runbook을 둔다.
- `.fordeploy/aws-backup/`은 AWS runtime 파일의 local staging/backup 영역이다.
  application source가 아니다.
- real env 파일, credential 파일, private key, image tarball은 commit하지
  않는다.
- clean clone 또는 명시적으로 선택한 clean source directory에서 Docker image를
  build한다.
- build한 image는 tarball로 저장한 뒤 Bastion/private host 경로로 전송한다.
- target host에서 image를 load하고 application container만 교체한다.
- PostgreSQL volume과 runtime data는 일반 redeploy에서 삭제하거나 recreate하지
  않는다.
- image tarball은 transfer artifact일 뿐이다. local과 remote에서 load 후
  삭제한다.

### AWS target과 전송 정책

기본 비용 절감형 demo에서는 `aws-bastion` 자체가 target host다.

```text
local clean build
  -> docker save image tarballs
  -> copy tarballs to aws-bastion
  -> docker load on aws-bastion
  -> replace caddy/tool-api/voice-runtime containers
  -> health checks
  -> cleanup tarballs
```

나중에 private host를 별도로 둘 경우에는 다음 흐름을 사용한다.

```text
local clean build
  -> docker save image tarballs
  -> copy tarballs to Bastion
  -> copy tarballs to private host
  -> docker load on private host
  -> replace caddy/tool-api/voice-runtime containers
  -> health checks
  -> cleanup tarballs
```

runtime env 파일이나 credential 파일을 AWS로 복사해야 할 때는 script가 다음을
지켜야 한다.

- `.fordeploy/aws-backup/` 안의 후보 파일을 absolute path로 해석한다.
- 복사 전 source absolute path, destination host, destination absolute path를
  출력한다.
- 각 파일마다 terminal에서 `y/N` 확인을 받는다.
- `y` 또는 `Y`가 아니면 해당 파일 전송을 skip하거나 bootstrap step을 중단한다.
- env 파일과 credential 파일의 내용은 절대 출력하지 않는다.
- 파일명, byte size, checksum, timestamp, target path는 출력해도 된다.

AWS target path 예:

```text
/home/ubuntu/ai-voice-engine
/home/ubuntu/docker_images/ai-voice-engine
```

### Script 작성 원칙

- script는 `.fordeploy/` 아래에 둔다.
- shell script는 `set -euo pipefail`을 사용한다.
- image name, tag, container name, port, remote path, health check path,
  Bastion host, private host, SSH user, SSH key path는 환경변수로 조정 가능해야
  한다.
- build, save, transfer, load, replace, health check, cleanup의 major step을
  실행 전에 출력한다.
- secret이나 full env 파일을 출력하지 않는다.
- application image deploy와 runtime file bootstrap은 분리한다.
- application redeploy가 target host의 env 파일을 암묵적으로 덮어쓰면 안 된다.
- AWS에 필요한 runtime 파일이 없으면 빠르게 실패하되, 오류 메시지는 필요한
  target path를 설명하고 secret 내용을 출력하지 않는다.
- destructive reset script는 이름과 문서에서 파괴적 작업임을 명확히 표시해야
  하며, 일반 배포 script에서 PostgreSQL volume을 삭제하면 안 된다.

### 완료 조건

- `.fordeploy/README.md`에 AWS 수동 배포 절차가 문서화되어 있다.
- Caddy가 React static과 `/api/*` proxy를 제공한다.
- Supervisor SSE가 Caddy와 ALB 뒤에서 동작한다.
- LiveKit public endpoint와 media port 정책이 문서화되어 있다.
- 기본 demo 배포가 기존 public `aws-bastion` target을 지원한다.
- 추가 NLB와 NAT gateway 없이 동작하는 경로를 먼저 검증한다.
- `t3a.micro`에서 memory pressure나 latency 문제가 확인되면 `t3a.small`로
  올리는 절차와 판단 기준이 runbook에 기록되어 있다.
- clean source에서 image를 build하고 tarball로 전송하는 script 또는 runbook이
  존재한다.
- 배포 후 app HTTP, Tool API health, LiveKit availability, PostgreSQL 연결을
  확인한다.
- 실제 secret, env 파일, credential 파일, image tarball이 Git과 Docker image에
  포함되지 않는다.

---

## Phase 15 — README와 최종 데모 문서

### README 필수 항목

1. 프로젝트 한 문장 설명
2. 실제 구현된 기능
3. 아키텍처 다이어그램
4. 기술 스택
5. 사전 요구사항
6. 환경변수 설명
7. Docker Compose 실행법
8. 데모 시나리오
9. 테스트 방법
10. 개인정보·보안 주의사항
11. AWS 수동 배포 전략 요약
12. 알려진 한계
13. 향후 확장 방향

### 추가 문서

- `docs/runbooks/demo.md`: 3분 데모 순서
- `docs/runbooks/aws-manual-deploy.md`: Caddy, ALB, Bastion/private host 기반
  수동 배포 순서
- `docs/architecture/runtime-events.md`: event 목록과 의미
- `docs/adr/0002-livekit-media-plane.md`
- `docs/adr/0003-postgres-authoritative-store.md`
- `docs/adr/0004-sse-for-supervisor-mvp.md`
- `docs/adr/0005-caddy-aws-app-gateway.md`

### 완료 조건

- 새 개발자가 README만 보고 실행할 수 있다.
- 3분 데모 문서를 따라가면 준비된 시나리오가 재현된다.
- 구현되지 않은 기능을 구현된 것처럼 표현하지 않는다.

---

## 11. Codex 작업 방식

Codex는 다음 규칙을 지킨다.

1. 작업 시작 전에 현재 repository 상태와 기존 변경사항을 확인한다.
2. 기존 사용자 변경을 덮어쓰거나 삭제하지 않는다.
3. 한 번에 하나의 Phase만 `in_progress`로 둔다.
4. 각 Phase 안에서도 가능한 가장 작은 수직 단위로 구현한다.
5. 파일 편집 후 관련 test, lint, type check를 실행한다.
6. 실패를 발견하면 원인을 수정하고 검증이 성공할 때까지 해당 Phase를 완료 처리하지 않는다.
7. 외부 API key가 없어 검증할 수 없는 항목은 자동 통과 처리하지 않는다. 검증하지 못한 항목과 필요한 수동 절차를 정확히 기록한다.
8. dependency를 추가할 때 이유와 사용 위치가 분명해야 한다.
9. 임시 mock과 production path를 코드와 문서에서 명확히 구분한다.
10. Phase마다 변경 파일, 실행한 검증, 남은 위험을 간결하게 보고한다.
11. 각 Phase 종료 시 `git diff --check`를 실행한다.
12. 사용자가 요청하지 않는 한 원격 push, 배포, 유료 인프라 생성은 하지 않는다.
13. AWS 수동 배포 작업은 Caddy app gateway, ALB/ACM, LiveKit public endpoint,
    Bastion/private host 전송, clean clone image build 전략을 따른다.

### Phase 완료 보고 형식

```text
Phase N 완료

- 구현:
- 주요 파일:
- 검증:
- 수동 확인 필요:
- 알려진 한계:
- 다음 Phase:
```

---

## 12. 환경변수 초안

```dotenv
# Public URLs
VITE_API_BASE_URL=http://localhost:8000
VITE_LIVEKIT_URL=ws://localhost:7880

# LiveKit server-side credentials
LIVEKIT_URL=ws://livekit:7880
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=replace-with-a-long-local-secret

# Database
POSTGRES_DB=voice_engine
POSTGRES_USER=voice_engine
POSTGRES_PASSWORD=replace-with-a-local-password
DATABASE_URL=postgresql+psycopg://voice_engine:replace-with-a-local-password@postgres:5432/voice_engine

# Realtime model
VOICE_PROVIDER=openai
VOICE_MODEL=
OPENAI_API_KEY=
GOOGLE_API_KEY=

# Application
APP_ENV=development
LOG_LEVEL=INFO
```

실제 provider와 model identifier는 구현 시점의 공식 SDK 문서를 확인한 후 설정한다. 특정 preview model name을 코드 기본값으로 영구 고정하지 않는다.

---

## 13. MVP 품질 기준

### 기능

- 브라우저에서 실제 한국어 음성 대화가 된다.
- AI 음성이 스피커로 나온다.
- AI 발화 중 사용자가 끼어들 수 있다.
- 청구 조회 tool이 실제 API를 통해 실행된다.
- 통화 기록과 결과가 Supervisor Dashboard에 표시된다.

### 안정성

- 한 번의 데모에서 서비스 재시작 없이 최소 3회 연속 통화할 수 있다.
- 통화 종료 후 마이크와 media resource가 해제된다.
- provider 오류가 전체 애플리케이션 crash로 이어지지 않는다.

### 보안·개인정보

- secret이 브라우저 bundle이나 repository에 포함되지 않는다.
- 실제 주민번호, 전화번호, 결제정보를 요구하지 않는다.
- fixture임을 UI와 README에 표시한다.

### 관찰 가능성

- 모든 로그를 `call_id`로 추적할 수 있다.
- tool 실패와 provider 실패를 구분할 수 있다.
- Supervisor에서 최소 latency와 interruption 지표를 볼 수 있다.
- operational log는 structured stdout JSON으로 남기고, domain audit event는
  PostgreSQL의 `call_events`와 관련 table에 저장한다.
- logging/observability 기본 경로는 cloud vendor-neutral이어야 한다.
- CloudWatch, Azure Monitor, Google Cloud Logging 같은 vendor-specific 서비스는
  template 기본 구현에 직접 결합하지 않는다.

---

## 14. 최종 Definition of Done

다음 조건을 모두 만족해야 프로젝트 MVP를 완료로 본다.

- [ ] `docker compose up --build`로 모든 필수 서비스가 실행된다.
- [ ] Caller Lab에서 마이크 권한을 받아 상담을 시작할 수 있다.
- [ ] 한국어로 질문하고 AI 한국어 음성 답변을 들을 수 있다.
- [ ] AI가 mock 고객 확인 및 청구 조회 tool을 호출한다.
- [ ] AI 발화 중 끼어들었을 때 응답이 취소된다.
- [ ] transcript가 고객과 AI 화자로 구분되어 저장된다.
- [ ] tool argument, result, status, duration이 저장된다.
- [ ] Supervisor Dashboard에서 진행 중 통화를 볼 수 있다.
- [ ] Supervisor Dashboard가 transcript와 tool call을 실시간으로 갱신한다.
- [ ] 통화 종료 후 outcome과 기본 metrics가 표시된다.
- [ ] API key가 repository, browser bundle, 로그에 노출되지 않는다.
- [ ] unit, integration, 핵심 E2E test가 통과한다.
- [ ] README와 3분 데모 runbook이 현재 구현과 일치한다.
- [ ] 실제로 구현되지 않은 전화망·멀티모델·RAG 기능을 과장하지 않는다.

---

## 15. MVP 이후 확장 후보

MVP 완료 전에는 아래 항목을 구현하지 않는다.

1. OpenAI Realtime과 Gemini Live 비교 adapter
2. STT → text LLM → TTS cascade 비교
3. 실제 SIP 전화 연결
4. 상담원 warm handoff
5. 통화 녹음과 replay evaluation
6. 한국어 숫자·날짜·주소·고유명사 평가 dataset
7. `contexture-bridge`를 통한 근거 기반 enterprise context 조회
8. PII masking과 retention policy
9. OpenTelemetry Collector 및 Grafana dashboard
10. multi-tenant 조직과 권한 모델

이 목록은 roadmap일 뿐 현재 구현의 완료 조건이 아니다.
