
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
  (1) Initial State
  (2) Learn State (State Code = 1)
  액티브 라우터에서 헬로 메시지를 아직 수신하지 못함, 자신의 가상 IP Address를 알지 못함
  (3) Listen State (Staet Code = 2)
  (4) Speak State (State Code = 4)
  (5) Standby State (State Code = 8)
  (6) Active State
  
● HSRP 설정시 고려사항
  (1) VLAN을 통한 HSRP인지 Physical Port를 통한 hsrp인지 구분
  (2) virtual route rG/W
  (3) VLAN G/W
  (4) 우선순위 Priority 우선순위가 같은 경우 일반적으로 IP가 높은 쪽이 Active
  (5) Hello Dead주기 timer(Sec)
  (6) hsrp group number
  (7) track
  (8) preempt
  (9) authentication


## FHRP(First Hop Redundancy Protoocols) 실
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
# 2. VRRP(Virtual Router Redundancy Protocol) <Standard Protocol>

# 3. GLBP(Gateway Load Balancing Protocol)    <CISCO Protocol>

