훅을 사용하면 **Claude가 도구를 실행하기 전이나 후에 특정 작업을 실행**하게 할 수 있다. 파일 편집 후 코드 포매터 실행, 파일 변경 시 테스트 실행, 특정 파일 접근 차단 등 **자동화된 워크 플로우 구현에 유용**하다.
## How Hooks Work
![Claude_Code의_상호작용](images/claude/Claude_Code의_상호작용.png)
- Claude Code에게 질의를 하면 해당 질의가 도구 정의와 함께 Claude 모델로 전송된다.
- Claude가 도구의 사용이 필요하다고 추론하면 해당 도구의 사용을 Claude Code에게 요청한다.
- Claude Code는 해당 도구를 실행하고 결과를 다시 Claude에게 반환한다.
- **훅은 도구 사용 전후에 특정 작업을 수행하게 할 수 있게** 돕는다.
- hooks은 두 가지 종류가 있다.
	- PreToolUse hooks
	- PostToolUse hooks
## Hook Configuration
- hooks은 Claude settings file에 저장된다.
- ~/.claude/settings.json -> Global
- .claude/settings.json -> Project
- .claude/settings.local.json -> personal settings(not committed)
- json 파일에 직접 입력하거나 클로드 코드 내에서 /hooks 명령어를 통해 훅을 설정할 수 있다.
```json
{
	"hooks": {
		"PreToolUse": [
			{
				"matcher": "Read",
				"hooks": [
					{
						"type": "command",
						"command": "node /home/hooks/read_hook.ts"
					}
				]
			}
		],
		"PostToolUse": [
			{
				"matcher": "Write|Edit|MultiEdit",
				"hooks": [
					{
						"type": "command",
						"command": "node /home/hooks/edit_hook.ts"
					}
				]
			}
		],
	}
}
```
- hooks.type
	- command: 터미널 창에 직접 타이핑해서 실행할 수 있는 명령어
	- webhook(or http): 다른 시스템에 신호 보내기
	- module: 내장 도구 (스킬 or 플러그인 호출)
### PreToolUse Hooks
- PreToolUse는 도구 실행 전에 호출되기 때문에 도구의 실행을 막을 수 있음
### PostToolUse Hooks
- PostToolUse는 도구 실행 후에 호출되기 떄문에 도구의 실행을 막을 순 없음
- 후속 작업을 실행하게 할 수 있음 (코드 edit 작업 후 포메팅)
- 도구 사용에 대한 추가 피드백을 Claude에게 제공
### 실용적인 적용법
- Claude edit 후에 자동 코드 포메팅하기
- file change 후에 테스팅 수행하기
- 특정 파일을 읽거나 수정하려고 할 때 Blocking 하기
- 클로드가 접근하거나 변경한 파일에 대해 logging 하기
- Edit이나 Write 후에 코드 린터나 type checker 호출하기
- Edit이나 Write 후에 네이밍 컨벤션이나 코딩 표준 체크하기

> [!tip] 훅을 통해 내장 도구와 프로세스를 워크플로우에 통합함으로써 Claude Code의 기능을 확장

## ## Building a Hook
1. Decide on a PreToolUse or PostToolUse hook
2. Determine **which type of tool calls** you want to watch for
	- 모든 도구의 이름을 기억하기에는 무리가 있다.
	- Claude에게 직접 접근할 수 있는 도구 리스트를 반환 받을 수 있다.
		- ex) List out the names of all the tools you have access to, buillet point list;
3. Write a command that will receive the tool call
	- command 로직에는 standard input으로 호출 관련 메타데이터가 들어오며 이를 이용하여 작업을 수행할 수 있다.
	- ```json
	  { "session_id": "2d6a1e4d-6...",
	    "transcript_path": "/Users/sg/...",
	    "hook_event_name": "PreToolUse",
	    "tool_name": "Read",
	    "tool_input": { "file_path": "/code/queries/.env" } 
	  }
	  ```
	- ```javascript
	  (async function main(){
	    const chunks = [];
		
		for await (const chunk of process.stdin) {
		  chunks.push(chunk);
		}
		
		const toolArgs = JSON.parse(Buffer.concat(chunks).toString());
		
		const readPath = 
		toolArgs.tool_input?.file_path 
		|| toolArgs.tool_input?.path 
		|| "";
		
		if (readPath.includes(".env")) {
			process.exit(2); // Code 2
		}
	  })()
	  ```
	  - Exit Code 0 - Everything is fine
	  - Exit Code 2 - Block the tool call

## Gotchas around hooks
- 보안상의 이유로 반드시 **절대 경로**를 사용할 것을 강력히 권고한다.
- 하지만, 절대 경로를 적어버리면 공유할 수 없다.
- $PWD 기호와 자동 변환 스크립트를 사용하여 해결할 수 있다.

