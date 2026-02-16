## 전체 개요
### Agent Runtime
- OpenClaw는 pi-mono에서 파생된 단일 임베디드 에이전트 런타임을 사용한다.
	- 단일 임베디드: OpenClaw 프로세스 내부에서 에이전트 런타임이 실행
	- 에이전트 런타임: 일반 코드 실행 런타임(LLM이 아님)
	- 에이전트 런타임이 LLM 실행, 도구 호출을 담당
	- pi-mono 에이전트 런타임을 세션 관리, 도구 연결, 워크스페이스 발견 등을 커스터마이징해서 확장함
### Workspace
- OpenClaw는 도구와 컨텍스트를 저장할 단일 작업 공간을 가지며 agents.defaults.workspace로 설정
- Workspace는 에이전트의 작업 공간이며, 여기에 메모리, 도구, 파일 등을 담는다. 여러 에이전트를 쓴다면 각자 워크스페이스를 분리
### Bootstrap files
- Workspace 내부에 편집 가능한 파일들
- 각 파일들은 에이전트 초기 컨텍스트로 첫 세션에 주입됨.
- 파일 목록
	1. AGENTS.md: 운영 지침 + 영구 지식 저장
	2. SOUL.md: 에이전트의 페르소나
	3. TOOLS.md: 도구 사용 노트
	4. BOOTSTRAP.md: 첫 실행 리추얼
	5. IDENTITY.md: 에이전트 이름, 분위기, 이모지
	6. USER.md: 사용자 프로필 + 선호 호칭
- Boostrap files는 초기 세션에만 주입됨.
- 대화 중 저장할 필요가 있는 내용은 ~/.openclaw//agents/main/sessions/.jsonl에 저장
- self-surgery 기능을 킨다면 AGENTS.md 파일만 수정 가능
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
- 스트리밍 모드 중 메시지 처리
- 


