Agent Loop는 Agent의 실제 작업 과정에 대한 설명으로 아래 flow를 따른다.
**입력 -> 컨텍스트 조합 -> LLM 모델 -> tool 실행 -> 스트리밍 응답 -> 지속 저장**
OpenClaw는 직렬화된 단일 세션(대화 마다 순차적으로 한 번만 실행)으로 각 단계별 스트림 이벤트를 발생시키고, 그러한 스트림 이벤트가 다음 flow를 따르게 한다.

## Entry points
Agent Loop의 실행 방법
- Gateway RPC: `agent` and `agent.wait`
	- agent: 에이전트에게 작업을 시작할 때 사용
	- agent.wait: 에이전트가 끝날 때까지 기다렸다, 실행 중인 결과를 확인할 때 사용
	- 만약 웹 API나 OpenClaw를 사용한다면, RPC를 HTTP 요청으로 호출 가능
- CLI:`agent` command
	- 터미널에서 직접 agent 명령어로 에이전트를 실행
	- ex) openclaw agent --model=gpt-4 --query="오늘의 일기 작업 실행해"
### How it works (high-level)
1. agent RPC가 파라미터를 검증하고, 세션을 처리한 후 즉시 { runId, acceptedAt }을 반환
	- 세션 처리: sessionKey나 sessionId로 이전 작업물을 찾거나 현재 작업의 메타데이터를 관리
	- 즉시 { runId, acceptedAt } 반환: 특정 작업을 시작했다는 것을 알려주고, 백그라운드에서 실행함
2. agentCommand가 에이전트 run
	- AI 모델 종류와 사고 옵션 (thinking/verbose) 설정
	- 스킬 스냅샵 로드
	- runEmbeddedPiAgent를 호출 
	- 만약 내부 loop가 끝나지 않으면 end나 error를 발생 
3. runEmbeddedPiAgent
	- 세션별 + 전체 대기열로 작업을 순서대로 처리
	- 모델과 auth를 확인하고, AI 환경 설정
	- pi events를 구독하여 assistant/tool의 변화를 스트리밍
	- 타임아웃을 적용
	- 페이로드(결과 데이터)와 사용량 메타데이터 반환
4. subscribeEmbeddedPiSession
	- **pi-agent-core에서 나오는 이벤트를 OpenClaw의 스트림으로 변환**
	- tool events -> steam: "tool"
	- assistant deltas -> tream: "assistant"
	- lifecycle events -> stream: "lifecycle" (phase: "start" | "end" | "error")
5. agent.wait는 waitForAgentJob을 사용
	- runId를 기반하여 특정 작업의 lifecycle end나 error를 기다림
	- 반환 값 { status: "ok" | "error" | "timeout", startedAt, endedAt, error? }
- 전체 플로우
	1. 요청: PRC나 CLI로 에이전트 호출 -> 즉시 runId 받음
	2. 준비: 내부에서 AI 모델/스킬 준비하여 pi-core로 작업 넘김
	3. 실행: pi-core에서 작업 실행 후 이벤트 구독
	4. 모니터링: 이벤트 추적
	5. 완료: wait로 완료 결과 확인

## Queueing + concurrency
- 각 세션별로 작업이 순서대로 처리하여 같은 세션 안에서는 작업이 동시에 실행되지 않음
- 세션 작업은 session lane에서 관리
- 글로버 부하를 고려하여 Global lane이라는 공유 대기열이 존재하며, 여기서 전체 작업수를 제한
- **Global lane와 Session lane 덕에 도구나 세션 간 race conditions가 방지되고, 세션 히스토리가 항상 일관되게 유지**된다.
- 메시징 채널은  queue modes를 선택해 lane 시스템에 작업을 feed 가능
	- Collect: 메시지를 모아서 한 번에 처리(배치 작업, default debounceMs 1000)
	- Steer: 실시간으로 방향 조정(현재 작업 중인 작업에 새 메시지를 즉시 끼워넣음)
	- Followup: 후속 작업 우선(대화 이어가기, A 작업이 끝나고 새로운 B 작업을 수행)

## Workspace Preparation
- Workspace is resovled and created
	- Openclaw가 세션 ID나 세션 키를 바탕으로 워크스페이스를 찾거나 새로 만듦
- Skiils are loaded and injected into env and propmts
	- 스킬 로드 후 .env나 propmts에 해당 내용 주입
- Bootstrap/context files are resolved and injected into the system prompt report
	- 부트스트랩/컨텍스트 파일이 결정되고, 시스템 프롬프트 레포트에 주입
	- 프로프트 레포터: 프롬프트의 최종 버전
