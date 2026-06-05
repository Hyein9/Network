# OSPF 라우팅 실습

## 개념 정리

### OSPF (Open Shortest Path First)
- 링크 상태(Link-State) 기반의 라우팅 프로토콜로, RFC 2328에 정의되어 있음
- SPF(Dijkstra) 알고리즘으로 최단 경로를 계산하고, Area 단위로 네트워크를 계층적으로 분리함
- 같은 Area에 속한 라우터끼리 LSDB(Link-State Database)를 동기화하고, 이웃 상태가 FULL이 되어야 정상 동작

### OSPF 주요 용어
- **Process ID**: 한 라우터에서 여러 OSPF 프로세스를 구분하기 위한 값. 라우터마다 달라도 상관없음
- **Router ID**: 라우터를 식별하는 고유 값. 선정 우선순위는 `router-id` 명령 > Loopback IP > 가장 큰 활성 인터페이스 IP 순서
- **Area 0 (Backbone Area)**: 모든 Area는 Area 0과 직접 연결되거나, Virtual Link를 통해 논리적으로 연결되어야 함
- **ABR (Area Border Router)**: 서로 다른 Area를 연결하는 라우터
- **ASBR (AS Boundary Router)**: 다른 라우팅 프로토콜과 연결되는 경계 라우터

### LSA 타입
| 타입 | 이름 | 설명 |
|------|------|------|
| Type 1 | Router LSA | 각 라우터가 자기 링크 정보를 Area 내부에 광고 |
| Type 2 | Network LSA | DR이 멀티액세스 네트워크 정보를 광고 |
| Type 3 | Summary LSA | ABR이 다른 Area의 네트워크 정보를 요약해서 전달 |
| Type 4 | ASBR Summary LSA | ASBR의 위치를 다른 Area에 알려줌 |
| Type 5 | External LSA | ASBR이 외부 경로를 OSPF 도메인 전체에 광고 |

### 라우팅 테이블에서 OSPF 경로 표기
- **O**: 같은 Area 내부 경로 (Intra-Area)
- **O IA**: 다른 Area에서 온 경로 (Inter-Area)
- **O E1/E2**: 외부에서 재분배된 경로 (External)

### Virtual Link
- Area 0과 직접 연결되지 않는 Area가 있을 때, 중간 Area(Transit Area)를 경유해서 논리적으로 Area 0에 연결하는 기능
- Transit Area에 속한 두 ABR 사이에 설정하며, 양쪽 모두 상대방의 Router ID를 지정해야 함
- Transit Area에 인증이 걸려 있으면 Virtual Link도 영향을 받고, Virtual Link 자체에 별도로 인증을 걸 수도 있음

### OSPF 인증
- OSPF 이웃 간 허가되지 않은 라우터의 참여를 막기 위해 인증을 적용할 수 있음
- MD5 인증이 일반적이며, 양쪽 라우터에서 Key 번호와 암호가 동일해야 함
- Area 단위로 일괄 적용하거나, 인터페이스별로 개별 적용 가능

### Stub Area / Totally Stub Area
- LSDB에 저장되는 경로 정보를 줄여서 라우터 자원을 절약하기 위한 Area 타입
- 상세한 외부 경로 정보 대신 Default Route(0.0.0.0/0)를 받아서 통신함

| 구분 | 설명 |
|------|------|
| Stub Area | 외부 경로(LSA Type 5)를 수용하지 않음. Default Route로 외부와 통신. ASBR이 존재할 수 없음 (ABR=ASBR인 경우 제외) |
| Totally Stub Area | Stub Area의 특성에 더해 LSA Type 3(Summary)도 차단. LSA Type 3~5를 전부 Default Route로 대체. Cisco에서 개발했으나 현재는 대부분의 벤더가 지원 |

**Stub Area의 장점**
- LSDB 크기와 라우팅 테이블이 줄어들어 메모리와 CPU 리소스 절약
- 30분마다 수행되는 LSA Flooding 부하가 감소

### 재분배 (Redistribution)
- 서로 다른 라우팅 프로토콜 간에 경로를 교환하려면 ASBR에서 재분배 설정이 필요
- 양방향 통신을 위해 양쪽 프로토콜 모두에 재분배를 적용해야 함

