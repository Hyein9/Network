# FHRP(First Hop Redundancy Protocols)
## 1. HSRP(Hot Standby Router Protocol)  <CISCO Protocol>
  (1) 이중게이트웨이 사이에서 IP와 MAC Address를 공유 -> 무정지 경로 이중화를 제공.
  
  (2) Physical Port 동기화, VLAN Interface 동기화(L3 Switch)
  
  (3) 다수의 물리적인 router를 가상의 논리적인 router로 동기화

### ● HSRP의 구성
  - Active Router : 가상 IP Address로 오는 트래픽을 전송
  - Standby Router : Active Router 장애 발생에 대비한 백업 라우터
  - virtual Router : 실제 라우터가 아닌 HSRP 그룹 개념 가상라우터로 패킷이 전송되면 실제로 패킷을 전달하는 것은 액티브 라우터.
  - Member Router : Active, Standby 둘다 아니지만 액티브와 스탠바이를 모니터링.
    
### ● HSRP 특징
  - UDP 1985번 포트 / Multicast 224.0.0.2를 사용한다.
  - HSRP는 Layer-3 이상에서 동작을 수행한다.
  - Hello Message를 이용하여 우선순위를 결정 : Priority Value = Default 100 그룹별로 설정 가능
  - Hsrp 그룹에서 한 대의 라우터가 Active로 선출(Priority 높은 라우터, 혹은 먼저 작동된 라우터)
  - Active Router는 헬로 메시지 전송으로 Active역할을 수행 유지한다.
  - Active Router 장애시 Standby Router가 Active 역할을 수행하는데 이는 홀드타임(Holdtime)이 만료되었을 경우 발생한다.
```
Standby hello time 3 seconds Standby holdtime 10 seconds
```

### ● HSRP 상태

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

### ● HSRP 설정시 고려사항
  (1) VLAN을 통한 HSRP인지 Physical Port를 통한 hsrp인지 구분
      -> ``` interface vlan 10 / interface Gi0/0 ```
  
  (2) virtual route rG/W 
      -> ``` standby 10 ip 192.168.10.1 ```
  
  (3) VLAN G/W 
      -> ``` ip address 192.168.10.2 ```
  
  (4) 우선순위 Priority 우선순위가 같은 경우 일반적으로 IP가 높은 쪽이 Active
      -> ``` standby 10 priority 110 ```
  
  (5) Hello Dead주기 timer(Sec)
      -> ``` standby 10 timers 3 10 ```
  
  (6) hsrp group number
      -> ``` standby 10 ... ```
  
  (7) track -> ``` standby 10 track Gi0/1 20 ```
  
  (8) preempt -> ``` standby 10 preempt ```
  
  (9) authentication -> ```standby 10 authentication md5 key-string HSRPKEY ```



### R1 설정 (중앙 라우터)
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
### R2 설정 (HSRP Standby)
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

### 검증 & 디버그 전체 흐름
상태 확인
```
show standby brief
show ip interface brief
```

R3(config)#interface f0/0
R3(config-if)#shutdown


#### R2에서 기존 VIP 제거 + 새 VIP 등록
```
conf t
 interface f0/0
  no standby <HSRP_GRP> ip <OLD_VIP>
  standby <HSRP_GRP> ip <NEW_VIP>
end
```
#### R3에서 기존 VIP 제거 + 새 VIP 등록
```
conf t
 interface f0/0
  no standby <HSRP_GRP> ip <OLD_VIP>
  standby <HSRP_GRP> ip <NEW_VIP>
end
```

### PC 설정 (VPCS 기준)
```
PC1> ip <PC1_IP> <PC_MASK> <GW_VIP>
PC2> ip <PC2_IP> <PC_MASK> <GW_VIP>
```
#### 변경 후 검증
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


### 1. VLAN 인터페이스 기반 HSRP (SVI)

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
### 2. Physical Port 기반 HSRP (라우터 직결 환경)

환경 예시

인터페이스: Gi0/0

가상 게이트웨이: 10.0.0.1

```
R1
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

 R2
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

### << 상태 확인 명령어 >> (시험 + 실무 필수)
```
show standby brief
show standby
```
출력 예:
```
Vlan10 - Group 10
  State is Active
  Virtual IP address is 192.168.10.1
  Priority 110 (configured 110)
