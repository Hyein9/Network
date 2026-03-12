# Policy Based Routing (PBR)

## 1. PBR (Policy Based Routing)이란?

**Policy Based Routing (PBR)**은 일반적인 **Destination 기반 라우팅**과 달리
**정책(Policy)**에 따라 패킷의 라우팅 경로를 결정하는 기술이다.

일반적인 라우팅은 다음과 같이 동작한다.

```
Destination IP → Routing Table → Next Hop 결정
```

하지만 **PBR**은 다음과 같은 다양한 조건을 기반으로 라우팅을 수행할 수 있다.

- Source IP
- Destination IP
- Protocol
- Port
- DSCP / QoS
- Interface

즉, **"누가 보내느냐 / 어떤 트래픽이냐"**에 따라 **다른 경로로 보낼 수 있다.**

---

# 2. 일반 라우팅 vs PBR

## 일반 라우팅

```
Client → Router → Routing Table → Next Hop
```

예시

| Destination | Next Hop |
|---|---|
| 0.0.0.0/0 | ISP-A |

모든 트래픽이 **ISP-A**로 나간다.

---

## PBR 라우팅

정책을 적용하면 다음처럼 동작할 수 있다.

```
Source 10.0.1.0/24 → ISP-A
Source 10.0.2.0/24 → ISP-B
```

즉 같은 목적지라도 **출발지에 따라 다른 경로 사용**

---

# 3. PBR 사용 목적

대표적인 사용 사례

### 1️⃣ 멀티 ISP 로드 분산

```
사내망 A → ISP1
사내망 B → ISP2
```

### 2️⃣ 특정 서비스 전용 회선 사용

```
VoIP → 저지연 회선
일반 인터넷 → 일반 ISP
```

### 3️⃣ 보안 정책

```
특정 서버 → 방화벽 경유
```

---

# 4. Cisco Router PBR 예제

### 네트워크 구성

```
LAN-A (10.1.1.0/24) ---- Router ---- ISP1
LAN-B (10.2.2.0/24) ---- Router ---- ISP2
```

정책

```
LAN-A → ISP1
LAN-B → ISP2
```

---

## Step 1. Access List 생성

```cisco
access-list 10 permit 10.1.1.0 0.0.0.255
access-list 20 permit 10.2.2.0 0.0.0.255
```

---

## Step 2. Route Map 생성

```cisco
route-map PBR permit 10
 match ip address 10
 set ip next-hop 1.1.1.1
```

```cisco
route-map PBR permit 20
 match ip address 20
 set ip next-hop 2.2.2.2
```

설명

```
10.1.1.0/24 → next-hop 1.1.1.1 (ISP1)
10.2.2.0/24 → next-hop 2.2.2.2 (ISP2)
```

---

## Step 3. Interface에 PBR 적용

```cisco
interface GigabitEthernet0/0
 ip policy route-map PBR
```

---

# 5. Linux PBR 예제

Linux에서는 **iproute2**를 이용하여 PBR을 구현한다.

---

## Step 1. Routing Table 추가

```bash
echo "100 isp1" >> /etc/iproute2/rt_tables
echo "200 isp2" >> /etc/iproute2/rt_tables
```

---

## Step 2. 라우팅 경로 설정

```bash
ip route add default via 1.1.1.1 table isp1
ip route add default via 2.2.2.2 table isp2
```

---

## Step 3. 정책 적용

```bash
ip rule add from 10.1.1.0/24 table isp1
ip rule add from 10.2.2.0/24 table isp2
```

---

## 결과

```
10.1.1.0/24 → ISP1
10.2.2.0/24 → ISP2
```

---

# 6. PBR 동작 흐름

```
Packet 수신
      │
      ▼
Policy Match 확인
      │
      ├─ Match → Policy Route 적용
      │
      └─ No Match → 일반 Routing Table 사용
```

---

# 7. PBR 주의사항

### 1️⃣ CPU 사용량 증가 가능
PBR은 **Routing Table lookup 전에 정책 검사**를 수행한다.

### 2️⃣ 트래픽이 많으면 성능 영향

대규모 네트워크에서는 다음을 고려한다.

```
TCAM / Hardware PBR
```

### 3️⃣ 관리 복잡성 증가

정책이 많아지면 관리가 어려워질 수 있다.

---

# 8. 요약

| 항목 | 설명 |
|---|---|
| 개념 | 정책 기반 라우팅 |
| 기존 라우팅 | Destination 기반 |
| PBR | Source / Protocol / Port 등 정책 기반 |
| 사용 목적 | ISP 분산, QoS, 보안 |

---

# 9. 참고

- Cisco Policy Based Routing
- Linux iproute2
- RFC 1812 Router Requirements

# Policy Based Routing (PBR) Lab

## Lab Overview

이 실습에서는 **Policy Based Routing (PBR)**을 이용하여  
특정 트래픽을 **기본 라우팅 경로가 아닌 다른 경로로 우회시키는 방법**을 학습한다.

기본적으로 R1 → R5 통신은 **R2 → R3 → R4** 경로를 사용하지만  
**PBR을 이용해 R2 → R4 (Serial Link)** 경로로 보내도록 구성한다.

