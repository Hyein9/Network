▶︎ Disabled : 포트가 고장나서 사용할 수 없거나 네트워크 관리자가 포트를 일부러 Shut down 시켜놓은 상태
∙ 데이터 전송 불가능
∙ 맥 어드레스 러닝 불가능
∙ BPDU 송수신 불가능


▶︎ Blocking : 스위치를 맨 처음 켜거나 Disabled 되어 있는 포트를 관리자가 다시 살린 상태.
              이 상태에서는 데이터 전송은 불가능하고 오직 BPDU만 송수신이 가능하다. 스패닝 트리의 3가지 선정 과정이 이 블로킹 단계에서 이뤄진다.
∙ 데이터 전송 불가능
∙ 맥 어드레스 러닝 불가능
∙ BPDU 송수신
- 0초 or 20초


▶︎ Listening : 블로킹 상태에 있던 스위치 포트가 루트포트나 데지그네이티드 포트로 선정되면 포트는 바로 리스닝 상태로 넘어간다.
               리스닝 상태에 있던 포트도 상황에 따라 다시 Non Designated 포트로 변할 수 있고 그러면 다시 블로킹 상태로 돌아간다.
∙ 데이터 전송 불가능
∙ 맥 어드레스 러닝 불가능
∙ BPDU 송수신


▶︎ Learning : 리스닝 상태에 있던 스위치 포트가 포워딩 딜레이 디폴트 시간인 15초 동안 계속 그 상태를 유지하면 리스닝 상태는 러닝 상태로 넘어간다.
             이 상태에서야 비로소 맥 어드레스를 배워 맥 어드레스 테이블을 만들기 시작할 수 있다.
∙ 데이터 전송 불가능
∙ 맥 어드레스 러닝
∙ BPDU 송수신


▶︎ Forwarding : 스위치 포트가 러닝 상태에서 다른 상태로 넘어가지 않고 다시 포워딩 딜레이 디폴트 시간인 15초동안 유지하면 러닝 상태에서 포워딩 상태로 넘어가게 된다.
               포워딩 상태가 되어야 스위치 포트는 드디어 데이터 프레임을 주고받을 수 있게 된다.
∙ 데이터 전송
∙ 맥 어드레스 러닝
∙ BPDU 송수신

Disabled -> Blocking(20초) -> Listening(15초) -> Learning (15초) -> Forwarding



## 스패닝트리 프로토콜 종류
#### 1. CST (Common Spanning Tree)
- 물리적인 네트워크 연결을 기반으로 하나의 Port를 Block 시켜 루프를 방지
 
#### 2. PVST (Per Vlan Spanning Tree)
- 기존 STP의 확장판으로 VLAN 별로 하나의 Port를 Block 시킴.
- 시스코 전용 프로토콜의 프로토콜로 CST와 호환되지않음.

#### 3. PVST+
- PVST의 확장판
- 802.1q 트렁크 방식 지원
- 시스코 전용, CST와 호환


#### 4. RSTP
- STP의 진화형으로 빠른 수렴 속도를 가짐


#### 5. PVRST
- 시스코에서 만든 RSTP의 확장판


#### 6. MSTP (Multiple Spanning Tree Protocol)
- 여러개의 VLAN을 그룹으로 묶어 STP를 동작시킴
- VLAN의 개수가 많을수록 비례하여 일반적인 PVST는 BPDU를 전송하기 때문에 트래픽의 부하가 발생.
  MSTP는 VLAN을 그룹으로 묶어 BPDU를 보내기 때문에 트래픽의 부하가 적음!

### STP 스패닝 트리 프로토콜
- 스위치나 브리지에서 루핑을 막아주는 프로토콜
- 스위치나 브리지 구성에서 출발지로부터 목적지까지의 
```
SW1#sh spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0004.9A66.7111
             Cost        19
             Port        2(FastEthernet0/2)
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)  *Priority value is changeable*
             Address     0004.9A76.8182
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Fa0/2            Root FWD 19        128.2    P2p
```

