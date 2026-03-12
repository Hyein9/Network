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