- A session write lock is acquired; SessionManager is opened and prepared before streaming
	- Session write lock: 다른 프로세스가 세션을 동시에 수정하지 못하게 잠금
	- SessionManager opend and prepared: 세션 객체를 로드하고, 히스토리/메타데이터 준비. 그 후 스트리밍 시작
## Prompt assembly + system prompt
- System prompt is built from OpenClaw’s base prompt, skills prompt, bootstrap context, and per-run overrides
	- 시스템 프로프트는 OpenClaw의 기본 프롬프트, 스킬 프롬프트, 부트스트랩 컨텍셔트, 작업별 오버라이드로 구축
- Model-specific limits and compaction reserve tokens are enforced
	- 모델별 제한 토큰을 지키며, 압축하여 여유롭게 조절
- See System prompt for what the model sees
	- System prompt: AI 모델이 매번 작업할 때 마다 받는 기본 지시서
	- What the model sees: 최종 조립된 시스템 프로프트로

## Hook points
- OpenClaw 프로세스 중간에 끼어들어(intercept) 로직을 추가할 수 있는 지점
- 두 가지 훅 시스템이 존재
	- Internal hooks (Gateway hooks): 이벤트 기반 스크립트
		- 게이트웨이에서 이벤트(명령 입력, 라이프사이클 변화)를 감지해 스크립트를 실행하는 훅
	- Plugin hooks: 플러그인 기반 확장 포인트 
		- 플러그인으로 에이전트 루프, 도구 라이프사이클, 게이트웨이 파이프라인을 확장하는 훅
		- 예시 훅 (일부만)
			- before_model_resolve
			- message_received

## Streaming + partial replies
- Assistant deltas are streamed from pi-agent-core and emitted as assistant events
	- Assistant deltas: 어시스턴트의 작은 변화
	- streamed from pi-agent-core: pi-agent-core에서 실시간 스트리밍
	- emitted as assistant events: 어시스턴트 이벤트로 방출
- Block streaming can emit partial replies either on text_end or message_end
	- OpenClaw는 스트리밍 시스템에서, 응답을 완성된 블록 단위로 보내는데, 그 응답을 text_end나 message_end에 보냄

## Tool execution + messaging tools
- Tool start/update/end events are emitted on the tool stream
	- 도구의 start/update/end 이벤트는 tool stream 과정에서 발생
- Tool results are sanitized for size and image payloads before logging/emitting
	- 도구 결과가 logging/emitting 전에 sanitized(정제)
- Messaging tool sends are tracked to suppress duplicate assistant confirmations
	- Messaging tool: 메시지 채널로 메시지를 보내는 툴을 의미
	- Sends are tracked: 보내는 행위를 추적
	- Suppress duplicate assistant confirmations: 불필요한 중복 확인을 억제
## Reply shaping + suppression
- Final payloads are assembled from: **assistant text** (and optional reasoning), **inline tool summaries** (when verbose + allowed), **assistant error text** when the model errors
	- Assistant text (and optional reasoning): AI 어시스턴트의 기본 텍스트 응답, 선택적 reasoning
	- Inline tool summaries (when verbose + allowed): 도구 요약을 인라인(텍스트 안에) 넣음
	- Assistant error text when the model errors: AI 모델 오류 시 오류 텍스트 추가
- NO_REPLY is treated as a silent token and filtered from outgoing payloads
	- NO_REPLY는 silent token으로 취급되어 나가는 payloads에서 제거
- Messaging tool duplicates are removed from the final payload list
	- 메시징 도구의 중복이 최종 페이로드 리스트에서 제거
- If no renderable payloads remain and a tool errored, a fallback tool error reply is emitted (unless a messaging tool already sent a user-visible reply)
	- 최소한의 응답 오류만 유저에게 보냄
## Compaction + retries
- Auto-compaction emits compaction stream events and can trigger a retry
	- Auto-compaction이 압축 스트림 이벤트를 emit하고 재시도를 trigger할 수 있음.
	- **압축 과정에서 실시간 이벤트를 스트림**으로 보내며, 압축 후 **문제가 있으면 자동 재시도**함
- On retry, in-memory buffers and tool summaries are reset to avoid duplicate output
	- 재시도 시, in-memory buffers와 tool summaries을 리셋하여 **중복 출력 방지**
- See Compaction for the compaction pipeline
	- Compaction pipeline: 압축의 단계별 흐름. 히스토리 분석 -> 요약 생성 -> 토큰 계산 -> 적용