---

# Network Topology

![topology](./topology.png)

---

# Network Addressing

| Device | Interface | IP |
|------|------|------|
| R1 | F0/0 | 10.1.1.1/24 |
| R1 | Lo0 | 192.168.1.1/24 |
| R1 | Lo1 | 192.168.11.1/24 |
| R2 | F0/0 | 10.1.1.2/24 |
| R2 | F0/1 | 10.1.23.1/24 |
| R2 | S0/0 | 10.1.24.1/24 |
| R3 | F0/1 | 10.1.23.2/24 |
| R3 | F0/0 | 10.1.34.1/24 |
| R4 | F0/1 | 10.1.34.2/24 |
| R4 | F0/0 | 10.1.45.1/24 |
| R4 | S0/0 | 10.1.24.4/24 |
| R5 | F0/0 | 10.1.45.2/24 |
| R5 | Lo0 | 172.16.5.1/24 |
| R5 | Lo1 | 172.16.55.1/24 |

---

# Goal

기본 경로

```
R1 → R2 → R3 → R4 → R5
```

PBR 적용 후

```
R1 → R2 → R4 → R5
```

즉 **R3를 우회하도록 정책 라우팅 적용**

---

# Step 1. Interface Configuration

## R1

```cisco
conf t

interface f0/0
ip address 10.1.1.1 255.255.255.0
no shutdown

interface lo0
ip address 192.168.1.1 255.255.255.0

interface lo1
ip address 192.168.11.1 255.255.255.0
```

---

## R2

```cisco
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
```

---

## R3

```cisco
conf t

interface f0/1
ip address 10.1.23.2 255.255.255.0
no shutdown

interface f0/0
ip address 10.1.34.1 255.255.255.0
no shutdown
```

---

## R4

```cisco
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
```

---

## R5

```cisco
conf t

interface f0/0
ip address 10.1.45.2 255.255.255.0
no shutdown

interface lo0
ip address 172.16.5.1 255.255.255.0

interface lo1
ip address 172.16.55.1 255.255.255.0
```

---

# Step 2. Routing Configuration

실습 단순화를 위해 **OSPF 사용**

## R1

```cisco
router ospf 1
network 10.1.1.0 0.0.0.255 area 0
network 192.168.1.0 0.0.0.255 area 0
network 192.168.11.0 0.0.0.255 area 0
```

---

## R2

```cisco
router ospf 1
network 10.1.1.0 0.0.0.255 area 0
network 10.1.23.0 0.0.0.255 area 0
network 10.1.24.0 0.0.0.255 area 0
```

---

## R3

```cisco
router ospf 1
network 10.1.23.0 0.0.0.255 area 0
network 10.1.34.0 0.0.0.255 area 0
```

---

## R4

```cisco
router ospf 1
network 10.1.24.0 0.0.0.255 area 0
network 10.1.34.0 0.0.0.255 area 0
network 10.1.45.0 0.0.0.255 area 0
```

---

## R5

```cisco
router ospf 1
network 10.1.45.0 0.0.0.255 area 0
network 172.16.5.0 0.0.0.255 area 0
network 172.16.55.0 0.0.0.255 area 0
```

---

# Step 3. 기본 경로 확인

R1에서 traceroute

```cisco
traceroute 172.16.5.1
```

결과

```
R1
R2
R3
R4
R5
```

---

# Step 4. Policy Based Routing 설정

R2에서 **R1 트래픽을 R4로 직접 보내도록 설정**

---

## ACL 생성

```cisco
access-list 10 permit 192.168.1.0 0.0.0.255
```

---

## Route-map 생성

```cisco
route-map PBR permit 10
match ip address 10
set ip next-hop 10.1.24.4
```

설명

```
R1 Loopback 트래픽 → R4로 직접 전달
```

---

## Interface에 적용

```cisco
interface f0/0
ip policy route-map PBR
```

R1에서 들어오는 트래픽에 PBR 적용

---

# Step 5. PBR 확인

```cisco
show route-map
```

```cisco
show ip policy
```

```cisco
show access-lists
```

---

# Step 6. 결과 확인

다시 traceroute

```cisco
traceroute 172.16.5.1
```

결과

```
R1
R2
R4
R5
```

R3를 우회하는 것을 확인 가능

---

# Packet Flow

Before PBR

```
R1 → R2 → R3 → R4 → R5
```

After PBR

```
R1 → R2 → R4 → R5
```

---

# Key Takeaways

PBR 특징

- Destination 기반 라우팅이 아님
- Policy 기반 라우팅
- Source / Protocol / Port 기준 가능
- Traffic Engineering 가능

사용 사례

```
Multi ISP
Security routing
QoS routing
Traffic engineering
```

---

# Verification Commands

```cisco
show ip route
show ip policy
show route-map
show access-lists
show ip cef
```

---

# Summary

| Feature | Description |
|------|------|
| Routing Type | Policy Based |
| Control Method | Route-map |
| Match Tool | ACL |
| Action | set ip next-hop |

---

# Author

