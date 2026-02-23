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
