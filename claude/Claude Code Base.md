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
- Langauge Model이 **특정 도구**를 사용하기 위해서는, 해당 도구 사용을 의미하는 **일반 텍스트 형식으로 요청**해야 한다.
	- 이때 요청 형식과 도구의 대응 관계는 지침 형태로 Language Model에 제공된다.
- 언어 모델은 사용자 입력과 도구 정의를 받고, 도구 사용이 필요하다면 tool_use 블록을 생성해 도구를 호출한다.
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
	1. **CLAUDE.md**: Shared with other people -> **project level**
	2. **CLAUDE.local.md**: Not Shared with other people -> **local level**
	3. **~/.claude/CLAUDE.md**: Used with all projects on your machine -> **pc level**
## Augment Claude's intelligence
- **Plan Mode**
	- 복잡한 작업을 시작하기 전에 작업을 단계별로 쪼개고, 작업 순서를 정하고, 위험 요소를 미리 점검함.
	- 환각 대폭 감소하며, 작업 품질 향상.
- **Thinking modes**
	- 아래로 갈수록 더 많은 추론을 위해 할당받을 수 있는 토큰의 양이 큼.
	1. Think
	2. Think more
	3. Think a lot
	4. Think longer
	5. Ultrathink
## Context Control
- `Escape`클로드 실행을 일시 중지하고, 응답 방향을 바꾸거나 재지시할 수 있다.
- `Escape` + `Escape` 로 특정 시점을 기준으로 **컨텍스트를 되돌**릴 수 있다.
- `/compact` 현재 대화의 모든 메시지를 가져와 **요약**한다.
- `/clear` 전체 대화 기록을 **삭제**한다.
## Custom Command
- Claude Code의 명령어는 **기본 내장 명령어**와 **사용자 명령어**로 구성된다.
- .claude/commands/copy_file.md 라는 구조로 사용자 명령어를 추가할 수 있다.
- `cusom_command_name.md`
	- `/` + 파일명(copy_file)으로 사용자 명령어를 호출할 수 있다.
	- 자연어로 업무에 대한 지시를 작성할 수 있다.
	- $ 기호로 매개변수를 전달할 수 있다
		- ```markdown
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

## MCP Server
- **MCP 서버를 사용하면 Claud Code에 새로운 도구와 기능**을 추가할 수 있다.
- MCP 서버는 원격이나 로컬 컴퓨터에서 실행된다.
- Playwright MCP Server를 추가해보자.
	- Playwright MCP Server:  Claude Code에게 **브라우저 제어 기능**을 부여할 수 있다.
	- 설치: bash `claude mcp add playwright npx @playwright/mcp@latest`
	- 실행: claude 접속 후 자연어로 호출
		- Open the browser and navigate to localhost:3000
	- 권한: Claude는 매번 권한을 요청하기 때문에 필요하다면 권한을 부여할 수 있음.
		- ```json
		  { "permissions": { "allow": ["mcp__playwright"], "deny": [] } }
		  ```
- 특정 개발 **요구사항에 부합하는 MCP 서버**를 통해 단순한 코드 보조 도구에서 **전체 툴체인과 상호작용할 수 있는 포괄적인 개발 파트너**로 전환활 수 있다. 

## Github integration
- Claude Code는 Github Actions 내에서 Claude를 실행할 수 있도록 공식 GitHub 통합 기능을 제공
- Issue 나 PR에 대한 멘션 지원(@claude)과 PR auto review라는 두 가지 주요 워크플로우를 제공함

### integration install
- `/install-github-app` in Claude를 호출하면 아래 설정 과정을 단계별로 안내함
	1. GitHub에 Claude Code 앱 설치하기
	2. API 키 추가하기
	3. workflow 파일이 포함된 PR 자동 생성하기
- 이렇게 생성된 풀 리퀘스트는 깃허브 저장소에 두 개의 **GitHub Actions**를 추가해 준다.
	- Mention Action
	- Pull Request Action
- 개발자는 merged만 하면 됨.

## Default GitHub Actions
### Mention Action
- any issue or pull request using @cluade
- When mentioned, Claude will
	1. 요청을 분석하고 작업 계획을 짠다.
	2. codebase에 대한 완전한 접근 권한으로 작업을 수행한다.
	3. Issue나 PR에 직접 결과를 반환한다.
### Pull Request Action
- Whenever you create a pull request, Claude automatically
	1. 변경에 대한 리뷰
	2. 변화에 대한 영향도 분석
	3. PR에 대한 자세한 보고서 작성
## Customizing the Workflows
- 최초의 pr 이후에 workflow 파일들을 내 프로젝트 요구에 맞출 수 있다.
- 프로젝트 셋팅 방법과 프로젝트 셋팅과 관련된 컨텍스트 주입하는 예시
```markdown
name: Project Setup
run: |
	npm run setup
	npm run dev:daemon
cutom_instructions: |
	The project is already set up with all dependencies installed. The server is already running at localhost:3000. Logs from it are being written to logs.txt. If needed, you can query the db with the 'sqlite3' cli. If needed, use the mcp__playwright set of tools to launch a browser and interact with the app

```
- mcp server configuration도 전달하여 클로드가 이용하게 할 수 있음
```markdown
mcp_config: |
	{ "mcpServers": 
		{ "playwright": { 
			"command": "npx",
			"args": [ 
				"@playwright/mcp@latest", 
				"--allowed-origins", 
				"localhost:3000;cdn.tailwindcss.com;esm.sh" 
				] 
			} 
		} 
	}

```
- 허용된 모든 도구를 명시적으로 내열하여 전달할 수 있음
```markdown
allowed_tools:
"Bash(npm:*),
 Bash(sqlite3:*),
 mcp__playwright__browser_snapshot,
 mcp__playwright__browser_click,..."
```
- 단, GitHub는 세부 도구들의 이름을 설정 파일에 하나하나 전부 다 적어서 명시적으로 권한을 줘야함
- playwright를 연결했다고 끝이 아니라 웹페이지 열기, 버튼 클릭하기, 스크린샷 찍기 같은 세부 도구들의 이름을 설정 파일에 하나하나 적어야 함.

## Github integration Best Practices
- start with the default workflows and **customize gradually**
- use custom **instructions to provide project-specific context**
- **be explicit about toll permissions** when using MCP servers
- Test your workflows with simple tasks before complex ones