이미 사용중인chat app을 통해서 OpenClaw에게 대화를 걸 수 있다. 각각의 채널은 항상 게이트웨이를 통하며, 채널별로 컨텐츠 타입의 차이가 있다.

## Supported channels

| 채널       | 이미지 지원 여부 | 비고                        |
| -------- | --------- | ------------------------- |
| Telegram | 지원        | 비디오, 오디오, 대부분의 파일도 거의 가능  |
| WhatsApp | 지원        | Baileys 라이브러리로 이미지/미디어 가능 |
| Discord  | 지원        | 이미지 업로드/임베드 가능            |
| Slack    | 지원        | 이미지 업로드 가능                |

## 채널 공통 정보
- 채널은 동시에 실행될 수 있으며, OpenClaw가 채널 별로 라우팅한다.
- 그룹 동작은 채널 마다 다르다.
- DM 페어링 및  allowlist는 보안을 위해 강제 적용된다.

## Discord 설정법
1. Discord Developer Protal 입장
2. New Application 클릭 후 app 이름 입력
3. Bot 메뉴에서 Add Bot으로 봇 생성
4. Privileged Gateway Intents 섹션에서 필요한 설정 추가
	- Server Members Intent
	- Message Content Intent
5. Auth2 → URL Generator에서 권한 체크 -> 생성된 초대 링크로 서버에 봇 초대
6. Token 복사하여 OpenClaw 설정에 넣기
7. 보안 관련 정책 설정하기
	- ```json
		{
			"channels": {
				"discord": {
					"enabled": true,
					"token": "봇_토큰",
					"groupPolicy": "allowlist",
					"guilds": {
						"1471874173323448473": {
							"channels": {
								"1471874173834891499": {
									"allow": true
									}
								}
							}
						}
					}
			   }
		   }
		}
		```

## Discord 채널과 OpenClaw의 연결
- Gateway가 Discord 연결 전체를 관리한다.
- 답변은 항상 Discord로만 간다.
- 각 채널별로 완전 별도의 세션이다.
- 내장 슬래쉬 명령어는 별도의 세션으로 격리되지만, 원래 대화 컨텍스트는 유지된다.

## Access control and routing
### DM 정책
- pairing (승인한 사람만 허용, 코드로 직접 승인하는 방식, default)
- allowlist (허용 목록을 미리 작성)
- open (누구나 가능)
- disabled (완전 차단)
```json
"discord": {
	"dmPolicy": "pairing"	
}
```
### Guild policy (서버 접근 제어 정책)
봇이 어떤 discord 서버, 채널, 역할에서 작동할지 제한.
- open
- allowlist (허용한 목록을 미리 작성, default)
	- 명시 안 하면 초대 받은 모든 서버에서 작동
- disabled
```json
"discord": { 
	"token": "YOUR_TOKEN", 
	"guilds": {
		"1471874173323448473": { // 서버 ID 
		"allow": true, // 이 서버 허용 (false면 명시적 차단) 
		"channels": { 
			"1471874173834891499": { "allow": true }, // #general 허용
			 "987654321098765432": { "allow": false } // 다른 채널 차단 
			 }, 
		 "roles": {
			 "112233445566778899": { "allow": true } // 특정 역할만 허용 (라우팅용) 
		 } 
	   },	
	}
```
### Mentions and group DMs
- 서버 채널에서 봇이 반응하기 위해서는 @봇 멘션이 필요한지 아닌지 설정
- ```json
  "discord": { 
    "mentionGating": true // 기본 true, 
  }
  ```
- 그룹 dm에서 봇이 메시지 처리할지 여부  
- ```json
  "dm": { 
    "groupEnabled": true // false가 기본 (안전), true로 켜면 그룹 DM 작동 
  }
  ```

