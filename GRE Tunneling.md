# GRE over IPsec VPN Lab

This lab demonstrates how to configure **GRE over IPsec Site-to-Site VPN**.

- GRE → creates tunnel between routers
- IPsec → encrypts tunnel traffic

---

# Network Topology

```
PC1 ─ SW1 ─ R1 ===== R2 ===== R3 ─ SW2 ─ PC2
                WAN
```

LAN1 : 192.168.10.0/24  
LAN2 : 192.168.20.0/24  

Tunnel Network

```
172.16.10.0/30
```

---

# 1. Router Basic Configuration

라우터 인터페이스에 IP 주소 설정

## R1

```bash
enable
conf t
hostname R1

# LAN interface
interface g0/0
ip address 192.168.10.1 255.255.255.0
no shutdown

# WAN interface
interface s0/0
ip address 10.10.10.1 255.255.255.252
no shutdown
```

---

## R2

중간 ISP 역할

```bash
enable
conf t
hostname R2

interface s0/0
ip address 10.10.10.2 255.255.255.252
no shutdown

interface s0/1
ip address 10.10.20.2 255.255.255.252
no shutdown
```

---

## R3

```bash
enable
conf t
hostname R3

# LAN interface
interface g0/0
ip address 192.168.20.1 255.255.255.0
no shutdown

# WAN interface
interface s0/1
ip address 10.10.20.1 255.255.255.252
no shutdown
```

---

# 2. Static Routing

서로의 LAN 네트워크로 가기 위한 경로 설정

## R1

```bash
ip route 192.168.20.0 255.255.255.0 10.10.10.2
```

---

## R2

```bash
ip route 192.168.10.0 255.255.255.0 10.10.10.1
ip route 192.168.20.0 255.255.255.0 10.10.20.1
```

---

## R3

```bash
ip route 192.168.10.0 255.255.255.0 10.10.20.2
```

---

# 3. GRE Tunnel Configuration

GRE 터널 생성  
R1과 R3 사이에 **가상 인터페이스 생성**

## R1

```bash
interface tunnel1
ip address 172.16.10.1 255.255.255.252
tunnel source 10.10.10.1
tunnel destination 10.10.20.1
no shutdown
```

설명

- tunnel source → 터널 시작 IP
- tunnel destination → 터널 목적지

---

## R3

```bash
interface tunnel1
ip address 172.16.10.2 255.255.255.252
tunnel source 10.10.20.1
tunnel destination 10.10.10.1
no shutdown
```

---

# 4. Interesting Traffic (ACL)

IPsec으로 **암호화할 트래픽 정의**

## R1

```bash
access-list 100 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
```

설명

```
LAN1 → LAN2 트래픽을 암호화
```

---

## R3

```bash
access-list 100 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```

---

# 5. ISAKMP Policy (IKE Phase 1)

VPN 협상 규칙 설정

## R1

```bash
crypto isakmp policy 10
authentication pre-share
exit
```

설명

```
pre-share → 사전 공유 키 방식 사용
```

---

## R3

```bash
crypto isakmp policy 10
authentication pre-share
exit
```

---

# 6. Pre Shared Key

양쪽 라우터 인증용 패스워드 설정

## R1

```bash
crypto isakmp key 0 aws address 10.10.20.1
```

설명

```
aws = shared key
```

---

## R3

```bash
crypto isakmp key 0 aws address 10.10.10.1
```

---

# 7. IPsec Transform Set

암호화 방식 정의

## R1

```bash
crypto ipsec transform-set IPSEC esp-aes 256
```

설명

```
AES-256 encryption 사용
```

---

## R3

```bash
crypto ipsec transform-set IPSEC esp-aes 256
```

---

# 8. Crypto Map

VPN 정책 설정

## R1

```bash
crypto map GRE 10 ipsec-isakmp
set peer 10.10.20.1
set transform-set IPSEC
match address 100
exit
```

설명

```
peer → 상대 라우터
transform-set → 암호화 방식
ACL 100 → 암호화할 트래픽
```

---

## R3

```bash
crypto map GRE 10 ipsec-isakmp
set peer 10.10.10.1
set transform-set IPSEC
match address 100
exit
```

---

# 9. Apply Crypto Map

IPsec 정책을 터널 인터페이스에 적용

## R1

```bash
interface tunnel1
crypto map GRE
```

---

## R3

```bash
interface tunnel1
crypto map GRE
```

---

# 10. Verification

VPN 상태 확인

```
show crypto isakmp sa
```

```
show crypto ipsec sa
```

패킷 증가 확인

```
#pkts encrypt
#pkts decrypt
```

---

# 11. Test Connectivity

PC1에서 테스트

```
ping 192.168.20.10
```

---

# Packet Flow

```
PC1
 ↓
R1
 ↓
GRE Tunnel
 ↓
IPsec Encryption
 ↓
R2 (Internet)
 ↓
IPsec Decryption
 ↓
GRE Tunnel
 ↓
R3
 ↓
PC2
```

---

# Technologies

| Technology | Description |
|---|---|
| GRE | Tunnel protocol |
| IPsec | Encryption |
| ISAKMP | VPN negotiation |
| ACL | Interesting traffic |
| Crypto Map | Apply IPsec policy |

---
ISAKMP	VPN 협상 프로토콜
Transform Set	암호화 알고리즘 정의
Crypto Map	IPsec 정책 적용