재분배 설정 방법:
```
OSPF가 EIGRP 경로를 받을 때:
  router ospf [process-id]
  redistribute eigrp [AS number] subnets

EIGRP가 OSPF 경로를 받을 때:
  router eigrp [AS number]
  redistribute ospf [process-id] metric [Bandwidth] [Delay] [Reliability] [Load] [MTU]

직접 연결된 네트워크를 OSPF에 재분배:
  router ospf [process-id]
  redistribute connected subnets
```

### 축약 (Summarization)
- ABR 또는 ASBR에서만 가능하며, 여러 네트워크를 하나로 묶어서 광고함으로써 라우팅 테이블을 줄임
```
router ospf [process-id]
area [area number] range [네트워크 IP] [서브넷 마스크]
```

---

## 실습 1: 멀티 Area OSPF 구성

### 실습 환경
- Cisco Packet Tracer 사용
- 라우터 4대 (R1~R4), PC 4대

### IP 주소 구성

**라우터**

| 라우터 | 인터페이스 | IP 주소 |
|--------|-----------|---------|
| R1 | se0/0 | 1.1.1.1 /8 |
| R1 | fa0/0 | 192.168.10.2 /24 |
| R2 | se0/0 | 1.1.1.2 /8 |
| R2 | se0/1 | 2.1.1.1 /8 |
| R2 | fa0/0 | 126.255.255.2 /8 |
| R3 | se0/0 | 2.1.1.2 /8 |
| R3 | se0/1 | 3.1.1.1 /8 |
| R3 | fa0/0 | 191.255.255.2 /8 |
| R4 | se0/0 | 3.1.1.2 /8 |
| R4 | fa0/0 | 223.255.255.2 /24 |

**PC**

| PC | IP | Gateway |
|----|----|---------|
| PC1 | 192.168.10.1 | 192.168.10.2 |
| PC2 | 126.255.255.1 | 126.255.255.2 |
| PC3 | 191.255.255.1 | 191.255.255.2 |
| PC4 | 223.255.255.1 | 223.255.255.2 |

---

### 1단계. 물리 연결 및 기본 확인

- 라우터 전원 ON
- 라우터 간 Serial DCE 케이블 연결
- PC와 라우터는 Copper Straight-Through 케이블 사용
- `show ip interface brief`로 인터페이스 존재 여부 확인

### 2단계. 인터페이스 IP 설정

**R1**
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

**R2**
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

**R3**
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

**R4**
```
conf t
interface se0/0
 ip address 3.1.1.2 255.0.0.0
 no shutdown

interface fa0/0
 ip address 223.255.255.2 255.255.255.0
 no shutdown
```

### 3단계. PC IP 설정

각 PC의 Desktop > IP Configuration에서 IP와 Gateway 입력.

### 4단계. OSPF 설정

**R1 (Area 10)**
```
router ospf 1
 network 1.0.0.0 0.255.255.255 area 10
 network 192.168.10.0 0.0.0.255 area 10
```

**R2 (Area 10 / Area 0 / Area 20 - ABR)**
```
router ospf 1
 network 1.0.0.0 0.255.255.255 area 10
 network 2.0.0.0 0.255.255.255 area 0
 network 126.0.0.0 0.255.255.255 area 20
```

**R3 (Area 0 / Area 30 / Area 40 - ABR)**
```
router ospf 1
 network 2.0.0.0 0.255.255.255 area 0
 network 3.0.0.0 0.255.255.255 area 40
 network 191.255.0.0 0.255.255.255 area 30
```

**R4 (Area 40)**
```
router ospf 1
 network 3.0.0.0 0.255.255.255 area 40
 network 223.255.255.0 0.0.0.255 area 40
```

### 5단계. OSPF 이웃 확인

```
show ip ospf neighbor
```
이웃 라우터가 FULL 상태여야 정상.

```
show ip ospf database
```
LSA들이 Area 간에 교환되는지 확인.

### 6단계. 라우팅 테이블 확인

```
show ip route
```
- O 표시: 같은 Area 경로
- O IA 표시: 다른 Area 경로

### 7단계. 통신 테스트

```
PC1 > ping 223.255.255.1
```
모든 PC 간 ping이 성공해야 정상.

---

## 실습 2: 단일 Area OSPF 구성

### 토폴로지

```
[LAN1]                     [LAN2]
192.168.10.0/24            192.168.20.0/24
     |                          |
   R1 f0/0                  R2 f0/0
 192.168.10.1             192.168.20.1
     |                          |
   R1 s0/0 ---- 10.10.10.0/24 ---- R2 s0/0
  10.10.10.1              10.10.10.254
```

