## Components and flows
### Gateway daemon
- **채널과의 연결을 계속 유지**하고 관리
- 타입 정의된 WebSocket API를 외부에 제공
- 입력 값 json 형식 검증
- 다양한 **이벤트들을 방출**함
- 크게 두가지 연결 방식이 존재
	1. 내부 WS API(Port: 18789)로 Cli, mac app, web dashboard를 관리
	2. 채널 연결로 메시징 플랫폼별 adapter를 사용하여 유지  
### Clients
- Client: OpenClaw를 제어하거나 확인하는 도구들
	- mac app, Cli command, web dashboard
- 모든 Client는 각각의 **독립적인 WS로 gateway와 연결**된다.
- 게이트웨이로 requests를 보냄
	- health, send, status, agent, system-present
-  게이트웨이의 event를 구독
	- tick, agent, shutdown, present

### Nodes
- OpenClaw를 실행하는 **추가 기기나 노드**들
- Node로는 앱 형태(mac OS, ios, android app)나 화면 없는 백그라운드(headless) sw가 있음
- 연결에는 장치 고유 id를 사용하며, 한번 approve하면 영구 저장한 후 사용
- Node는 게이트웨이에 명령어 셋을 expose하고, 게이트웨이는 명령어 셋을 등록한 후에 agent가 사용할 수 있게 함
### WebChat
- dashboard의 채팅 기능
- static UI로 가벼움
- 로컬에서는 바로 접근할 수 있으며, remote에서는 ssh로 연결

## Wire Protocol
- Transport : 웹 소켓 기반이며 ,텍스트 기반 JSON 형식을 따름
- 첫 요청은 반드시 connect여야 함
- After handshake
	- Requests: 
		- `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
	- Events:
		- {type:"event", event, payload, seq?, stateVersion?}
- OPENCLAW_GATEWAY_TOKEN (or --token)이 설정되어 있으면 반드시 연결할 때 auth.token 맞춰야함
- 중요한 명령은 idempotency key를 붙여 안전하게 retry할 수 있게 함
- Node는 연결할 때 role:node와 자신의 기능 목록을 함께 보내야 함

## Pairing + local trust
- 모든 WS 클라이언트는 connect 요청에 device identity를 포함
- 새로운 id를 가진 장비는 승인을 반드시 받아야 함
- Local 연결은 자동 승인
- Non-Local(외부/로컬) 연결은 gateway가 주는 challenge nonce(임시 난수)에 서명하고 운영자가 직접 approve해야 함.
- gateway 인증은 설정되어 있다면 Local, Non-Local 상관없이 무조건 인증 필요
## Remote access
- Preferred: Tailscale or VPN.
- Alternative: SSH tunnel
	- ```bash
	  ssh -N -L 18789:127.0.0.1:18789 user@host
	  #내 pc의 18789를 user@host라는 pc의 18789 포트로 연결하겠다.
	  ```
	- 로컬 PC의 SSH 공개키를 원격 PC가 갖고 있어야 함.