# Cisco-Packet-Tracer

## OSPF configuration

 1) 기본 설정
> router ospf [process-id]
   - 하나의 라우터에 여러개의 OSPF를 설정할 때 구분하기 위한 값이기 때문에 다른 라우터들끼리 달라도 상관이 없음
> router-id x.x.x.x 
   - 최적의 경로를 누가 알려줬는지 파악하기 위해 필요, 일반적으로 가장 작은 IP를 설정하기 때문에 loop back 같은곳에 IP를 넣고 설정
> network [네트워크대역] [와일드마스크] area [area number]
 
 2) 축약 설정
 - ABR 또는 ASBR에서만 축약 가능
> router ospf [process-id]
> area [축약하고싶은 area number] range [N/W IP] [wildcard mask]
 
 3) 재분배
 - ASBR 에서 재분배 설정을 해야 함
 - 모두 통신이 되기 위해선 다른 프로토콜을 사용하는 인터페이스와 OSPF를 사용하는 인터페이스 모두 재분배를 해주어야 쌍방향 통신 가능
 ㄱ) OSPF가 EIGRP를 알 수 있도록 OSPF에 재분배 설정 방법
    -> router ospf [process-id]
    -> redistribute [반대편 프로토콜] [AS number] subnets
 ㄴ) EIGRP가 OSPF를 알 수 있또록 EIGRP에 재분배 설정 방법
    -> router eigrp [AS number]
    -> redistribute [반대편 프로토콜] [process number] metric [Bandwidth값] [delay값] [reliability값] [load값] [MTU값]
 ㄷ) 직접 연결된 대역대를 OSPF에게 재분배
    -> router ospf [process-id]
    -> redistribute connected subnets
출처: https://nirsa.tistory.com/32 [The Nirsa Way:티스토리]
```
#### 🌐 IP 주소
```
🔹 R1
G0/0 → 192.168.10.1 /24
G0/1 → 10.0.12.1 /30

🔹 R2
G0/0 → 10.0.12.2 /30
G0/1 → 10.0.23.1 /30

🔹 R3
G0/0 → 10.0.23.2 /30
G0/1 → 192.168.30.1 /24

🔹 PC
PC1: 192.168.10.10 /24 GW 192.168.10.1
PC2: 192.168.10.11 /24 GW 192.168.10.1
PC3: 192.168.30.10 /24 GW 192.168.30.1
PC4: 192.168.30.11 /24 GW 192.168.30.1
```

✅ 1단계. 물리 연결 & 기본 확인

라우터 전원 ON

라우터 간 Serial DCE 케이블 연결

PC ↔ Router는 Copper Straight-Through

시리얼 인터페이스 존재 확인
```
show ip interface brief
```
✅ 2단계. 인터페이스 IP 설정 (라우터별)
🔹 R1
```
conf t
interface se0/0
ip address 1.1.1.1 255.0.0.0
clock rate 64000
no shutdown

interface fa0/0
ip address 192.168.10.2 255.255.255.0
no shutdown
```
🔹 R2
```
conf t
interface se0/0
ip address 1.1.1.2 255.0.0.0
no shutdown

interface se0/1
ip address 2.1.1.1 255.0.0.0
clock rate 64000
no shutdown

interface fa0/0
ip address 126.255.255.2 255.0.0.0
no shutdown
```
🔹 R3
```
conf t
interface se0/0
ip address 2.1.1.2 255.0.0.0
no shutdown

interface se0/1
ip address 3.1.1.1 255.0.0.0
clock rate 64000
no shutdown

interface fa0/0
ip address 191.255.255.2 255.0.0.0
no shutdown
```
🔹 R4
```
conf t
interface se0/0
ip address 3.1.1.2 255.0.0.0
no shutdown

interface fa0/0
ip address 223.255.255.2 255.255.255.0
no shutdown
```
✅ 3단계. PC IP 설정
```
PC	IP	Gateway
PC1	192.168.10.1	192.168.10.2
PC2	126.255.255.1	126.255.255.2
PC3	191.255.255.1	191.255.255.2
PC4	223.255.255.1	223.255.255.2
```
✅ 4단계. OSPF 설정 (가장 중요)
🔹 R1 (Area 10)
```
router ospf 1
network 1.0.0.0 0.255.255.255 area 10
network 192.168.10.0 0.0.0.255 area 10

