# Policy Based Routing (PBR) 실습

## 개념 정리

### PBR이란
- 일반적인 Destination 기반 라우팅과 달리, 정책(Policy)에 따라 패킷의 경로를 결정하는 기술
- 출발지 IP, 프로토콜, 포트 번호 등 다양한 조건을 기반으로 트래픽별 다른 경로를 지정할 수 있음

### 일반 라우팅 vs PBR

```
일반 라우팅:
  패킷 수신 → 목적지 IP 확인 → 라우팅 테이블 조회 → Next Hop 결정
  (목적지가 같으면 모두 같은 경로)

PBR:
  패킷 수신 → Policy 조건 확인 → 조건 일치 시 지정된 경로로 전달
                                → 불일치 시 일반 라우팅 테이블 사용
  (출발지, 프로토콜 등에 따라 다른 경로)
```

| 구분 | 일반 라우팅 | PBR |
|------|----------|-----|
| 판단 기준 | Destination IP만 | Source IP, Protocol, Port 등 |
| 경로 결정 | 라우팅 테이블 기반 | 정책(Route-map) 기반 |
| 유연성 | 제한적 | 높음 (세밀한 트래픽 제어) |
| 처리 순서 | 라우팅 테이블 조회 | 라우팅 테이블 조회 전에 정책 검사 |

### PBR 동작 흐름

```
패킷 수신
    |
    v
Route-map 조건 확인
    |
    +-- Match → set ip next-hop으로 지정된 경로 사용
    |
    +-- No Match → 일반 라우팅 테이블 사용
```

### PBR 구성 요소

| 요소 | 역할 |
|------|------|
| ACL | 정책 대상 트래픽을 정의 (match 조건) |
| Route-map | ACL과 Next Hop을 연결하는 정책 |
| `match ip address` | 트래픽 매칭 조건 (ACL 참조) |
| `set ip next-hop` | 매칭된 트래픽의 경로 지정 |
| `ip policy route-map` | 인터페이스에 PBR 적용 |

### PBR 사용 사례

```
1. 멀티 ISP 로드 분산
   사내망 A → ISP1
   사내망 B → ISP2

2. 특정 서비스 전용 회선
   VoIP 트래픽 → 저지연 전용 회선
   일반 인터넷 → 일반 ISP

3. 보안 정책
   특정 서버 트래픽 → 방화벽 경유

4. 트래픽 엔지니어링
   특정 경로 우회 (장애 대비, 부하 분산)
```

### Route-map 개념

Route-map은 if-then 구조로 동작한다:

```
route-map PBR permit 10        ← sequence 10
 match ip address 10           ← if (ACL 10에 매칭되면)
 set ip next-hop 1.1.1.1      ← then (1.1.1.1로 보냄)

route-map PBR permit 20        ← sequence 20
 match ip address 20           ← if (ACL 20에 매칭되면)
 set ip next-hop 2.2.2.2      ← then (2.2.2.2로 보냄)

매칭 안 되면 → 일반 라우팅 테이블 사용
```

Route-map은 위에서 아래로 sequence 순서대로 검사하며, 매칭되면 해당 동작 수행 후 종료.

---

## 실습 토폴로지

### 네트워크 구조

```
                                  [R3]
                                f0/1  f0/0
                               .2      .1
           f0/1    1.1.23.0/24    1.1.34.0/24    f0/1
  [R1]----f0/0----[R2]-----------------------------[R4]----f0/0----[R5]
           10.1.1   |        기본 경로               |      10.1.45
           .0/24    |   R2 → R3 → R4                |       .0/24
                    |                                |
                    s0/0                          s0/0
                    |     10.1.24.0/24             |
                    +-------(Serial Link)----------+
                          PBR 적용 후 경로
                          R2 → R4 직접
```

### 상세 연결도

```
  [R1]               [R2]               [R3]               [R4]               [R5]
  f0/0  Lo0  Lo1     f0/0  f0/1  s0/0   f0/1  f0/0        f0/1  f0/0  s0/0   f0/0  Lo0  Lo1
   |                   |     |     |      |     |            |     |     |      |
   +---10.1.1.0/24----+     |     |      |     |            |     |     |      +---10.1.45.0/24
   .1              .2        |     |      |     |            |     |     .4      .2
                             |     .1     .2    .1           .2    .1            172.16.5.1
                             |  10.1.23   10.1.34         10.1.45               172.16.55.1
  192.168.1.1                |  .0/24     .0/24            .0/24
  192.168.11.1               |
                              +---10.1.24.0/24---+
                              .1    (Serial)     .4
```