- OSPF Area: Area 0
- Router ID: R1 = 1.1.1.1, R2 = 1.1.1.2

### Router ID 관련 주의사항

OSPF는 시작할 때 반드시 Router ID가 필요하다. 아래 우선순위로 자동 선정됨:
1. `router-id` 명령으로 수동 지정한 값
2. Loopback 인터페이스 IP
3. 가장 큰 활성 인터페이스 IP

인터페이스 IP도 없고 router-id도 없는 상태에서 OSPF를 실행하면 `%OSPF-4-NORTRID: OSPF process 1 failed to allocate unique router-id` 에러가 발생한다. 반드시 인터페이스 IP를 먼저 설정한 뒤에 OSPF를 실행해야 함.

### R1 설정

**(1) 인터페이스 IP 설정**
```
conf t
int s0/0
 ip address 10.10.10.1 255.255.255.0
 no shutdown

int f0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

**(2) OSPF 프로세스 생성 및 Router ID 지정**
```
router ospf 1
 router-id 1.1.1.1
```

**(3) 네트워크 광고**
```
 network 10.10.10.0 0.0.0.255 area 0
 network 192.168.10.0 0.0.0.255 area 0
```

확인:
```
show ip ospf database
```
`OSPF Router with ID (1.1.1.1)` 출력되면 정상.

### R2 설정

**(1) 인터페이스 IP 설정**
```
conf t
int s0/0
 ip address 10.10.10.254 255.255.255.0
 no shutdown

int f0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```

**(2) OSPF 실행 및 네트워크 광고**
```
router ospf 2
 network 10.10.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
```

**(3) Router ID 변경**
```
 router-id 1.1.1.2
```

이미 OSPF가 동작 중이라 `Reload or use "clear ip ospf process"` 메시지가 뜬다. 아래 명령으로 프로세스를 재시작해야 반영됨:
```
clear ip ospf process
```

### 인접 상태 확인

```
%OSPF-5-ADJCHG: Nbr 1.1.1.1 on Serial0/0 from LOADING to FULL
```
FULL 상태가 되면 이웃 형성 성공.

`show ip ospf database`에서 Router LSA가 1.1.1.1과 1.1.1.2 두 개 보이면 정상.

---

## 실습 3: Virtual Link 구성

### 토폴로지

```
[R1] ---- [R2] ---- [R3] ---- [R4] ---- [R5]
  area 0     area 1     area 1     area 2
     192.168.10   192.168.20   192.168.30   192.168.40
       .0/24        .0/24        .0/24        .0/24
```

| 라우터 | 인터페이스 | IP | Area |
|--------|-----------|-----|------|
| R1 | s0/0 | 192.168.10.1/24 | area 0 |
| R2 | s0/0 | 192.168.10.2/24 | area 0 |
| R2 | s0/1 | 192.168.20.1/24 | area 1 |
| R3 | s0/1 | 192.168.20.2/24 | area 1 |
| R3 | s0/2 | 192.168.30.1/24 | area 1 |
| R4 | s0/2 | 192.168.30.2/24 | area 1 |
| R4 | s0/3 | 192.168.40.1/24 | area 2 |
| R5 | s0/3 | 192.168.40.2/24 | area 2 |

Router ID: R1=1.1.1.1, R2=2.2.2.2, R3=3.3.3.3, R4=4.4.4.4, R5=5.5.5.5

### 1단계. IP 주소 설정

**R1**
```
conf t
int s0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

**R2**
```
conf t
int s0/0
 ip address 192.168.10.2 255.255.255.0
 no shutdown

int s0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```

**R3**
```
conf t
int s0/1
 ip address 192.168.20.2 255.255.255.0
 no shutdown

int s0/2
 ip address 192.168.30.1 255.255.255.0
 no shutdown
```

**R4**
```
conf t
int s0/2
 ip address 192.168.30.2 255.255.255.0
 no shutdown

int s0/3
 ip address 192.168.40.1 255.255.255.0
 no shutdown
```

**R5**
```
conf t
int s0/3
 ip address 192.168.40.2 255.255.255.0
 no shutdown
```

### 2단계. OSPF 기본 설정

Router ID 설정 후 OSPF 재시작이 필요할 수 있음.

**R1**
```
router ospf 1
 router-id 1.1.1.1
 network 192.168.10.0 0.0.0.255 area 0
```