🔹 R2 (Area 10 / 0 / 20 → ABR)
router ospf 1
network 1.0.0.0 0.255.255.255 area 10
network 2.0.0.0 0.255.255.255 area 0
network 126.0.0.0 0.255.255.255 area 20

🔹 R3 (Area 0 / 40 / 30 → ABR)
router ospf 1
network 2.0.0.0 0.255.255.255 area 0
network 3.0.0.0 0.255.255.255 area 40
network 191.255.0.0 0.255.255.255 area 30

🔹 R4 (Area 40)
router ospf 1
network 3.0.0.0 0.255.255.255 area 40
network 223.255.255.0 0.0.0.255 area 40
```
✅ 5단계. OSPF 이웃 확인
```
show ip ospf neighbor ✔ 이웃 라우터가 FULL 상태여야 함

show ip ospf database ✔ LSA들이 Area 간에 교환되는지 확인
```
정상 출력:

FULL/ -

✅ 6단계. 라우팅 테이블 확인
show ip route


✔ 표시 확인:

O → 같은 Area

O IA → 다른 Area

✅ 7단계. 통신 테스트
PC1 > ping 223.255.255.1


👉 모든 PC 간 ping 성공해야 정상



# 토폴로지(2)
1️⃣ 토폴로지 해석 (그림 기준)
```
[LAN1]                     [LAN2]
192.168.10.0/24            192.168.20.0/24
     |                          |
   R1 f0/0                  R2 f0/0
192.168.10.1              192.168.20.1
     |                          |
   R1 s0/0  ---- 10.10.10.0/24 ---- R2 s0/0
 10.10.10.1               10.10.10.254
```

OSPF Area: Area 0
Router-ID
R1 → 1.1.1.1
R2 → 1.1.1.2

목표
R1 ↔ R2 OSPF 인접(FULL)
LAN 간 상호 통신 가능
2️⃣ 네가 처음에 막혔던 이유 (중요 ⭐)
❌ 에러
%OSPF-4-NORTRID: OSPF process 1 failed to allocate unique router-id

🔍 원인
OSPF는 Router-ID가 반드시 필요한데,
OSPF 실행 시점에 아래 조건을 만족 못 함:

OSPF Router-ID 선정 우선순위:

router-id 명령

Loopback IP

가장 큰 활성 인터페이스 IP

👉 처음엔

인터페이스 IP도 없고

router-id도 없어서
👉 OSPF가 시작 자체를 못 함

3️⃣ R1 설정 흐름 정리 (깔끔 버전)
✅ (1) 인터페이스 IP 먼저
```
R1(config)#int s0/0
R1(config-if)#ip address 10.10.10.1 255.255.255.0
R1(config-if)#no shutdown

R1(config)#int f0/0
R1(config-if)#ip address 192.168.10.1 255.255.255.0
R1(config-if)#no shutdown
```
✅ (2) OSPF 프로세스 생성
```
R1(config)#router ospf 1
```

이 시점엔 Router-ID가 자동으로
👉 10.10.10.1로 잡힘 (확인됨)

✅ (3) Router-ID 수동 지정 (베스트 프랙티스)
```
R1(config-router)#router-id 1.1.1.1
```

확인:
```
R1#show ip ospf database
OSPF Router with ID (1.1.1.1)
```
✅ (4) 네트워크 광고
```
R1(config-router)#network 10.10.10.0 0.0.0.255 area 0
R1(config-router)#network 192.168.10.0 0.0.0.255 area 0
```
4️⃣ R2 설정 흐름 정리
✅ (1) 인터페이스 IP
```
R2(config)#int s0/0
R2(config-if)#ip address 10.10.10.254 255.255.255.0
R2(config-if)#no shutdown