## OpenClaw의 Event Streams (todays)
- Lifecycle: emitted by subscribeEmbeddedPiSession (and as a fallback by agentCommand)
	- lifecycle event: 에이전트의 단계별 알림
	- Emitted by subscribeEmbeddedPiSession: subscribeEmbeddedPiSession가 pi-agent-core 이벤트를 **lifecycle 스트림으로 변환해 이벤트**를 발생
	- As a fallback by agentCommand: 만약 subscribeEmbeddedPiSession가 제대로 **이벤트를 만들지 못하면 agentCommand가 end/error 이벤트를 직접 보냄**
- Assistant: streamed deltas from pi-agent-core
	- Streamed deltas: **AI 응답의 부분을 실시간**으로 보냄
	- From pi-agent-core: PI 코어에서 생성
- Tool: streamed tool events from pi-agent-core
	- Streamed tool events: **도구 실행 이벤트**
	- From pi-agent-core: PI 코어에서 생성
## Chat channel handling
-  **OpenClaw가 챗 채널에서 AI 응답을 어떻게 처리할까**에 대한 설명
- Assistant deltas are buffered into chat delta messages
	- 어시스턴트 델타가 버퍼링돼 챗 델타 메시지로 만들어짐
	- Buffered: deltas를 바로 보내지 않고, 적당한 크기로 만듦
	- Into chat delta messages: 챗 앱에 맞는 델타 메시지로 변환
- A chat final is emitted on lifecycle end/error
	- 라이프사이클 끝이나 오류 시, 최종 챗 메시지가 방출
## Timeouts
- agent.wait default: 30s
	- agent.wait --timeoutMs=6000로 config에서 변경 가능
- Agent runtime: agents.defaults.timeoutSeconds default 600s; enforced in runEmbeddedPiAgent abort timer
	- 에이전트 런타임의 기본 타임아웃은 10분
	- Agent runtime: pi-agent-core에서 작업 돌리는 전체 기간
	- agents.defaults.timeoutSeconds default 600s
	- Enforced in runEmbeddedPiAgent abort timer: runEmbeddedPiAgent: timer가 넘으면 강제 abort 오류 이벤트 발생
## Where Things Can End Early
- **OpenClaw는 안전과 효율성을 위해 여러 '중단 포인트'를 둠**
- Agent timeout (abort)
	- 에이전트 타임아웃으로 인한 abort
- AbortSignal (cancel)
	- 사용자나 시스템이 에이전트를 중간에 멈추게 하는 신호
	- AbortSignal: /stop 명령이나 API 호출로 트리거 가능(Node.js의 표준 신호)
	- Cancel: 실행 중인 에이전트 중단 (pi-agent-core에서 처리)
- Gateway disconnect or RPC timeout
	- 게이트웨이 연결 끊김이나 RPC 시간 초과
- agent.wait timeout (wait-only, does not stop agent)
	- wait만 중단되며, agent 자체는 멈추지 않음

## 연속 작업 플로우
- A 작업 (5분 소요), agent.wait 30s, 같은 세션에서 연달아 실행한다고 가정
1. **A 작업 1 제출**:
    - agent RPC 호출: 파라미터(A 작업 내용) 주고 실행 시작.
    - 즉시 반환: { runId: "1234", acceptedAt: "2026-02-22 15:25" }
    - 백그라운드: 에이전트 실행 (5분 걸림). 큐에 들어감
2. **A 작업 1에 agent.wait 걸기**:
    - agent.wait 호출: runId="1234"로 완료 기다림.
    - 기본 30초 타임아웃: 30초 후 timeout (status: timeout) 반환
    - 에이전트는 백그라운드에서 작업 계속.
3. **A 작업 2 제출** (연달아)**:
    - agent RPC 다시 호출: A 작업 2 파라미터 주고 실행.
    - 즉시 반환: { runId: "5678", acceptedAt: "2026-02-22 15:26" }.
    - 백그라운드: 큐에 들어감. 같은 세션이라 A 작업 1 끝나고 실행
4. **A 작업 2에 agent.wait 걸기**:
    - agent.wait 호출: runId="5678"로 기다림.
    - 30초 후 timeout
5. **A 작업 1 결과 리턴 및 runId 추적**:
    - A1이 5분 후 완료: lifecycle "end" 이벤트 방출 (백그라운드).
    - runId="1234"로 추적
6. **A 작업 2 실행:
	- A1 끝난 후 A2 시작
7. **A 작업 2 결과 리턴 및 runId 추적**:
	- A2이 5분 후 완료: lifecycle "end" 이벤트 방출 (백그라운드).
	- runId="5678"로 추적