```
### 실무 꿀팁

✔ Priority 같으면 IP 높은 쪽이 Active

✔ preempt 없으면 Active 죽었다 살아나도 Standby로 남음

✔ Track 안 걸면 업링크 죽어도 Active 유지됨 (망함 😇)

✔ HSRP 인증 키 다르면 Active/Standby 절대 안 됨

# 2. VRRP(Virtual Router Redundancy Protocol) <Standard Protocol>
### 특징
(1) HSRP와 같이 이중화게이트웨잉 프로토콜
(2) 라우터 그룹을 하나의 가상 라우터로 형성 가능
(3) HSRP와 달리 표준 프로토콜, Multi-vendor 환경에서 이중화 프로토콜 사용이 가능

Master(VRRP) = Active(HSRP) 1대

Backup(VRRP) = Standby(HSRP) 여러대~

1초마다 VRRP 정보를 전송(※ Master -> Backup 단방향)

HSRP = 3초마다(양방향)
- UDP 112 번 사용
- HSRP= 1985번
- Multicast = 224.0.0.18
- HSRP Multicast = 224.0.0.2
  시리제 포트 주소를 가상 주소로 사용이 가능하다

### VRRP Master 결정조건
  - Virtual-router의 IP와 실제 Router IP가 같을 때 (IP Address Owner)
    이 경우 track옵션이 비활성되어야 한다.
  - 라우터의 우선순위 결정 값은 1-254까지
  - Priority가 높은 라우터 (Deafult = 100)
  - 인터페이스 IP주소가 높은 라우터
  - Virtual MAC address 0000.5e00.01XX
    0000.5e00.01 >> VRRP 고정 MAC 값 XX -> VRRP 그룹 번호

```
R2(config-if)#do sh vrrp
FastEthernet0/0 - Group 10
  State is Backup
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 100
  Master Router is 192.168.10.200, priority is 100
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.609 sec (expires in 3.093 sec)
```
```
R3(config-if)#do sh vrrp
FastEthernet0/0 - Group 10
  State is Master
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 100
  Master Router is 192.168.10.200 (local), priority is 100
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.609 sec
```
  - preemption 기본적으로 활성화됨. 나중에 시작했다더라도 ip가 높아지면 마스터 결정됨
  - backup(standby)정보 출력 안됨. Master에 대한 정보만 출력.
  
  - 2번라우터에 priority 높이기 ```vrrp 10 priority 150```
    잠깐 10 shutdown하면 state master -> init Wireshark에서 ```priority : 0``` 으로 변경됨.

(3) R3에 물리적 주소로 변경
```
R3(config)#int f0/0
R3(config-if)#ip add 192.168.10.1 255.255.255.0
R3(config-if)#
*Mar  1 00:22:14.739: %OSPF-5-ADJCHG: Process 3, Nbr 192.168.10.100 on FastEthernet0/0 from FULL to DOWN, Neighbor Down: Interface down or detached
R3(config-if)#
*Mar  1 00:22:14.747: %VRRP-6-STATECHANGE: Fa0/0 Grp 10 state Backup -> Disable
*Mar  1 00:22:14.747: %VRRP-6-STATECHANGE: Fa0/0 Grp 10 state Init -> Master
*Mar  1 00:22:14.767: %VRRP-6-STATECHANGE: Fa0/0 Grp 10 state Master -> Disable
*Mar  1 00:22:14.771: %VRRP-6-STATECHANGE: Fa0/0 Grp 10 state Init -> Master
```
후에 되돌림 ip add 192.168.10.200 255.255.255.0


(4) R2에 시간 변경
``` vrrp 10 timers advertise 3 ```
```
R2(config-if)#do sh vrrp
FastEthernet0/0 - Group 10
  State is Master
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 3.000 sec
  Preemption enabled
  Priority is 150
  Master Router is 192.168.10.100 (local), priority is 150
  Master Advertisement interval is 3.000 sec
  Master Down interval is 9.414 sec
```
R3은 현재 1초로 출력되지만
```
R3(config-if)#vrrp 10 timers learn
R3(config-if)#
*Mar  1 00:27:47.587: %VRRP-6-STATECHANGE: Fa0/0 Grp 10 state Master -> Backup
R3(config-if)#do sh vrrp
FastEthernet0/0 - Group 10
  State is Backup
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption disabled
  Priority is 100
  Master Router is 192.168.10.100, priority is 150
  Master Advertisement interval is 3.000 sec
  Master Down interval is 9.609 sec (expires in 7.125 sec) Learning
```

학습에 의해 변경된 것이라 learning이라고 표시됨

```
R2(config-if)#do sh vrrp
FastEthernet0/0 - Group 10
  State is Master
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 90  (cfgd 150)
    Track object 10 state Down decrement 60
  Master Router is 192.168.10.100 (local), priority is 90
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.414 sec
```
no sh 후

```
R2(config)#track 10 interface s0/0 line-protocol
R2(config-track)#int f0/0
R2(config-if)#vrrp 10 track 10 decrement 60

R2(config-track)#int s0/0
R2(config-track)#sh
```
line-protocol up -> Down & int s0/0 changed state to down

```
R2(config-if)#do sh vrrp
FastEthernet0/0 - Group 10
  State is Master
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 150
    Track object 10 state Up decrement 60
  Master Router is 192.168.10.100 (local), priority is 150
  Master Advertisement interval is 1.000 sec
  Master Down interval is 3.414 sec
```

```
R2(config-if)#vrrp 10 authentication AWS
R2(config-if)#vrrp 10 authentication md5 key-string AWS
```