R2(config)#int f0/0
R2(config-if)#ip address 192.168.20.1 255.255.255.0
R2(config-if)#no shutdown
```
✅ (2) OSPF 실행
```
R2(config)#router ospf 2
```

초기 Router-ID → 192.168.20.1

✅ (3) 네트워크 광고
```
R2(config-router)#network 10.10.10.0 0.0.0.255 area 0
R2(config-router)#network 192.168.20.0 0.0.0.255 area 0
```
✅ (4) Router-ID 변경
```
R2(config-router)#router-id 1.1.1.2
```

⚠️ 이미 OSPF가 돌아가고 있어서
Reload or use "clear ip ospf process"


→ 그래서 한 거 👇
R2#clear ip ospf process

5️⃣ OSPF 인접 상태 정상 확인
%OSPF-5-ADJCHG: Nbr 1.1.1.1 on Serial0/0 from LOADING to FULL


👉 OSPF 인접 FULL = 성공

DB 확인:

R2#show ip ospf database


결과:

Router LSA 2개

1.1.1.1

1.1.1.2

정상 👍


## Virtual-LiNK
0️⃣ 전제 (그림 기준 정리)
Router	Interface	IP	Area
R1	s0/0	192.168.10.1/24	area 0
R2	s0/0	192.168.10.2/24	area 0
R2	s0/1	192.168.20.1/24	area 1
R3	s0/1	192.168.20.2/24	area 1
R3	s0/2	192.168.30.1/24	area 1
R4	s0/2	192.168.30.2/24	area 1
R4	s0/3	192.168.40.1/24	area 2
R5	s0/3	192.168.40.2/24	area 2

Router ID:

R1: 1.1.1.1

R2: 2.2.2.2

R3: 3.3.3.3

R4: 4.4.4.4

R5: 5.5.5.5

1️⃣ IP 주소 설정 (전부 no shut 필수)
🔹 R1
```
conf t
int s0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```
🔹 R2
```
conf t
int s0/0
 ip address 192.168.10.2 255.255.255.0
 no shutdown

int s0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```
🔹 R3
```
conf t
int s0/1
 ip address 192.168.20.2 255.255.255.0
 no shutdown

int s0/2
 ip address 192.168.30.1 255.255.255.0
 no shutdown
```
🔹 R4
```
conf t
int s0/2
 ip address 192.168.30.2 255.255.255.0
 no shutdown

int s0/3
 ip address 192.168.40.1 255.255.255.0
 no shutdown
```
🔹 R5
```
conf t
int s0/3
 ip address 192.168.40.2 255.255.255.0
 no shutdown
```
2️⃣ OSPF 기본 설정 (Router ID 고정)

⚠️ Router ID 설정 후 OSPF 재시작 필요
```
R1
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0

R2
router ospf 1
 router-id 2.2.2.2
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 1

R3
router ospf 1
 router-id 3.3.3.3
 network 192.168.20.0 0.0.0.255 area 1
 network 192.168.30.0 0.0.0.255 area 1

R4
router ospf 1
 router-id 4.4.4.4
 network 192.168.30.0 0.0.0.255 area 1
 network 192.168.40.0 0.0.0.255 area 2

R5
router ospf 1
 router-id 5.5.5.5
 network 192.168.40.0 0.0.0.255 area 2
```

👉 여기까지 하면
❌ Area 2 ↔ Area 0 안 됨 (정상)

3️⃣ Virtual Link 설정 (핵심)

📌 Transit Area = area 1
📌 R2 ↔ R4
```
R2
router ospf 1
 area 1 virtual-link 4.4.4.4

R4
router ospf 1
 area 1 virtual-link 2.2.2.2
```

확인:
```
show ip ospf virtual-links
```

→ State: FULL

4️⃣ OSPF Authentication 설정 (Area 1 기준)

👉 Virtual Link는 Transit Area의 인증을 그대로 따라감
그래서 Area 1에 인증 걸면 Virtual Link에도 적용됨

🔐 방식: MD5 인증 (시험·실무 국룰)
🔹 Area 1 전체에 인증 활성화

(R2, R3, R4 전부)
```
router ospf 1
 area 1 authentication message-digest
```
🔹 인터페이스별 Key 설정
```
R2
int s0/1
 ip ospf message-digest-key 1 md5 CISCO

