# GRE over IPsec VPN 실습

## 개념 정리

### GRE (Generic Routing Encapsulation)
- 라우터 간에 가상 터널을 생성하여 서로 다른 네트워크를 논리적으로 직접 연결하는 터널링 프로토콜
- 원본 패킷을 GRE 헤더로 감싸서 터널을 통해 전달하고, 상대측에서 GRE 헤더를 벗겨 원본 패킷을 복원
- 멀티캐스트, 브로드캐스트, 다양한 프로토콜을 캡슐화할 수 있어 라우팅 프로토콜(OSPF, EIGRP 등)도 터널 위에서 동작 가능
- 단점: GRE 자체에는 암호화 기능이 없음

### IPsec (IP Security)
- IP 패킷을 암호화하고 무결성을 검증하는 보안 프로토콜
- 단독으로는 멀티캐스트/브로드캐스트를 지원하지 않아 라우팅 프로토콜을 직접 전달할 수 없음
- GRE와 결합하면 터널링(GRE) + 암호화(IPsec) 두 가지를 모두 확보할 수 있음

### GRE over IPsec
- GRE 터널로 네트워크를 연결하고, IPsec으로 터널 트래픽을 암호화하는 구성
- Site-to-Site VPN에서 가장 일반적으로 사용되는 조합

### IPsec 협상 단계

```
IKE Phase 1 (ISAKMP)                   IKE Phase 2 (IPsec)
+---------------------------+           +---------------------------+
| 양쪽 라우터가 서로 인증    |           | 실제 데이터 암호화 방식    |
| 보안 채널(SA) 수립         |  ------> | Transform Set 적용         |
| Pre-Shared Key 교환       |           | Crypto Map으로 정책 적용   |
+---------------------------+           +---------------------------+
```

### IPsec 주요 용어

| 용어 | 설명 |
|------|------|
| ISAKMP | VPN 협상 프로토콜. Phase 1에서 보안 채널을 수립 |
| Pre-Shared Key | 양쪽 라우터가 사전에 공유하는 인증용 키 |
| Transform Set | 암호화 알고리즘을 정의 (예: AES-256) |
| Crypto Map | IPsec 정책을 묶어서 인터페이스에 적용하는 설정 |
| SA (Security Association) | 양쪽 간에 합의된 보안 매개변수 집합 |
| Interesting Traffic | IPsec으로 암호화할 대상 트래픽 (ACL로 정의) |

### GRE over IPsec 패킷 구조

```
원본 패킷:
+--------+--------+------+
| IP Hdr | TCP/UDP| Data |
| (원본) | Header |      |
+--------+--------+------+

GRE 캡슐화 후:
+--------+--------+--------+--------+------+
| New IP | GRE    | IP Hdr | TCP/UDP| Data |
| Header | Header | (원본) | Header |      |
+--------+--------+--------+--------+------+

IPsec 암호화 후:
+--------+--------+==========================================+
| New IP | ESP    | 암호화된 (GRE Header + 원본 IP + Data)    |
| Header | Header |                                          |
+--------+--------+==========================================+
```

---

## 실습 토폴로지

```
  [PC1]                                                           [PC2]
192.168.10.10                                                   192.168.20.10
     |                                                               |
   [SW1]                                                           [SW2]
     |                                                               |
   g0/0                                                            g0/0
  [R1] --------s0/0-------- [R2 (ISP)] --------s0/1-------- [R3]
192.168.10.1    10.10.10.1   10.10.10.2   10.10.20.2   10.10.20.1   192.168.20.1
     |                                                               |
     +===================== GRE Tunnel ============================+
     tunnel1: 172.16.10.1                           tunnel1: 172.16.10.2
                            172.16.10.0/30
                      (IPsec으로 암호화된 터널)
```

### 상세 연결도

```
      LAN1                    WAN                     LAN2
  192.168.10.0/24      10.10.10.0/30             192.168.20.0/24
                       10.10.20.0/30

  [PC1]              [R1]          [R2]          [R3]              [PC2]
   .10    g0/0  .1  s0/0  .1    .2  s0/0  s0/1  .2  s0/1  .1  g0/0    .10
    |------+------+--------+------+------+------+--------+------+---------|
                  |                  ISP                  |
                  |                                       |
                  +========= GRE Tunnel (172.16.10.0/30) =+
                  |       tunnel1: .1          tunnel1: .2|
                  |                                       |
                  +========= IPsec 암호화 =================+
```

### IP 주소 구성

**라우터**

