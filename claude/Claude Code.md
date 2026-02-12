## What is Coding Assistant 
- 언어 모델을 활용하여 복잡한 프로그래밍 작업을 처리하는 정교한 시스템이다.
- **사람 개발자가 문제에 접근하는 방식과 유사한 과정으로 작업을 처리**한다.
	![Claude Code의 문제 접근방식](images/claude/Claude_Code의_문제접근방식.png)
	1. LLM 모델이 Task(작업)을 받고 이해한다.
	2. 작업을 수행하기 위해 컨텍스트를 수집한다.
	3. 작업 계획을 수립한다.
	4. 계획을 수행한다.
	5. Task(작업)이 올바르게 수행됐는지 확인하고 아니라면 2로 돌아간다.
- 언어 모델 자체에는 파일을 읽거나 쓰고, 명령어를 실행하는 등의 작업을 수행할 능력이 없다.
- Claude Code는 **메인 에이전트와 기존 언어 모델이 수행하지 못하는 기능들을 수행할 도구들로 구성**된다.
	- Tools: ReadFile, WriteFile, RunBash, ...
- Langauge Model이 **특정 도구**를 사용하기 위해서는, 해당 도구를 사용을 의미하는 **일반 텍스트 형식으로 응답**해야 한다.
	- 이때 응답 형식과 도구의 대응 관계는 지침 형태로 Language Model에 제공된다.
- 언어 모델이 도구 사용을 요청하면 **코딩 어시스턴**트가 해당 도구가 해야 할 **실제 작업을 수행**한다.
- Claude는 작업(task)를 올바르게 수행하기 위해 사용해야 할 도구가 무엇이며, 해당 도구가 어떤 업무를 수행하는 지 잘 알고 있다.
- Claude는 **다양한 도구를 조합하여 복잡한 작업을 처리**할 수 있으며, 새로운 도구를 추가하는 것도 가능하다.

## Claude Code의 Tools

| Name      | Purpose                  |
| --------- | ------------------------ |
| **Agent** | subagent를 실행하여 특정 작업을 맡김 |
| **Bash**  | 쉘 명령어 실행                 |
| **Write** | 파일 쓰기(새 파일 생성 또는 덮어쓰기)   |
| **Edit**  | 파일 수정                    |
| **Read**  | 파일 읽기                    |
| **Glob**  | 패턴으로 파일 찾기               |
| **Grep**  | 파일 내용 검색                 |
| LS        | 파일/디렉토리 목록 보기            |
| MultiEdit | 동시에 여러 파일 수정             |
| TodoRead  | 생성된 To-Do 목록 중 하나 읽기     |
| WebFetch  | URL로 컨텐츠 가져오기            |
| WebSearch | 웹 검색                     |

## Claude Code Context
- **불필요한 컨텍스트**가 너무 많으면 클로드의 성능이 **저하되기** 때문에, 코딩 프로젝트를 진행할 때는 컨텍스트 관리가 매우 중요하다.
- 프로젝트의 컨텍스트 관리의 시작은 **/init 명령어**이다.
	- 프로젝트의 목적, 일반 아키텍처, 관련 명령어, 중요 파일을 훑고 **CLAUDE.md 파일**에 저장
- **CLAUDE.md 파일**은 프로젝트의 중요 정보뿐만 아니라 **일반적인 지침에 대한 정보를 제공**할 수 있는 최적의 위치이다.
- CLAUDE.md 파일은 3가지 종류로 분류된다.
	1. CLAUDE.md: Shared with other people -> project level
	2. CLAUDE.local.md: Not Shared with other people -> local level
	3. ~/.claude/CLAUDE.md: Used with all projects on your machine -> pc level
- **`#` 지시어**를 사용하면 프롬프트를 **CLAUDE.md 파일 중 하나를 지능적으로 업데이트**한다.
- **`@` 지시어**를 사용하면 특정 파일을 요청에 포함시킬 수 있다.

## Augment Claude's intelligence
- **Plan Mode**
	- 복잡한 작업을 시작하기 전에 작업을 단계별로 쪼개고, 작업 순서를 정하고, 위험 요소를 미리 점검함.
	- 환각 대폭 감소하며, 작업 품질 향상.
- **Thinking modes**
	- 아래로 갈수록 더 많은 추론을 위해 할당받을 수 있는 토큰의 양이 큼.
	- 1. Think
	1. Think more
	2. Think a lot
	3. Think longer
	4. Ultrathink
## Context Control
- `Escape`클로드 실행을 일시 중지하고, 응답 방향을 바꾸거나 재지시할 수 있다.
- `Escape` + `Escape` 로 특정 시점을 기준으로 **컨텍스트를 되돌**릴 수 있다.
- `/compact` 현재 대화의 모든 메시지를 가져와 **요약**한다.
- `/clear` 전체 대화 기록을 **삭제**한다.

## Custom Command
- Claude Code에는 기본 내장 명령어와 **사용자 명령어**로 구성된다.
- .claude/commands/custom_command_name.md 라는 구조로 사용자 명령어를 추가할 수 있다.
- `cusom_command_name.md`
	- 파일명(`/cusom_command_name`)으로 사용자 명령어를 호출할 수 있다.
	- 자연어로 업무에 대한 지시를 작성할 수 있다.
	- $ 기호로 매개변수를 전달할 수 있다
		- /copy_file.md 파일
		- ```md
			You are a coding agent.
			The user passed arguments:
		
			$ARGUMENTS
			Split arguments by space:
			- first argument = source file path
			- second argument = new file name			
			Copy the file and create a new file in the same directory
			with the new name.
			Do not modify original file content.
			```
		- `/copy_file src/components/Card.vue NewCard.vue`

## MCP Server
- MCP 서버를 사용하면 Cloud Code에 새로운 도구와 기능을 추가할 수 있다.
- Playwright MCP Server를 추가해보자.
	- Playwright MCP Server:  Claude Code에게 브라우저 제어 기능을 부여할 수 있다.
	- 설치: bash `claude mcp add playwright npx @playwright/mcp@latest`
	- 실행: claude 접속 후 자연어로 호출
		- Open the browser and navigate to localhost:3000
- MCP 서버는 다양한 기능을 제공하기 때문에 프로젝트에 도움이 될 수 있는 MCP 서버를 최대한 살펴보는 것이 좋다.

 