Network Lab - PBR Practice


---

# OSPF Redistribution & Route-map Metric Control

이번 단계에서는 **R1의 Loopback 네트워크를 OSPF에 재분배(redistribute)** 하면서  
**Route-map을 이용하여 Metric과 Metric Type을 제어**한다.

이를 통해 **OSPF External Route의 동작(E1 / E2)**을 확인한다.

---

# Step 1 OSPF Network 제거

기존에는 Loopback을 **OSPF network 명령으로 광고**하고 있었다.

이를 제거한다.

```cisco
R1(config)#router ospf 1
R1(config-router)#no network 192.168.1.0 0.0.0.255 area 0
R1(config-router)#no network 192.168.11.0 0.0.0.255 area 0
```

설명

```
Loopback을 OSPF에서 직접 광고하지 않도록 제거
```

---

# Step 2 Connected Redistribution 설정

Loopback을 **redistribute connected** 방식으로 광고한다.

```cisco
R1(config-router)#redistribute connected subnets
```

설명

```
Connected Network → OSPF External Route로 광고
```

---

# Step 3 Route-map 생성 (Metric Control)

Route-map을 이용해 **특정 네트워크의 OSPF Metric을 조정**한다.

---

## ACL 생성

```cisco
R1(config)#access-list 10 permit 192.168.1.0 0.0.0.255
```

설명

```
R1 Loopback0
192.168.1.0/24
```

---

## Route-map 생성

```cisco
R1(config)#route-map RE permit 10
R1(config-route-map)#match ip address 10
R1(config-route-map)#set metric 99
```

설명

```
192.168.1.0/24

OSPF Metric
99
```

---

# Step 4 Redistribution에 Route-map 적용

```cisco
R1(config)#router ospf 1
R1(config-router)#redistribute connected subnets route-map RE
```

설명

```
Connected → OSPF

하지만 Route-map 조건 적용
```

---

# Step 5 두 번째 정책 추가

다른 Loopback에 대해 **Metric Type 변경**

---

## ACL 생성

```cisco
R1(config)#access-list 11 permit 192.168.11.0 0.0.0.255
```

---

## Route-map 추가

```cisco
R1(config)#route-map RE permit 20
R1(config-route-map)#match ip address 11
R1(config-route-map)#set metric-type type-1
```

설명

```
192.168.11.0/24

OSPF External Type
E1
```

---

# OSPF External Route Type

OSPF External Route에는 두 가지 타입이 있다.

---

## E1 (External Type 1)

```
Total Cost

External Metric
+
Internal OSPF Cost
```

즉

```
경로 길이에 영향 받음
```

---

## E2 (External Type 2)

```
External Metric만 사용
```

즉

```
Internal OSPF Cost 무시
```

---

# Routing Table 확인 (R2)

```
R2#show ip route
```

---

## 결과 1

```
O E2 192.168.1.0/24 [110/99] via 10.1.1.1
```

설명

```
OSPF External Type 2
Metric
99
```

---

## 결과 2

```
O E1 192.168.11.0/24 [110/30] via 10.1.1.1
```

설명

```
OSPF External Type 1
Internal Cost 포함
```

---

# Routing Table 분석

| Network | Type | Metric |
|------|------|------|
| 192.168.1.0 | O E2 | 99 |
| 192.168.11.0 | O E1 | Internal + External |

---

# Packet Flow 영향

### E2

```
Metric 고정

Topology 영향 적음
```

---

### E1

```
Internal Cost 포함

Shortest Path 영향
```

---

# 실제 Routing Table

Example

```
R2#show ip route

O E1 192.168.11.0/24 [110/30] via 10.1.1.1
O E2 192.168.1.0/24 [110/99] via 10.1.1.1
```

---

# Lab에서 확인 가능한 핵심 개념

이번 실습에서 확인한 기술

```
Policy Based Routing
OSPF Redistribution
Route-map
OSPF External Route
Metric Control
```

---

# 실무 응용

이 구조는 실제 네트워크에서 매우 많이 사용된다.

---

## 1 Multi-Protocol Routing

예

```
BGP → OSPF
Static → OSPF
EIGRP → OSPF
```

Route Redistribution 시

```
Route-map으로 정책 제어
```

---

## 2 Metric Control

특정 경로를 **우선순위 낮게 설정**

예

```
Backup Link
DR Site
```

---

## 3 Traffic Engineering

특정 네트워크를

```
Primary Path
Backup Path
```

로 제어

---

## 4 ISP Edge Design

예

```
Internet → BGP
Internal → OSPF
```

Redistribution 시

```
Metric Control 필수
```

---

# Lab Summary

이번 실습에서는 다음 기술을 동시에 구성했다.

```
PBR
OSPF
Redistribution
Route-map
Metric Control
OSPF E1 / E2
```

---

# Key Takeaway

Route-map은 다음 기능을 수행할 수 있다.

```
Routing Policy Control
Metric Manipulation
Route Filtering
Traffic Engineering
```

---

# Useful Verification Commands

```
show ip route
show ip ospf database
show ip policy
show route-map
show access-lists
```

---
