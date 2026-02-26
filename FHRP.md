## FHRP(First Hop Redundancy Protoocols)
# 1. HSRP(Hot Standby Router Protocol)  <CISCO Protocol>
  (1) 이중게이트웨이 사이에서 IP와 MAC Address를 공유 -> 무정지 경로 이중화를 제공.
  (2) Physical Port 동기화, VLAN Interface 동기화(L3 Switch)
  (3) 다수의 물리적인 router를 가상의 논리적인 router로 동기화

● HSRP의 구성
  - Active Router : 가상 IP Address로 오는 트래픽을 전송
  - Standby Router : Active Router 장애 발생에 대비한 백업 라우터
  - virtual Router : 실제 라우터가 아닌 HSRP 그룹 개념 가상라우터로 패킷이 전송되면 실제로 패킷을 전달하는 것은 액티브 라우터.
  - Member Router : Active, Standby 둘다 아니지만 액티브와 스탠바이를 모니터링.
● HSRP 특징
  - UDP 1985번 포트 / Multicast 224.0.0.2를 사용한다.
  - HSRP는 Layer-3 이상에서 동작을 수행한다.
  - Hello Message를 이용하여 우선순위를 결정 : Priority Value = Default 100 그룹별로 설정 가능
  - Hsrp 그룹에서 한 대의 라우터가 Active로 선출(Priority 높은 라우터, 혹은 먼저 작동된 라우터)
  - Active Router는 헬로 메시지 전송으로 Active역할을 수행 유지한다.
  - Active Router 장애시 Standby Router가 Active 역할을 수행하는데 이는 홀드타임(Holdtime)이 만료되었을 경우 발생한다.
```
Standby hello time 3 seconds Standby holdtime 10 seconds
```
# 2. VRRP(Virtual Router Redundancy Protocol) <Standard Protocol>

# 3. GLBP(Gateway Load Balancing Protocol)    <CISCO Protocol>

