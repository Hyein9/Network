# FHRP(First Hop Redundancy Protoocols)
## 1. HSRP(Hot Standby Router Protocol)  <CISCO Protocol>
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

● HSRP 상태

  **(1) Initial State**
   -  HSRP 그룹의 Active 라우터 IP를 학습하는 상태.
   - 아직 Active나 Standby가 아님.
      - 라우터는 다른 라우터로부터 Hello 패킷을 받고 가상 IP 주소를 학습함
  
  **(2) Learn State (State Code = 1)**
    - 액티브 라우터에서 헬로 메시지를 아직 수신하지 못함, 자신의 가상 IP Address를 알지 못함
  
  **(3) Listen State (Staet Code = 2)**
    - 그룹의 Active와 Standby 라우터가 이미 존재함을 알고 있음.
    = 패킷 수신만 가능하며 가상 IP를 사용하지 않음.
    - Active/Standby 선출 과정에 참여하지 않음.
  
  **(4) Speak State (State Code = 4)**
    - Hello 패킷을 주기적으로 송신하여 그룹 멤버에게 자신의 존재를 알림.
    - 그룹 내 다른 라우터와 Active/Standby 선출 과정에 참여함.
  
  **(5) Standby State (State Code = 8)**
   **- Active 라우터가 장애 발생 시 즉시 역할을 인계받을 준비 상태**
   = 현재는 가상 IP를 사용하지 않고 대기 중.
   = Active와 Standby 라우터 간 Hello 패킷을 주고받으며 상태 유지
  
  **(6) Active State (State Code = 16)**
  **- 가상 IP를 소유하고 패킷 포워딩을 수행하는 라우터.**
  = 그룹 내 최우선 라우터이며, 장애 발생 시 Standby가 대체.
  - 주기적으로 Hello 패킷 송신하여 다른 라우터에게 자신의 Active 상태 알림

● HSRP 설정시 고려사항
  (1) VLAN을 통한 HSRP인지 Physical Port를 통한 hsrp인지 구분 -> ``` interface vlan 10 / interface Gi0/0 ```
  
  (2) virtual route rG/W  -> ``` standby 10 ip 192.168.10.1 ```
  
  (3) VLAN G/W -> ``` ip address 192.168.10.2 ```
  
  (4) 우선순위 Priority 우선순위가 같은 경우 일반적으로 IP가 높은 쪽이 Active -> ``` standby 10 priority 110 ```
  
  (5) Hello Dead주기 timer(Sec) -> ``` standby 10 timers 3 10 ```
  
  (6) hsrp group number -> ``` standby 10 ... ```
  
  (7) track -> ``` standby 10 track Gi0/1 20 ```
  
  (8) preempt -> ``` standby 10 preempt ```
  
  (9) authentication -> ```standby 10 authentication md5 key-string HSRPKEY ```



### 🛠️ R1 설정 (중앙 라우터)
```
conf t
 hostname R1

 interface s0/0
  ip address <R1_R2_LINK_IP> <MASK_30>
  no shutdown

 interface s0/1
  ip address <R1_R3_LINK_IP> <MASK_30>
  no shutdown

 ip routing
end
```
### 🛠️ R2 설정 (HSRP Standby)
```
conf t
 hostname R2

 interface f0/0
  ip address <R2_LAN_IP> <PC_MASK>
  no shutdown

  standby version 2
  standby <HSRP_GROUP> ip <GW_VIP>
  standby <HSRP_GROUP> priority <R2_PRIORITY>
  standby <HSRP_GROUP> authentication md5 key-string <HSRP_KEY>
  standby <HSRP_GROUP> timers <HELLO> <HOLD>

 interface s0/0
  ip address <R2_R1_LINK_IP> <MASK_30>
  no shutdown

 ip routing
end
```

### R3 설정 (HSRP Active + Tracking + Preempt)
```
conf t
 hostname R3

 interface f0/0
  ip address <R3_LAN_IP> <PC_MASK>
  no shutdown

  standby version 2
  standby <HSRP_GROUP> ip <GW_VIP>
  standby <HSRP_GROUP> priority <R3_PRIORITY>
  standby <HSRP_GROUP> preempt
  standby <HSRP_GROUP> authentication md5 key-string <HSRP_KEY>
  standby <HSRP_GROUP> timers <HELLO> <HOLD>
  standby <HSRP_GROUP> track <TRACK_IF> <TRACK_DEC>

 interface s0/1
  ip address <R3_R1_LINK_IP> <MASK_30>
  no shutdown

 ip routing
end
```

### 🧪 검증 & 디버그 전체 흐름
✅ 상태 확인
```
show standby brief
show ip interface brief
```

R3(config)#interface f0/0
R3(config-if)#shutdown


#### ✅ R2에서 기존 VIP 제거 + 새 VIP 등록
```
conf t
 interface f0/0
  no standby <HSRP_GRP> ip <OLD_VIP>
  standby <HSRP_GRP> ip <NEW_VIP>
end
```
#### ✅ R3에서 기존 VIP 제거 + 새 VIP 등록
```
conf t
 interface f0/0
  no standby <HSRP_GRP> ip <OLD_VIP>
  standby <HSRP_GRP> ip <NEW_VIP>
end
```

### 🖥️ PC 설정 (VPCS 기준)
```
PC1> ip <PC1_IP> <PC_MASK> <GW_VIP>
PC2> ip <PC2_IP> <PC_MASK> <GW_VIP>
```
#### 9️⃣ 변경 후 검증
```
R2# show standby brief
R3# show standby brief
```
✔ Virtual IP = <NEW_VIP>
✔ Active / Standby 정상 유지
```
PC1> ping <R3_LAN_IP>
PC1> ping <R1_TO_R2_IP>
```


### ✅ 1️⃣ VLAN 인터페이스 기반 HSRP (SVI)

환경 예시

VLAN 10

가상 게이트웨이: 192.168.10.1

R1(Active), R2(Standby)
```
🔹 R1 (Active Router)
conf t
interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
 standby 10 timers 3 10
 standby 10 authentication md5 key-string HSRPKEY
 standby 10 track GigabitEthernet0/1 20
 no shutdown
end
```
```
🔹 R2 (Standby Router)
conf t
interface vlan 10
 ip address 192.168.10.3 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 100
 standby 10 preempt
 standby 10 timers 3 10
 standby 10 authentication md5 key-string HSRPKEY
 standby 10 track GigabitEthernet0/1 20
 no shutdown
end
```
### ✅ 2️⃣ Physical Port 기반 HSRP (라우터 직결 환경)

환경 예시

인터페이스: Gi0/0

가상 게이트웨이: 10.0.0.1

```
🔹 R1
conf t
interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.0
 standby 1 ip 10.0.0.1
 standby 1 priority 120
 standby 1 preempt
 standby 1 timers 1 3
 standby 1 track GigabitEthernet0/1 30
 standby 1 authentication md5 key-string HSRPKEY
 no shutdown
end
```

🔹 R2
```
conf t
interface GigabitEthernet0/0
 ip address 10.0.0.3 255.255.255.0
 standby 1 ip 10.0.0.1
 standby 1 priority 100
 standby 1 preempt
 standby 1 timers 1 3
 standby 1 track GigabitEthernet0/1 30
 no shutdown
end
```
# 2. VRRP(Virtual Router Redundancy Protocol) <Standard Protocol>

# 3. GLBP(Gateway Load Balancing Protocol)    <CISCO Protocol>

