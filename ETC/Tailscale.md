
## Zero-Trust Mesh VPN
- WireGuard(와이어가드) 프로토콜을 기반하여 작동하며, 중앙 서버에 트래픽이 집중되는 방식이 아니라 각 노드가 Peer-to-Peer 방식으로 직접 연결된느 오버레이 네트워크이다.
### Nat Traversal (NAT 순회 및 홀펀칭) 
- STUN/TURN 서버를 활용하여 UDP 홀펀칭을 수행함
	- 복잡한 라우터 환경이나 엄격한 방화벽 뒤에 숨어 있는 기기들간의 **직접 통신을 성사**
### Identity-Based Authentication (신원 기반 인증)
- 네트워크 자체에서 비밀번호를 발급하지 않고, 기존의 SSO 제공자(Google, Apple 등)의 OIDC 인증을 그대로 차용
	- 새로운 아이디나 복잡한 VPN 전용 비밀번호 없이, 평소에 사용하던 **구글 아이디로 앱에서 로그인만하면 전용 네트워크에 즉시 편입**
### MagicDNS (자동 도메인 이름 시스템)
- 노드가 네트워크에 참여할 때마다 Tailscale 컨트롤 플레인이 자동으로 **기기명 기반의 DNS 레코드를 생성**하고 배포. IP 주소 대신 **hostname.domain.ts.net 형태로 노드에 직접 접근** 
### Subnet Routing (서브넷 라우팅)
- Tailsclae이 설치된 단일 노드를 게이트웨이로 활용하여, 해당 기기가 속한 로컬 물리 네트워크 전체의 트래픽을 라우팅할 수 있음