```
SW1#sh version
Cisco Internetwork Operating System Software
IOS (tm) C2950 Software (C2950-I6Q4L2-M), Version 12.1(22)EA4, RELEASE SOFTWARE(fc1)
Copyright (c) 1986-2005 by cisco Systems, Inc.
Compiled Wed 18-May-05 22:31 by jharirba
Image text-base: 0x80010000, data-base: 0x80562000

ROM: Bootstrap program is is C2950 boot loader
Switch uptime is 14 minutes, 7 seconds
System returned to ROM by power-on

Cisco WS-C2950-24 (RC32300) processor (revision C0) with 21039K bytes of memory.
Processor board ID FHK0610Z0WC
Last reset from system-reset
Running Standard Image
24 FastEthernet/IEEE 802.3 interface(s)

63488K bytes of flash-simulated non-volatile configuration memory.
Base ethernet MAC Address: 0004.9A76.8182
Motherboard assembly number: 73-5781-09 
Power supply part number: 34-0965-01
Motherboard serial number: FOC061004SZ
Power supply serial number: DAB0609127D
Model revision number: C0
Motherboard revision number: A0
Model number: WS-C2950-24
System serial number: FHK0610Z0WC
Configuration register is 0xF
```

Root bridge 선출
- 상대방과 bridge id 비교해서 낮은 값을 root bridge 선출
- 스위치들은 root id가 root bridge의 id값으로 변경
- root bridge가 선출되면 다른 스위치들은 매 2초마다 bpdu 수신
```
Switch(config)#spanning-tree vlan 1 root primary 

Switch#sh spanning-tree 
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    24577 변경됨
             Address     0090.21D3.D807
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)
             Address     0090.21D3.D807
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Fa0/2            Desg LRN 19        128.2    P2p
```
- 블락포트가 3번스위치로 밀려남 -
```
SW3(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    20481
             Address     0004.9A66.7111
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    20481  (priority 20480 sys-id-ext 1)
             Address     0004.9A66.7111
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg LSN 19        128.2    P2p
Fa0/1            Desg FWD 19        128.1    P2p
```
```
SW1(config)#spanning-tree vlan 1 root primary 
SW1(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    16385
             Address     0004.9A76.8182
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    16385  (priority 16384 sys-id-ext 1)
             Address     0004.9A76.8182
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg LSN 19        128.1    P2p
Fa0/2            Desg FWD 19        128.2    P2p
```
```
Switch(config)#spanning-tree vlan 1 root primary 
Switch(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    12289
             Address     0090.21D3.D807
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    12289  (priority 12288 sys-id-ext 1)
             Address     0090.21D3.D807
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Fa0/2            Desg LSN 19        128.2    P2p
```
```
SW3(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    8193
             Address     0004.9A66.7111
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    8193  (priority 8192 sys-id-ext 1)
             Address     0004.9A66.7111
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg LSN 19        128.2    P2p
Fa0/1            Desg FWD 19        128.1    P2p
```
```
SW1(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    4097
             Address     0004.9A76.8182
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4097  (priority 4096 sys-id-ext 1)
             Address     0004.9A76.8182
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg LSN 19        128.1    P2p
Fa0/2            Desg FWD 19        128.2    P2p
```
```
Switch(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    1
             Address     0090.21D3.D807
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    1  (priority 0 sys-id-ext 1)
             Address     0090.21D3.D807
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Fa0/2            Desg LSN 19        128.2    P2p
```
primary 값이 
4096의 배수만큼 돌면서 관리자가 직접 static으로 지정가능하고
가장 낮은 값으로 바뀔 것이고 가장 높은 값과 낮은값 사이 계산해서 작게 설정 이후에 0이 될때까지 사용가능하나 나머지가 동일하게 0이 되면 