| 라우터 | 인터페이스 | IP 주소 | 용도 |
|--------|-----------|---------|------|
| R1 | g0/0 | 192.168.10.1/24 | LAN1 게이트웨이 |
| R1 | s0/0 | 10.10.10.1/30 | WAN (R2 방향) |
| R1 | tunnel1 | 172.16.10.1/30 | GRE 터널 |
| R2 | s0/0 | 10.10.10.2/30 | WAN (R1 방향) |
| R2 | s0/1 | 10.10.20.2/30 | WAN (R3 방향) |
| R3 | g0/0 | 192.168.20.1/24 | LAN2 게이트웨이 |
| R3 | s0/1 | 10.10.20.1/30 | WAN (R2 방향) |
| R3 | tunnel1 | 172.16.10.2/30 | GRE 터널 |

**PC**

| PC | IP | Gateway |
|----|----|---------|
| PC1 | 192.168.10.10/24 | 192.168.10.1 |
| PC2 | 192.168.20.10/24 | 192.168.20.1 |

---

## 1단계. 라우터 인터페이스 IP 설정

**R1**
```
enable
conf t
hostname R1

interface g0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface s0/0
 ip address 10.10.10.1 255.255.255.252
 no shutdown
end
```

**R2 (ISP 역할)**
```
enable
conf t
hostname R2

interface s0/0
 ip address 10.10.10.2 255.255.255.252
 no shutdown

interface s0/1
 ip address 10.10.20.2 255.255.255.252
 no shutdown
end
```

**R3**
```
enable
conf t
hostname R3

interface g0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown

interface s0/1
 ip address 10.10.20.1 255.255.255.252
 no shutdown
end
```

---

## 2단계. 정적 라우팅

각 라우터가 상대 LAN 네트워크에 도달할 수 있도록 경로를 설정한다.

**R1**
```
ip route 192.168.20.0 255.255.255.0 10.10.10.2
```

**R2**
```
ip route 192.168.10.0 255.255.255.0 10.10.10.1
ip route 192.168.20.0 255.255.255.0 10.10.20.1
```

**R3**
```
ip route 192.168.10.0 255.255.255.0 10.10.20.2
```

이 시점에서 PC1 → PC2 ping이 성공해야 다음 단계로 진행 가능.

---

## 3단계. GRE 터널 생성

R1과 R3 사이에 가상 터널 인터페이스를 생성한다. R2(ISP)는 터널에 관여하지 않음.

**R1**
```
interface tunnel1
 ip address 172.16.10.1 255.255.255.252
 tunnel source 10.10.10.1
 tunnel destination 10.10.20.1
 no shutdown
```

**R3**
```
interface tunnel1
 ip address 172.16.10.2 255.255.255.252
 tunnel source 10.10.20.1
 tunnel destination 10.10.10.1
 no shutdown
```

- `tunnel source`: 터널의 출발지 IP (자신의 WAN IP)
- `tunnel destination`: 터널의 목적지 IP (상대의 WAN IP)

확인:
```
show ip interface brief
```
tunnel1 인터페이스가 up/up 상태여야 함.

```
R1# ping 172.16.10.2
```
터널 IP로 ping이 성공하면 GRE 터널 정상.

---

## 4단계. Interesting Traffic 정의 (ACL)

IPsec으로 암호화할 트래픽을 ACL로 지정한다.

**R1**
```
access-list 100 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
```

**R3**
```
access-list 100 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```

양쪽의 출발지/목적지가 서로 반대임에 주의.

---

## 5단계. ISAKMP Policy (IKE Phase 1)

VPN 협상 규칙을 설정한다. Phase 1에서 보안 채널을 수립.

**R1**
```
crypto isakmp policy 10
 authentication pre-share
exit
```

**R3**
```
crypto isakmp policy 10
 authentication pre-share
exit
```

`pre-share`: 사전 공유 키(Pre-Shared Key) 방식으로 상호 인증.

---

## 6단계. Pre-Shared Key 설정

양쪽 라우터가 동일한 키를 공유해야 한다.

**R1**
```
crypto isakmp key 0 aws address 10.10.20.1
```

**R3**
```
crypto isakmp key 0 aws address 10.10.10.1
```

- `aws`: 공유 키 문자열
- `address`: 상대 라우터의 WAN IP

---

## 7단계. IPsec Transform Set

암호화 알고리즘을 정의한다.

**R1**
```
crypto ipsec transform-set IPSEC esp-aes 256
```

**R3**
```
crypto ipsec transform-set IPSEC esp-aes 256
```

AES-256 암호화를 사용. Transform Set 이름(IPSEC)은 양쪽에서 동일하지 않아도 되지만, 알고리즘은 일치해야 함.

---

## 8단계. Crypto Map 설정

VPN 정책을 하나로 묶는 설정.

**R1**
```
crypto map GRE 10 ipsec-isakmp
 set peer 10.10.20.1
 set transform-set IPSEC
 match address 100
exit
```

**R3**
```
crypto map GRE 10 ipsec-isakmp
 set peer 10.10.10.1
 set transform-set IPSEC
 match address 100
exit
```