**R2**
```
router ospf 1
 router-id 2.2.2.2
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 1
```

**R3**
```
router ospf 1
 router-id 3.3.3.3
 network 192.168.20.0 0.0.0.255 area 1
 network 192.168.30.0 0.0.0.255 area 1
```

**R4**
```
router ospf 1
 router-id 4.4.4.4
 network 192.168.30.0 0.0.0.255 area 1
 network 192.168.40.0 0.0.0.255 area 2
```

**R5**
```
router ospf 1
 router-id 5.5.5.5
 network 192.168.40.0 0.0.0.255 area 2
```

여기까지만 하면 Area 2는 Area 0과 직접 연결되지 않아서 R5에서 Area 0 경로가 보이지 않는다.

### 3단계. Virtual Link 설정

Area 2가 Area 0에 도달하려면, Transit Area(Area 1)를 경유하는 Virtual Link가 필요하다. R2(ABR)와 R4(ABR) 사이에 설정.

**R2**
```
router ospf 1
 area 1 virtual-link 4.4.4.4
```

**R4**
```
router ospf 1
 area 1 virtual-link 2.2.2.2
```

확인:
```
show ip ospf virtual-links
```
State가 FULL이면 정상.

### 4단계. OSPF 인증 설정 (Area 1 기준)

Virtual Link는 Transit Area의 인증 설정을 따라간다. Area 1에 인증을 적용하면 Virtual Link에도 영향을 줌.

**(1) Area 1 전체에 MD5 인증 활성화 (R2, R3, R4)**
```
router ospf 1
 area 1 authentication message-digest
```

**(2) 인터페이스별 Key 설정**

**R2**
```
int s0/1
 ip ospf message-digest-key 1 md5 CISCO
```

**R3**
```
int s0/1
 ip ospf message-digest-key 1 md5 CISCO

int s0/2
 ip ospf message-digest-key 1 md5 CISCO
```

**R4**
```
int s0/2
 ip ospf message-digest-key 1 md5 CISCO
```

Key 번호와 암호가 양쪽에서 반드시 동일해야 한다.

### Virtual Link 전용 인증 설정

Area 인증과 별개로 Virtual Link에만 독립적으로 인증을 걸 수도 있다. 설정은 router ospf 모드에서 진행.

**R2**
```
router ospf 1
 area 1 virtual-link 4.4.4.4 authentication message-digest
 area 1 virtual-link 4.4.4.4 message-digest-key 1 md5 CISCO
```

**R4**
```
router ospf 1
 area 1 virtual-link 2.2.2.2 authentication message-digest
 area 1 virtual-link 2.2.2.2 message-digest-key 1 md5 CISCO
```

확인:
```
show ip ospf virtual-links
```
`Authentication: Message Digest`와 `State: FULL`이 출력되면 정상.

### Area 인증 vs Virtual Link 인증 비교

| 구분 | 적용 범위 |
|------|----------|
| Area Authentication | Area 전체 (모든 인터페이스 + Virtual Link 포함) |
| Virtual Link Authentication | 해당 Virtual Link만 |

Virtual Link는 Area 인증을 상속받을 수도 있고, 자체적으로 별도 인증을 가질 수도 있다. 시험에서는 이 차이를 묻는 문제가 자주 출제됨.

### 5단계. 최종 확인

```
show ip ospf neighbor
show ip ospf virtual-links
show ip route ospf
```

- Neighbor: FULL 상태
- Virtual Link: UP 상태
- R5에서 Area 0 경로가 라우팅 테이블에 보이면 정상

---

## OSPF 기본 설정 명령어 정리

```
router ospf [process-id]
 router-id x.x.x.x
 network [네트워크 대역] [와일드카드 마스크] area [area 번호]
```

## 축약 설정 명령어

```
router ospf [process-id]
 area [area 번호] range [네트워크 IP] [와일드카드 마스크]
```

## 재분배 설정 명령어

```
-- OSPF에 EIGRP 경로 재분배
router ospf [process-id]
 redistribute eigrp [AS number] subnets

-- EIGRP에 OSPF 경로 재분배
router eigrp [AS number]
 redistribute ospf [process-id] metric [Bandwidth] [Delay] [Reliability] [Load] [MTU]

-- 직접 연결 네트워크 재분배
router ospf [process-id]
 redistribute connected subnets
```