### IP 주소 구성

| 라우터 | 인터페이스 | IP 주소 | 용도 |
|--------|-----------|---------|------|
| R1 | f0/0 | 10.1.1.1/24 | R2 연결 |
| R1 | Lo0 | 192.168.1.1/24 | PBR 대상 네트워크 |
| R1 | Lo1 | 192.168.11.1/24 | 비교용 네트워크 |
| R2 | f0/0 | 10.1.1.2/24 | R1 연결 |
| R2 | f0/1 | 10.1.23.1/24 | R3 연결 (기본 경로) |
| R2 | s0/0 | 10.1.24.1/24 | R4 직결 (PBR 경로) |
| R3 | f0/1 | 10.1.23.2/24 | R2 연결 |
| R3 | f0/0 | 10.1.34.1/24 | R4 연결 |
| R4 | f0/1 | 10.1.34.2/24 | R3 연결 |
| R4 | f0/0 | 10.1.45.1/24 | R5 연결 |
| R4 | s0/0 | 10.1.24.4/24 | R2 직결 (PBR 경로) |
| R5 | f0/0 | 10.1.45.2/24 | R4 연결 |
| R5 | Lo0 | 172.16.5.1/24 | 목적지 네트워크 |
| R5 | Lo1 | 172.16.55.1/24 | 목적지 네트워크 |

### 경로 비교

```
기본 경로 (OSPF):    R1 → R2 → R3 → R4 → R5  (FastEthernet 경유)
PBR 적용 후:         R1 → R2 → R4 → R5        (Serial 직결, R3 우회)
```

---

## 1단계. 인터페이스 IP 설정

**R1**
```
conf t
interface f0/0
 ip address 10.1.1.1 255.255.255.0
 no shutdown

interface lo0
 ip address 192.168.1.1 255.255.255.0

interface lo1
 ip address 192.168.11.1 255.255.255.0
end
```

**R2**
```
conf t
interface f0/0
 ip address 10.1.1.2 255.255.255.0
 no shutdown

interface f0/1
 ip address 10.1.23.1 255.255.255.0
 no shutdown

interface s0/0
 ip address 10.1.24.1 255.255.255.0
 clock rate 64000
 no shutdown
end
```

**R3**
```
conf t
interface f0/1
 ip address 10.1.23.2 255.255.255.0
 no shutdown

interface f0/0
 ip address 10.1.34.1 255.255.255.0
 no shutdown
end
```

**R4**
```
conf t
interface f0/1
 ip address 10.1.34.2 255.255.255.0
 no shutdown

interface f0/0
 ip address 10.1.45.1 255.255.255.0
 no shutdown

interface s0/0
 ip address 10.1.24.4 255.255.255.0
 no shutdown
end
```

**R5**
```
conf t
interface f0/0
 ip address 10.1.45.2 255.255.255.0
 no shutdown

interface lo0
 ip address 172.16.5.1 255.255.255.0

interface lo1
 ip address 172.16.55.1 255.255.255.0
end
```

---

## 2단계. OSPF 설정

**R1**
```
router ospf 1
 network 10.1.1.0 0.0.0.255 area 0
 network 192.168.1.0 0.0.0.255 area 0
 network 192.168.11.0 0.0.0.255 area 0
```

**R2**
```
router ospf 1
 network 10.1.1.0 0.0.0.255 area 0
 network 10.1.23.0 0.0.0.255 area 0
 network 10.1.24.0 0.0.0.255 area 0
```

**R3**
```
router ospf 1
 network 10.1.23.0 0.0.0.255 area 0
 network 10.1.34.0 0.0.0.255 area 0
```

**R4**
```
router ospf 1
 network 10.1.24.0 0.0.0.255 area 0
 network 10.1.34.0 0.0.0.255 area 0
 network 10.1.45.0 0.0.0.255 area 0
```