R3
int s0/1
 ip ospf message-digest-key 1 md5 CISCO
int s0/2
 ip ospf message-digest-key 1 md5 CISCO

R4
int s0/2
 ip ospf message-digest-key 1 md5 CISCO
```

⚠️ Key 번호 / 암호 반드시 동일

5️⃣ 최종 확인 체크리스트 ✅
```
show ip ospf neighbor
show ip ospf virtual-links
show ip route ospf
```

정상 결과:

Neighbor: FULL

Virtual Link: UP

R5에서 Area 0 경로 보임

🔥 시험용 한 줄 정리

Virtual Link 인증은
Transit Area(Authentication)을 그대로 상속한다

Virtual-link에 “직접” authentication 거는 법 정리해줄게.
(Area 인증이랑 헷갈리는 데서 시험 문제 많이 나와 😈)

🔑 핵심 먼저

❌ area X authentication → Virtual-link 전용 아님

✅ Virtual-link는 따로 인증 설정 가능

설정 위치: router ospf 모드

양쪽 ABR에 동일하게 설정 필수

🎯 이 토폴로지 기준

Virtual-link: R2 ↔ R4

Transit Area: Area 1

Router ID

R2 → 2.2.2.2

R4 → 4.4.4.4

⚙️ Virtual-Link 전용 Authentication 설정
🔐 MD5 인증 (추천 / 시험 국룰)
▶ R2
```
router ospf 1
 area 1 virtual-link 4.4.4.4 authentication message-digest
 area 1 virtual-link 4.4.4.4 message-digest-key 1 md5 CISCO

▶ R4
router ospf 1
 area 1 virtual-link 2.2.2.2 authentication message-digest
 area 1 virtual-link 2.2.2.2 message-digest-key 1 md5 CISCO
```

📌 포인트

authentication message-digest → 인증 방식

message-digest-key → 키 지정

키 번호 / 암호 / 방식 전부 동일해야 함

🔍 확인 명령어
show ip ospf virtual-links


정상이면:

Authentication: Message Digest
State: FULL

⚠️ 시험·실무 함정 정리
❌ 이렇게 하면 안 됨

한쪽만 설정

RID 잘못 입력

Key 번호 다름

Area 인증만 걸고 Virtual-link 인증 안 건 경우

✅ 이렇게 이해하면 100점

Virtual-link는
Area 인증을 상속받을 수도 있고,
자기만의 인증을 가질 수도 있다

🧠 Area 인증 vs Virtual-Link 인증 차이
구분	적용 범위
Area Authentication	Area 전체 (모든 인터페이스 + VL)
Virtual-Link Authentication	해당 Virtual-link만
🔥 한 줄 요약

Virtual-link 인증은
router ospf 모드에서
area X virtual-link RID authentication





● OSPF Stub Area & Totally Stub Area 정의

 

LSDB에 저장되는 경로정보를 축소 및 프로세싱 자원을 절약하기 위해 상세한 외부경로 정보대신 Default Gateway 정보를 받아 라우팅 테이블에 내리는 기술이 적용되는 Area를 의미 합니다.

 

● OSPF Stub Area & Totally Stub Area 장점 및 특징

 

장점	- LSDB 크기 및 라우팅 테이블이 감소하여 자원 저장과 계산에 사용되는 리소스를 절약할 수 있습니다.
- 30분 마다 수행하는 LSA Flooding 작업의 부하가 감소 합니다.
특징	Stub Area	- ASBR이 수행한 재분배를 통해 학습된 외부경로를 수용하지 않는 Area 입니다. 
- 상세한 외부 정보 대신 Default Route (0.0.0.0/0)을 사용하여 외부와 통신 합니다. 
- Stub Area로 설정되면 ASBR이 존재할 수 없습니다. (ABR = ASBR인 경우 제외)
Totally Stub Area	- Cisco 전용 Area Type 입니다. (최근에는 대부분의 Vendor가 지원 합니다.)
- Stub Area의 특징을 가지고 있으며 LSA Type 3  정보도 받지 않는 Area 입니다.
- LSA Type 3 ~5 정보를 받지 않는 대신 Default Route를 사용하여 통신을 수행합니다.
