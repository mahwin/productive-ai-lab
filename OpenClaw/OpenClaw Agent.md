## 전체 개요
### Agent Runtime
- OpenClaw는 pi-mono에서 파생된 **단일 임베디드 에이전트 런타임**을 사용한다.
	- 단일 임베디드: OpenClaw 프로세스 내부에서 에이전트 런타임이 실행
	- 에이전트 런타임: 일반 코드 실행 런타임(LLM이 아님)
	- 에이전트 런타임이 LLM 실행, 도구 호출을 담당
	- pi-mono 에이전트 런타임을 세션 관리, 도구 연결, 워크스페이스 발견 등을 커스터마이징해서 확장함
### Workspace
- OpenClaw는 도구와 컨텍스트를 저장할 단일 작업 공간을 가지며 agents.defaults.workspace로 설정
- **Workspace는 에이전트의 작업 공간**이며, 여기에 메모리, 도구, 파일 등을 담는다. 여러 에이전트를 쓴다면 각자 워크스페이스를 분리
	- 단일 에이전트의 workspace 경로 설정
		- ```json
		  {
		    "agent": {
		      "defaults": {
		        "workspace": "~/my-projects/my-workspace" // <- 워크스페이스 경로 설정
		      }
		    }
		  }
		  ```
	- 멀티 에이전트의 workspace 경로 설정
		- ```json
		  {
		    "agents": {
		      "list": [
		        { "id": "home", "workspace": "~/claw-home" }
		        { "id": "work", "workspace": "~/claw-work" }
		      ]
		    }
		  }
		  ```
### Bootstrap files
- Workspace 내부에 편집 가능한 파일들
- 각 파일들은 **에이전트 초기 컨텍스트**로 세션을 시작하면 자동으로 주입.
- 파일 목록
	1. AGENTS.md: 운영 지침 + 영구 지식 저장
	2. SOUL.md: 에이전트의 페르소나
	3. TOOLS.md: 도구 사용 노트
	4. BOOTSTRAP.md: 첫 실행 리추얼
	5. IDENTITY.md: 에이전트 이름, 분위기, 이모지
	6. USER.md: 사용자 프로필 + 선호 호칭
- BOOTSTRAP.md는 워크스페이스 처음 만들 때만 사용하고 자동으로 삭제됨.

### Memory
- OpenClaw는 작업 내용이나 맥락, 도구들을 파일 형식으로 관리.
- 컨텐츠에 따라 다른 위치에 저장
```bash

~/.openclaw/workspace/
├── AGENTS.md, SOUL.md, ... ← 부트스트랩 file
├── MEMORY.md ← 장기 기억 (당신이 직접 정리하거나 에이전트가 요약해서 저장)
│
├── memory/ ← 가장 많이 쓰이는 저장소 (일일 자동 저장) 
	├── 2026-02-20.md │ 
	├── 2026-02-21.md ← 오늘 내용 append-only로 계속 쌓임
├── skills/ ← 워크스페이스 전용 스킬 

~/.openclaw/agents/default/sessions/
├── sessions.json ← 세션 목록 인덱스 (모든 세션 정보) 
└── 2026-02-21-095812-main.jsonl ← 실제 세션 기록 파일 (JSON Lines 형식)

```

| 저장 내용        | 저장되는 곳                                 | 특징                                  |
| ------------ | --------------------------------------- | ----------------------------------- |
| 일일 대화, 작업 로그 | memory/YYYY-MM-DD.md                    | 자동 append, 매 세션 시작 시 오늘 + 어제 읽음     |
| 장기 기억, 중요 사실 | MEMORY.md (워크스페이스 루트)                   | 에이전트가 스스로 요약해서 저장, private 세션에서만 주입 |
| 새 프로젝트 파일    | Workspace 루트 또는 하위 폴더                   | 에이전트가 write_file 도구로 직접 생성          |
| 새 스킬         | skills/폴더                               | 워크스페이스 전용 스킬 우선 적용                  |
| 세션 전체 기록     | ~/.openclaw/agens/`<agentId>`/sessions/ | JSONL 또는 md 형태                      |

### Session
- OpenClaw는 에이전트와의 하나의 연속된 대화 맥락을 세션이라고 함.
- OpenClaw는 채팅방.그룹,토픽마다 별도의 세션을 만들어서 기억을 분리함

```json

~/.openclaw/agents/<agent_id>/sessions/
├── sessions.json ← 세션 목록 인덱스 (모든 세션 정보) 
└── 2026-02-21-095812-main.jsonl ← 실제 세션 기록 파일 (JSON Lines 형식)

```

| 상황             | 세션 키                               | 저장 파일                        |
| -------------- | ---------------------------------- | ---------------------------- |
| 1:1 WhatsApp   | agent:default:main                 | 2026-02-21-095812-main.jsonl |
| 회사 Telegram 그룹 | agent:default:telegram:group:12345 | telegram-group-12345.jsonl   |
| Telegram 토픽    | agent:default:teletram:topic:678   | ...-topic-678.jsonl          |
| Discord 특정 채널  | agent:default:discord:channel:abc  | discord-channel-abc.jsonl    |

### Built-in-tool
- 파일 읽기/쓰기/실행/편집과 같은 core tool은 항상 사용 가능
- apply_patch와 같은 패치 전용 도구는 tools.exec.applyPatch을 켜야함
- TOOLS.md는 도구 존재/활성화가 아닌 사용에 대한 지침만 전달

### Skills
- Skills을 추가하여 실제 작업을 실행할 수 있음.
- Skills 저장 위치
	1. 
	2. 번들 Skill `~/.openclaw/skills/ ` -> npm install 시에 자동으로 ~/.openclaw/skills/ 하위로 스킬이 옮겨짐
	3. 로컬 : `~/.openclaw/skills/`
	4. 워크스페이스 `<workspace>/skills/`

### Steering while streaming
- Streaming -> LLM 모델에서 답변이 실시간으로 도착하는 상태
- Steering -> 사용자의 추가 메시지
- 큐 모드와 Block streaming으로 구성됨
- 큐 모드 내에는 steer 모드와 followup(=collect)이 있다.
	1. steer: 도구 호출마다 queue 확인. queue에 msg가 있으면 작업 Skipped하고 해당 내용 포함해서 다시 작업을 수행함
	2. followup or collect: 작업이 완전히 끝날 때까지 기다린 후 새 메시지를 모아서 다음 작업 턴에 넘김

### Model refs
- agents.defaults.model에 저장되며 **provider/model-id** 형식으로 저장
- ~/.openclaw/openclaw.json 설정 파일에 저장
	- ```json
	  {
	    "agents": {
	      "defaults": {
	        "model": {
	          "primary": "anthropic/claude-3-5-sonnet-20241022",
	          "fallbacks": ["openai/gpt-4o", "ollama/llama3.3"]      
	        }
	      }
	    }
	  }
	  ```
- 관련 명렁어
	- 채팅 중
		1. /model list <- 사용 가능한 모델 보기
		2. /model status <- 현재 사용중인 모델 보기
		3. /model anthropic/claude-3-5-sonnet-20241022 <- 모델 바꾸기
	- CLI
		1. openclaw models list <- 사용 가능한 모델 보기
		2. openclaw models set ollama/llama3.3 <- 모델 바꾸기