**R5**
```
router ospf 1
 network 10.1.45.0 0.0.0.255 area 0
 network 172.16.5.0 0.0.0.255 area 0
 network 172.16.55.0 0.0.0.255 area 0
```

---

## 3단계. 기본 경로 확인

R1에서 traceroute:
```
R1# traceroute 172.16.5.1
```

PBR 적용 전 결과:
```
1  10.1.1.2      (R2)
2  10.1.23.2     (R3)
3  10.1.34.2     (R4)
4  10.1.45.2     (R5 - 도착)
```

기본 OSPF 라우팅은 FastEthernet 경로(R2 → R3 → R4)를 선택한다. Serial 링크는 Cost가 높아 기본 경로로 사용되지 않음.

---

## 4단계. PBR 설정 (R2)

R1의 Loopback0(192.168.1.0/24)에서 오는 트래픽을 R3을 우회하여 R4로 직접 보내도록 설정.

### ACL 생성

```
conf t
access-list 10 permit 192.168.1.0 0.0.0.255
```

### Route-map 생성

```
route-map PBR permit 10
 match ip address 10
 set ip next-hop 10.1.24.4
exit
```

- `match ip address 10`: ACL 10에 매칭되는 출발지(192.168.1.0/24) 트래픽
- `set ip next-hop 10.1.24.4`: R4의 Serial 인터페이스로 직접 전달

### 인터페이스에 PBR 적용

```
interface f0/0
 ip policy route-map PBR
end
```

R1에서 들어오는 트래픽(f0/0)에 PBR을 적용.

---

## 5단계. PBR 적용 결과 확인

### Route-map 확인

```
R2# show route-map
```

출력 예시:
```
route-map PBR, permit, sequence 10
  Match clauses:
    ip address (access-lists): 10
  Set clauses:
    ip next-hop 10.1.24.4
  Policy routing matches: 0 packets, 0 bytes
```

### PBR 정책 확인

```
R2# show ip policy
```

출력 예시:
```
Interface      Route map
FastEthernet0/0 PBR
```

### Traceroute로 경로 변경 확인

R1의 Loopback0에서 출발:
```
R1# traceroute 172.16.5.1 source 192.168.1.1
```

PBR 적용 후 결과:
```
1  10.1.1.2      (R2)
2  10.1.24.4     (R4) ← R3 우회, Serial 직결
3  10.1.45.2     (R5 - 도착)
```

R1의 Loopback1에서 출발 (PBR 대상 아님):
```
R1# traceroute 172.16.5.1 source 192.168.11.1
```

결과 (기본 경로 사용):
```
1  10.1.1.2      (R2)
2  10.1.23.2     (R3) ← 기본 OSPF 경로
3  10.1.34.2     (R4)
4  10.1.45.2     (R5 - 도착)
```

### 경로 비교 다이어그램

```
192.168.1.0/24에서 출발 (PBR 적용):

  [R1] → [R2] → [R4] → [R5]
              Serial 직결

192.168.11.0/24에서 출발 (PBR 미적용):

  [R1] → [R2] → [R3] → [R4] → [R5]
              FastEthernet 경유 (기본 OSPF)
```

---

## 실습 확장: OSPF Redistribution과 Route-map Metric 제어

PBR에 이어서, Route-map을 활용한 OSPF 재분배 시 Metric과 Metric Type 제어를 실습한다.

### OSPF External Route 타입

| 타입 | Cost 계산 | 설명 |
|------|----------|------|
| E2 (기본값) | External Metric만 사용 | 내부 OSPF Cost 무시. Metric 고정 |
| E1 | External + Internal Cost | 경로 길이에 영향 받음. 최적 경로 선택 가능 |

```
E2: Total Cost = External Metric (고정)
    → 어느 라우터에서 보든 Metric 동일

E1: Total Cost = External Metric + Internal OSPF Cost
    → 라우터 위치에 따라 Metric 달라짐
```

### 재분배 흐름

```
  [R1]
  Lo0: 192.168.1.0/24   → redistribute → OSPF E2, metric 99
  Lo1: 192.168.11.0/24  → redistribute → OSPF E1, metric 기본값

  [R2] 라우팅 테이블:
  O E2  192.168.1.0/24  [110/99]  via 10.1.1.1
  O E1  192.168.11.0/24 [110/30]  via 10.1.1.1
```