- `set peer`: 상대 라우터 WAN IP
- `set transform-set`: 암호화 방식
- `match address 100`: ACL 100에 매칭되는 트래픽을 암호화

---

## 9단계. Crypto Map 적용

IPsec 정책을 터널 인터페이스에 적용한다.

**R1**
```
interface tunnel1
 crypto map GRE
```

**R3**
```
interface tunnel1
 crypto map GRE
```

---

## 10단계. 동작 확인

### ISAKMP SA 확인 (Phase 1)
```
show crypto isakmp sa
```

정상이면 State가 `QM_IDLE` (협상 완료 상태).

### IPsec SA 확인 (Phase 2)
```
show crypto ipsec sa
```

`#pkts encaps`와 `#pkts decaps` 값이 증가하면 암호화/복호화가 정상 동작 중.

### 터널 상태 확인
```
show ip interface brief
show interfaces tunnel1
```

---

## 11단계. 통신 테스트

```
PC1> ping 192.168.20.10
```

ping 성공 후 `show crypto ipsec sa`에서 패킷 카운터가 증가하는지 확인.

---

## 패킷 흐름 상세

```
PC1 (192.168.10.10)이 PC2 (192.168.20.10)에게 ping:

  [PC1] 원본 패킷 생성
  src: 192.168.10.10, dst: 192.168.20.10
     |
     v
  [R1] 라우팅 테이블 확인 → tunnel1로 전달
     |
     | (1) GRE 캡슐화
     |     원본 패킷을 GRE 헤더로 감쌈
     |     새 IP: src 10.10.10.1, dst 10.10.20.1
     |
     | (2) IPsec 암호화
     |     GRE 패킷 전체를 ESP로 암호화
     |
     v
  [R2 - ISP] 외부 IP 헤더만 보고 라우팅
     |        (내부 내용은 암호화되어 볼 수 없음)
     v
  [R3] 수신
     |
     | (3) IPsec 복호화
     |     ESP 헤더 제거, GRE 패킷 복원
     |
     | (4) GRE 디캡슐화
     |     GRE 헤더 제거, 원본 패킷 복원
     |
     v
  [PC2] 수신
  src: 192.168.10.10, dst: 192.168.20.10
```

---

## 설정 흐름 요약

```
1. 인터페이스 IP 설정
        ↓
2. 정적 라우팅 (양쪽 LAN 도달 가능하게)
        ↓
3. GRE 터널 생성 (tunnel source/destination)
        ↓
4. ACL로 암호화 대상 트래픽 정의
        ↓
5. ISAKMP Policy (Phase 1 협상 규칙)
        ↓
6. Pre-Shared Key 설정
        ↓
7. Transform Set (암호화 알고리즘)
        ↓
8. Crypto Map (정책 묶기)
        ↓
9. Crypto Map을 터널에 적용
        ↓
10. 확인 및 테스트
```

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show crypto isakmp sa` | IKE Phase 1 SA 상태 확인 |
| `show crypto ipsec sa` | IPsec Phase 2 SA 상태 (암호화/복호화 카운터) |
| `show interfaces tunnel1` | 터널 인터페이스 상태 |
| `show ip interface brief` | 전체 인터페이스 상태 |
| `show ip route` | 라우팅 테이블 |
| `show access-lists` | ACL 매칭 확인 |

---

## 주요 기술 요약

| 기술 | 역할 |
|------|------|
| GRE | 터널 프로토콜. 패킷을 캡슐화하여 가상 연결 생성 |
| IPsec | 암호화. 터널 트래픽을 AES 등으로 암호화 |
| ISAKMP | VPN 협상 프로토콜. 보안 채널 수립 |
| ACL | Interesting Traffic. 암호화 대상 트래픽 정의 |
| Crypto Map | IPsec 정책을 인터페이스에 적용 |
| Pre-Shared Key | 양쪽 라우터 간 사전 공유 인증 키 |
| Transform Set | 암호화 알고리즘 정의 (예: AES-256) |

---

## 정리

- GRE 터널은 라우터 간 가상 연결을 제공하지만 암호화 기능이 없으므로, IPsec을 결합하여 보안을 확보
- IPsec은 Phase 1(ISAKMP)에서 보안 채널을 수립하고, Phase 2에서 실제 데이터 암호화를 수행
- ACL로 암호화 대상 트래픽을 정의하고, Crypto Map으로 정책을 묶어 터널 인터페이스에 적용
- 양쪽 라우터에서 Pre-Shared Key, Transform Set, ACL이 서로 매칭되어야 VPN이 정상 동작
- `show crypto ipsec sa`에서 encrypt/decrypt 카운터가 증가하면 암호화 통신이 정상적으로 이루어지고 있는 것