### 절대경로 공유하기
1. settings.example.json이라는 껍데기 파일을 만들고 경로 자리에 진짜 주소 대신 식별자($PWD)를 넣기
2. 자동화 스크립트 실행 (npm run setup : npm install && node ./scripts/init-claude.js)
3. 식별자($PWD) 유저 경로로 변환하기
4. settings.local.json으로 copy 하기
- ./script/init-claude.js
```javascript
const pwd = process.cwd();

const templatePath = path.join(".claude", "settings.example.json");
const outputPath = path.join(".claude", "settings.local.json");

const templateContent = fs.readFileSync(templatePath, "utf8");
const processedContent = templateContent.replace(/\$PWD/g, pwd);

JSON.parse(processedContent);
const claudeDir = path.dirname(outputPath);

if (!fs.existsSync(claudeDir)) {
  fs.mkdirSync(claudeDir, { recursive: true });
}

// Write the processed content to settings.json
fs.writeFileSync(outputPath, processedContent, "utf8");
```

## Useful hooks!
- 훅을 사용하면 AI-assisted development의 **약점을 개선**할 수 있다.
- 훅은 Claude가 코드를 수정할 때 자동으로 실행되어 즉각적인 피드백을 제공한다.

### TypeScript Type Checking hook
- Claude가 function signiture를 수정할 때, 프로젝트 전체에서 해당 함수가 호출되는 모든 파일을 업데이트하지 않는 경우가 있다.
- Edit의 PostToolUse 훅으로 tsc --noEmit의 실행 결과를 다시 클로드에게 전달하고 수정하도록 하는 것은 매우 유용한 hook 사용법 중 하나이다.
 - ```json
   "PostToolUse": [
     "matcher":"Edit",
     "hooks":[
       {
         "type":"command",
         "command": "node ./hooks/tsc.js"
       }
     ]
   ]
   ```
- /hooks/tsc.js 로직
	1. stdin 데이터 json으로 변환하기
	2. tsconfig.json 파일 읽고, 프로그램에 맞춰 parse 하기
	3. 실행 프로그램 생성
	4. 실행 결과 가져오기
	5. 결과 확인 후 필요하다면 결과 stdout로 보내며, Code 2를 Exit하여 클로드에게 알리기

## Another useful hook
- PreToolUse, PostToolUse 이외에도 유용한 여러 hook이 존재한다.
- Notification
	- Claude needs a permission to use a tool
	- 비활성 상태 60초 유지시
- Stop
	- 클로드 코드가 응답을 완료했을 때 실행됨
- SubagentStop
	- 서브 에이전트가 작업을 끝냈을 때
- PreCompact
	- 컴팩트 작업이 발생하기 전에
- UserPromptSubmit
	- 유저가 프롬프트를 제출했을 때
- SessionStart
	- 세션이 시작될 때
- SessionEnd
	- 세션이 종료될 때

> [!note] 호출되는 훅의 종류에 따라 명령어에 대한 표 입력이 다르다!
> 
- TodoWrite에 대한 PostToolUse
```json
{ "session_id": "9ecf22fa-edf8-4332-ae85-b6d5456eda64",
  "transcript_path": "<path_to_transcript>",
  "hook_event_name": "PostToolUse", 
  "tool_name": "TodoWrite",
  "tool_input": 
    { "todos": 
      [{ "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }] 
    },
    
  "tool_response": { 
    "oldTodos": [], 
    "newTodos": 
      [{ "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }] 
  } 
}
```
- TodoWrite에 대한 Stop
```json
{  
	"session_id": "af9f50b6-f042-4773-b3e2-c3a4814765ce",
	"transcript_path": "<path_to_transcript>",
	"hook_event_name": "Stop",
	"stop_hook_active": false 
}
```
- 사용하는 훅과 매처에 따라 표준 입력이 달라지기 때문에 훅 작성이 까다로울 수 있다.
- 아래 헬퍼를 사용하여 특정 훅이 어떤 형식의 표준입력을 받는지 확인할 수 있다.
```json
"PostToolUse": [ // Or "PreToolUse" or "Stop", etc 
  { 
    "matcher": "*",
    "hooks": [ 
	  { 
	    "type": "command", 
	    "command": "jq . > post-log.json" 
	  } 
    ],
  }, 
]
```
- 위 명령어는 이 후크에 입력된 내용을 post-log.json 파일에 기록하기 때문에 어떤 표준입력이 들어오는지 눈으로 확인할 수 있음.

## Claude Code SDK
- Cladue Code SDK를 사용하면 자체 어플리케이션 및 스크립트 내에서 Cladue Code를 프로그래밍 방식으로 실행할 수 있다.
	- TypeScript, Python, CLI를 통해 제공
- 터미널에서 사용하는 것과 동일한 Cladue Code 기능을 더 큰 워크플로에 통합하여 사용할 수 있다.
### Basic Usage Claude Code SDK
```json
import { query } from "@anthropic-ai/claude-code";

const prompt = "질문 내용"

for await (const message of query({
	propmt,
	options: { 
		allowedTools: ["Edit"] // <- 허용할 도구 
	} 
})) {	
	console.log(JSON.stringify(message, null, 2));

}
```
- 기본적으로 Read만 허용하기 때문에 options으로 허용 도구를 설정
- 또는 .claude 디렉토리 내 설정 파일에서 프로젝트 전체 접근 권한을 구성할 수 있음
### 실질적인 사용 예시
- git hooks that automatically review code changes
- Build scripts that analyze and optimize code
- Helper commands for code maintenance tasks
	- run lint, run test, ... 등의 일련의 작업을 수행하고 결과를 받아 다시 edit 하는 식으로 가능.
- Automated documentation generation
- Code quality checks in CI/CD pipelines