---

### 1단계. OSPF에서 Loopback 네트워크 제거

```
R1(config)# router ospf 1
R1(config-router)# no network 192.168.1.0 0.0.0.255 area 0
R1(config-router)# no network 192.168.11.0 0.0.0.255 area 0
```

### 2단계. ACL 생성

```
R1(config)# access-list 10 permit 192.168.1.0 0.0.0.255
R1(config)# access-list 11 permit 192.168.11.0 0.0.0.255
```

### 3단계. Route-map 생성

```
R1(config)# route-map RE permit 10
R1(config-route-map)# match ip address 10
R1(config-route-map)# set metric 99
R1(config-route-map)# exit

R1(config)# route-map RE permit 20
R1(config-route-map)# match ip address 11
R1(config-route-map)# set metric-type type-1
R1(config-route-map)# exit
```

- sequence 10: 192.168.1.0/24를 Metric 99로 재분배 (E2 기본)
- sequence 20: 192.168.11.0/24를 E1 타입으로 재분배

### 4단계. Redistribution에 Route-map 적용

```
R1(config)# router ospf 1
R1(config-router)# redistribute connected subnets route-map RE
```

### 5단계. 결과 확인

R2에서 라우팅 테이블 확인:
```
R2# show ip route
```

```
O E2  192.168.1.0/24  [110/99]  via 10.1.1.1   ← Metric 99 고정
O E1  192.168.11.0/24 [110/30]  via 10.1.1.1   ← Internal Cost 포함
```

R5에서 확인 (거리가 먼 라우터):
```
R5# show ip route
```

```
O E2  192.168.1.0/24  [110/99]  via 10.1.45.1  ← Metric 변함없음 (99 고정)
O E1  192.168.11.0/24 [110/50]  via 10.1.45.1  ← Internal Cost 추가로 증가
```

E2는 어느 라우터에서 보든 Metric이 99로 동일하고, E1은 라우터 위치에 따라 Internal Cost가 추가되어 Metric이 달라진다.

---

## Route-map 활용 정리

Route-map은 다양한 상황에서 사용된다:

| 활용 | 설명 |
|------|------|
| PBR | 출발지/프로토콜 기반으로 경로 변경 |
| Redistribution Metric 제어 | 재분배 시 Metric, Metric Type 지정 |
| Route Filtering | 특정 경로만 재분배하거나 차단 |
| BGP Policy | BGP에서 경로 속성(AS-path, Community 등) 조작 |

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show ip policy` | PBR이 적용된 인터페이스 확인 |
| `show route-map` | Route-map 설정 및 매칭 통계 |
| `show ip route` | 라우팅 테이블 (O E1/E2 확인) |
| `show ip ospf database external` | OSPF 외부 경로 LSA 확인 |
| `show access-lists` | ACL 매칭 확인 |
| `traceroute [IP] source [source IP]` | 출발지 지정 경로 추적 |

---

## PBR 주의사항

| 항목 | 설명 |
|------|------|
| CPU 부하 | PBR은 라우팅 테이블 조회 전에 정책 검사를 수행하므로 CPU 사용량 증가 가능 |
| 대규모 환경 | TCAM 기반 Hardware PBR을 지원하는 장비 사용 권장 |
| 관리 복잡성 | 정책이 많아지면 Route-map 관리가 어려워짐. 문서화 필수 |
| 장애 시 | next-hop이 다운되면 해당 트래픽이 블랙홀 됨. Track과 연동 필요 |

---

## 정리

- PBR은 출발지 IP, 프로토콜, 포트 등 다양한 조건으로 라우팅 경로를 제어하는 정책 기반 라우팅
- ACL로 대상 트래픽을 정의하고, Route-map으로 next-hop을 지정하여 인터페이스에 적용
- 멀티 ISP 로드 분산, QoS 경로 분리, 보안 경유 등 실무에서 다양하게 활용
- Route-map은 PBR 외에도 OSPF 재분배 시 Metric/Metric Type 제어에 활용
- OSPF E2는 Metric 고정, E1은 Internal Cost를 포함하여 경로 위치에 따라 Metric이 달라짐