리셋 후 
```
Switch(config)#spanning-tree vlan 1 root secondary 
Switch(config)#do sh spa
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    28673
             Address     0090.21D3.D807
             This bridge is the root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    28673  (priority 28672 sys-id-ext 1)
             Address     0090.21D3.D807
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/1            Desg FWD 19        128.1    P2p
Fa0/2            Desg FWD 19        128.2    P2p
```



## 🌳 STP / RSTP 포트 역할 3종 세트

(기본 STP: IEEE 802.1D,
빠른 버전: IEEE 802.1w (RSTP))

✅ Root Port (RP)

📍 Root Bridge로 가는 최단 경로 포트
각 일반 스위치마다 1개만 존재
Root Bridge 방향으로 트래픽 보내는 포트

기준:
1️⃣ Root까지 Path Cost 가장 낮음
2️⃣ 상대 Bridge ID
3️⃣ 상대 Port ID

👉 쉽게 말해:
“내가 루트한테 가는 베스트 길”

✅ Designated Port (DP)

🏆 해당 네트워크 구간(세그먼트)의 대표 포트

각 링크(세그먼트)마다 1개씩 존재

해당 세그먼트에서 Root까지 비용이 제일 낮은 스위치의 포트

트래픽 전달 담당 포트 (Forwarding)

👉 쉽게 말해:
“이 구간에서 통신 담당자”

🔁 Alternate Port (AP) – RSTP 전용
🔄 Root Port의 백업 포트

RSTP(802.1w) 에서만 등장

평소엔 차단 상태 (Discarding)
BDPU를 수신받는 포트이지만 실제 데이터는 주고받지 않음.
BP가 선정되면 반대편은 DP가 됨
Root Port가 죽으면 👉 즉시 Forwarding으로 승격 ⚡

STP에서는 이게 그냥 Blocked Port로 뭉뚱그려짐

👉 쉽게 말해:
“루트로 가는 예비 경로”

🧠 한 방에 이해하는 그림 느낌
```
      [ Root Bridge ]
           |
         (DP)
           |
        [SW1]
        /    \
     (RP)   (AP)
      |      |
    [SW2]———[SW3]
          (DP)
```

SW1 → Root로 가는 포트 = Root Port

Root Bridge 쪽 포트 = Designated Port

SW1에서 Root 말고 다른 경로 = Alternate Port (백업)

⚔️ 한 줄 요약 비교
포트	정체성	평소 상태	핵심 역할
Root Port	루트로 가는 베스트 길	Forwarding	“내 주력 출구”
Designated Port	세그먼트 대표	Forwarding	“이 구간 통신 담당”
Alternate Port	백업 루트 경로	Discarding	“주력 죽으면 바로 투입”

💡 실무 감각 팁
RSTP 쓰는 환경이면 Alternate Port = 장애 대응 속도 핵심
STP만 쓰면 장애 나면 30초~50초 멈춤 → 체감 개빡셈
그래서 요즘은 거의 다 RSTP or PVST+ 씀

2️⃣ RSTP 포트 역할 (Roles)

RSTP는 포트 “역할”을 명확히 정의함:
Root Port (RP)
→ 루트 브리지로 가는 최단 경로
Designated Port (DP)
→ 해당 세그먼트의 대표 포트 (Forwarding)
Alternate Port (AP)
→ RP의 백업 경로 (평소 Discarding)
Backup Port (BP)
→ 같은 스위치 내 이중 연결 시 생기는 예비 포트 (드문 케이스)

👉 핵심: Alternate/Backup 포트가 이미 준비 중이라 장애 시 바로 승격 가능

1️⃣ STP vs RSTP 핵심 차이
구분	      STP (802.1D)	    RSTP (802.1w)
수렴 속도  	30~50초	          수 ms ~ 수 초
포트 상태  	5단계	            3단계
백업 포트  	Blocked (개념만)	  Alternate / Backup (명시)
링크 업 시  	타이머 기다림	    즉시 협상
토폴로지 변경 반응	느림    	  빠른 핸드셰이크
