# FHRP (First Hop Redundancy Protocols) 실습

## 개념 정리

### FHRP란
- 게이트웨이를 이중화하여 하나의 라우터가 장애가 발생해도 다른 라우터가 즉시 대체하는 무정지 경로 이중화 기술
- 여러 대의 물리적 라우터를 하나의 가상 라우터(Virtual Router)로 묶어서 PC에는 가상 IP를 게이트웨이로 설정
- PC 입장에서는 게이트웨이가 하나로 보이지만, 실제로는 뒤에서 여러 대가 이중화되어 있음

### FHRP가 필요한 이유

```
  게이트웨이 이중화 없이                  게이트웨이 이중화 적용

  [PC] ---> [R1] ---> Internet          [PC] ---> [VIP] ---> Internet
                                                    /  \
             R1 장애 시                           [R1]  [R2]
             통신 두절                            Active  Standby
                                                    |
                                               R1 장애 시
                                               R2가 즉시 대체
```

### FHRP 3종 비교

| 구분 | HSRP | VRRP | GLBP |
|------|------|------|------|
| 표준/벤더 | Cisco 전용 | IEEE 표준 (RFC 3768) | Cisco 전용 |
| Active 명칭 | Active | Master | AVG (Active Virtual Gateway) |
| Standby 명칭 | Standby | Backup | AVF (Active Virtual Forwarder) |
| 전송 프로토콜 | UDP 1985 | IP Protocol 112 | UDP 3222 |
| Multicast 주소 | 224.0.0.2 (v1) / 224.0.0.102 (v2) | 224.0.0.18 | 224.0.0.102 |
| Hello 주기 | 3초 (기본) | 1초 (기본) | 3초 (기본) |
| Preempt | 기본 비활성 (수동 설정) | 기본 활성 | 기본 비활성 |
| 로드밸런싱 | X (Active 1대만 전달) | X (Master 1대만 전달) | O (여러 라우터가 동시에 전달) |
| Virtual MAC | v1: 0000.0c07.acXX / v2: 0000.0c9f.fXXX | 0000.5e00.01XX | 0007.b400.XXYY |
| Priority 기본값 | 100 | 100 | 100 |
| 적용 환경 | Cisco 단일 벤더 | 멀티벤더 | Cisco 환경 로드밸런싱 필요 시 |

---

## 공통 실습 토폴로지

세 프로토콜 모두 동일한 토폴로지 기반으로 실습. 프로토콜만 변경하여 적용한다.

### Physical Port 기반 토폴로지

```
                         [Internet / 상위 네트워크]
                                  |
                               [R1]
                              /    \
                         s0/0        s0/1
                        /              \
                    s0/0                s0/1
                   [R2]                [R3]
                   f0/0                f0/0
              192.168.10.100      192.168.10.200
                     \                /
                      \    VIP      /
                       \192.168.10.1/
                        \         /
                    +----+-------+----+
                    |     L2 Switch   |
                    +----+-------+----+
                         |       |
                       [PC1]   [PC2]
                    .10.10     .10.20
                    GW: 192.168.10.1 (VIP)
```

### VLAN 인터페이스 기반 토폴로지 (L3 Switch)

```
                         [Uplink / 상위 네트워크]
                                  |
                            +-----+-----+
                            |           |
                         Gi0/1       Gi0/1
                   +------+---+  +---+------+
                   |   MLS1   |  |   MLS2   |
                   | (Active) |  |(Standby) |
                   | VLAN 10  |  | VLAN 10  |
                   | SVI .2   |  | SVI .3   |
                   +----+-----+  +-----+----+
                        |              |
                     (trunk)        (trunk)
                        |              |
                   +----+--------------+----+
                   |       L2 Switch        |
                   +----+-------+-------+---+
                        |       |       |
                      [PC1]   [PC2]   [PC3]
                      VLAN 10 전부
                      GW: 192.168.10.1 (VIP)
```

### IP 주소 구성 (공통)

| 장비 | 인터페이스 | IP 주소 | 역할 |
|------|-----------|---------|------|
| R2 (또는 MLS1) | f0/0 (또는 VLAN 10) | 192.168.10.100/24 | Active/Master/AVG |
| R3 (또는 MLS2) | f0/0 (또는 VLAN 10) | 192.168.10.200/24 | Standby/Backup/AVF |
| VIP | - | 192.168.10.1 | 가상 게이트웨이 |
| R1 | s0/0 | R2 방향 WAN | 상위 라우터 |
| R1 | s0/1 | R3 방향 WAN | 상위 라우터 |
| PC1 | - | 192.168.10.10/24 | GW: 192.168.10.1 |
| PC2 | - | 192.168.10.20/24 | GW: 192.168.10.1 |

---

# 1. HSRP (Hot Standby Router Protocol)

## HSRP 개념

### 동작 원리
- 여러 대의 물리적 라우터를 하나의 가상 라우터(Virtual Router)로 묶음
- 그룹 내에서 Active 라우터 1대가 가상 IP로 오는 트래픽을 전달
- Active 라우터 장애 시 Standby 라우터가 역할을 인수

### HSRP 구성 요소

| 구성 요소 | 역할 |
|----------|------|
| Active Router | 가상 IP로 오는 트래픽을 실제로 전달하는 라우터 |
| Standby Router | Active 장애 시 즉시 역할을 인수할 백업 라우터 |
| Virtual Router | 실제 장비가 아닌 HSRP 그룹 개념의 가상 라우터 |
| Member Router | Active/Standby 아닌 나머지. 모니터링만 수행 |

### HSRP Active 선출 기준
1. Priority가 높은 라우터 (기본값 100)
2. Priority가 같으면 IP 주소가 높은 라우터
3. 먼저 기동된 라우터

### HSRP 상태 전이

```
Initial → Learn → Listen → Speak → Standby → Active

(1) Initial   : HSRP 시작. 가상 IP를 학습하는 단계
(2) Learn     : Active로부터 Hello를 아직 수신하지 못한 상태
(3) Listen    : Active/Standby가 이미 존재함을 인지. 선출에 참여하지 않음
(4) Speak     : Hello 패킷을 송신하며 Active/Standby 선출에 참여
(5) Standby   : Active 장애 시 즉시 인수할 준비 상태
(6) Active    : 가상 IP를 소유하고 패킷을 포워딩하는 상태
```

### HSRP 타이머

| 타이머 | 기본값 | 설명 |
|--------|--------|------|
| Hello | 3초 | Hello 패킷 전송 주기 |
| Hold | 10초 | Hello를 수신하지 못하면 Active 장애로 판단 |

### Preempt와 Track

- **Preempt**: Priority가 더 높은 라우터가 나중에 복구되었을 때 Active를 다시 가져오는 기능. 기본 비활성이므로 수동으로 켜야 함
- **Track**: 특정 인터페이스(업링크 등)의 상태를 감시하여, 해당 인터페이스가 다운되면 Priority를 낮추는 기능. Track 없으면 업링크가 죽어도 Active를 유지하게 되어 블랙홀 발생

```
Track 동작 흐름:

  정상 상태                    업링크 장애 발생
  [R2] Priority: 110          [R2] Priority: 90 (110-20)
  Active                       → Standby로 강등
       |                            |
  [R3] Priority: 100          [R3] Priority: 100
  Standby                      → Active로 승격
```

---

## HSRP 실습 1: Physical Port 기반

### R2 설정 (Standby)

```
conf t
hostname R2

interface f0/0
 ip address 192.168.10.100 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 100
 standby 10 authentication md5 key-string HSRPKEY
 standby 10 timers 3 10

interface s0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.12.1
end
```

### R3 설정 (Active + Preempt + Track)

```
conf t
hostname R3

interface f0/0
 ip address 192.168.10.200 255.255.255.0
 no shutdown
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
 standby 10 authentication md5 key-string HSRPKEY
 standby 10 timers 3 10
 standby 10 track s0/1 20

interface s0/1
 ip address 10.0.13.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.13.1
end
```

### R1 설정 (상위 라우터)

```
conf t
hostname R1

interface s0/0
 ip address 10.0.12.1 255.255.255.252
 no shutdown

interface s0/1
 ip address 10.0.13.1 255.255.255.252
 no shutdown

ip route 192.168.10.0 255.255.255.0 10.0.12.2
ip route 192.168.10.0 255.255.255.0 10.0.13.2
end
```

### PC 설정

```
PC1> ip 192.168.10.10 255.255.255.0 192.168.10.1
PC2> ip 192.168.10.20 255.255.255.0 192.168.10.1
```

---

## HSRP 실습 2: VLAN 인터페이스 기반 (SVI)

L3 스위치에서 SVI(Switch Virtual Interface)에 HSRP를 설정하는 방식.

### MLS1 (Active)

```
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

### MLS2 (Standby)

```
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

---

## HSRP 장애 테스트

### Active 라우터 인터페이스 shutdown

```
R3(config)# interface f0/0
R3(config-if)# shutdown
```

R2에서 확인:
```
show standby brief
```

R2가 Active로 승격되어야 정상.

### Active 복구 후 Preempt 동작

```
R3(config)# interface f0/0
R3(config-if)# no shutdown
```

Preempt가 설정되어 있으므로 R3(Priority 110)이 다시 Active를 가져옴. Preempt가 없으면 R2가 Active를 유지.

---

## HSRP 확인 명령어

```
show standby brief
show standby
show ip interface brief
```

출력 예시:
```
Interface   Grp  Pri  P State    Active         Standby        Virtual IP
Fa0/0       10   110  P Active   local          192.168.10.100 192.168.10.1
```

- P: Preempt 활성
- State: Active 또는 Standby
- Virtual IP: 가상 게이트웨이

---

# 2. VRRP (Virtual Router Redundancy Protocol)

## VRRP 개념

### HSRP와의 주요 차이점
- IEEE 표준 프로토콜이라 멀티벤더 환경에서 사용 가능
- Master(= HSRP의 Active)가 Backup(= HSRP의 Standby)에게 단방향으로 Advertisement 전송
- Preempt가 기본 활성화되어 있음
- 실제 인터페이스 IP를 가상 IP로 사용할 수 있음 (IP Address Owner)
- Hello 주기 기본 1초 (HSRP는 3초)

### VRRP Master 선출 기준
1. Virtual IP와 실제 IP가 동일한 라우터 (IP Address Owner) → 무조건 Master
2. Priority가 높은 라우터 (기본값 100, 범위 1~254)
3. 인터페이스 IP 주소가 높은 라우터

### VRRP Virtual MAC
```
0000.5e00.01XX
              ──
               └─ VRRP 그룹 번호 (16진수)

예: 그룹 10 = 0000.5e00.010a
```

### VRRP 타이머

| 타이머 | 기본값 | 설명 |
|--------|--------|------|
| Advertisement | 1초 | Master → Backup 전송 주기 |
| Master Down | 약 3.6초 | 이 시간 동안 Advertisement를 못 받으면 Master 장애로 판단 |

---

## VRRP 실습

### R2 설정 (초기 Backup, Priority 높여서 Master로 변경)

```
conf t
hostname R2

interface f0/0
 ip address 192.168.10.100 255.255.255.0
 no shutdown
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 150
 vrrp 10 authentication md5 key-string AWS

interface s0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.12.1
end
```

### R3 설정 (초기 Master → Backup으로 변경됨)

```
conf t
hostname R3

interface f0/0
 ip address 192.168.10.200 255.255.255.0
 no shutdown
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 100
 vrrp 10 authentication md5 key-string AWS

interface s0/1
 ip address 10.0.13.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.13.1
end
```

### 확인

```
show vrrp
show vrrp brief
```

R2 출력 예시:
```
FastEthernet0/0 - Group 10
  State is Master
  Virtual IP address is 192.168.10.1
  Virtual MAC address is 0000.5e00.010a
  Advertisement interval is 1.000 sec
  Preemption enabled
  Priority is 150
```

R3 출력 예시:
```
FastEthernet0/0 - Group 10
  State is Backup
  Virtual IP address is 192.168.10.1
  Master Router is 192.168.10.100, priority is 150
```

VRRP에서는 Backup 정보가 별도로 출력되지 않고, Master에 대한 정보만 표시됨.

---

## VRRP Track 설정

R2에서 업링크(s0/0) 장애 시 Priority를 60만큼 낮춰 Master를 R3에 넘기는 설정.

```
conf t
track 10 interface s0/0 line-protocol

interface f0/0
 vrrp 10 track 10 decrement 60
end
```

업링크 장애 시:
```
Priority: 150 → 90 (150-60)
R3(Priority 100)이 Master로 승격
```

### VRRP Timer 변경 및 학습

Master의 Advertisement 주기를 변경하면 Backup에도 반영할 수 있다.

**R2 (Master)**
```
interface f0/0
 vrrp 10 timers advertise 3
```

**R3 (Backup) - Master의 타이머를 학습**
```
interface f0/0
 vrrp 10 timers learn
```

R3에서 `show vrrp` 확인 시 `Master Advertisement interval is 3.000 sec`으로 표시되며 `Learning`이라고 표기됨.

### IP Address Owner

실제 인터페이스 IP를 가상 IP와 동일하게 설정하면 해당 라우터가 무조건 Master가 됨:

```
R3(config)# interface f0/0
R3(config-if)# ip address 192.168.10.1 255.255.255.0
```

이 경우 Priority에 관계없이 R3이 Master. Track 옵션은 비활성화됨.

---

# 3. GLBP (Gateway Load Balancing Protocol)

## GLBP 개념

### HSRP/VRRP와의 핵심 차이
- HSRP와 VRRP는 Active/Master 1대만 트래픽을 전달하므로 나머지 라우터가 유휴 상태
- GLBP는 여러 라우터가 동시에 트래픽을 분산 처리 (로드밸런싱)
- 하나의 가상 IP에 여러 개의 가상 MAC을 할당하여 PC별로 다른 라우터를 게이트웨이로 사용

### GLBP 구성 요소

| 구성 요소 | 역할 |
|----------|------|
| AVG (Active Virtual Gateway) | 가상 IP를 관리하고 ARP 응답을 담당. 각 PC에게 서로 다른 가상 MAC을 응답 |
| AVF (Active Virtual Forwarder) | 실제로 트래픽을 포워딩하는 라우터. 최대 4대까지 가능 |

### GLBP 로드밸런싱 동작 원리

```
  [PC1] ARP 요청: 192.168.10.1의 MAC은?
     |
     v
  [AVG] 응답: MAC = 0007.b400.0a01 → R2의 가상 MAC
     
  [PC2] ARP 요청: 192.168.10.1의 MAC은?
     |
     v
  [AVG] 응답: MAC = 0007.b400.0a02 → R3의 가상 MAC

  → 같은 가상 IP인데 PC마다 다른 MAC을 응답
  → 자연스럽게 트래픽이 R2, R3에 분산됨
```

### GLBP Virtual MAC 구조

```
0007.b400.XXYY
          ││││
          ││└└─ Forwarder 번호 (01, 02, 03, 04)
          └└─── GLBP 그룹 번호
```

### GLBP 로드밸런싱 방식

| 방식 | 설명 |
|------|------|
| Round-Robin (기본) | ARP 요청마다 순서대로 다른 AVF의 MAC을 응답 |
| Weighted | 라우터별 가중치에 따라 트래픽 비율 조절 |
| Host-Dependent | 출발지 MAC 기준으로 항상 같은 AVF에 매핑 |

### GLBP 타이머

| 타이머 | 기본값 |
|--------|--------|
| Hello | 3초 |
| Hold | 10초 |

---

## GLBP 실습

### 토폴로지 (GLBP 로드밸런싱)

```
                         [R1 - 상위]
                          /        \
                     s0/0            s0/1
                    /                  \
               s0/0                    s0/1
              [R2]                    [R3]
           f0/0 .100               f0/0 .200
          AVG + AVF1               AVF2
                \                    /
                 \   VIP: .1       /
                  \              /
              +----+------------+----+
              |      L2 Switch       |
              +----+------+-----+---+
                   |      |     |
                 [PC1]  [PC2] [PC3]
                  .10    .20   .30
                  GW: 192.168.10.1

              PC1 → R2의 가상 MAC으로 통신
              PC2 → R3의 가상 MAC으로 통신
              PC3 → R2의 가상 MAC으로 통신 (Round-Robin)
```

### R2 설정 (AVG + AVF)

```
conf t
hostname R2

interface f0/0
 ip address 192.168.10.100 255.255.255.0
 no shutdown
 glbp 10 ip 192.168.10.1
 glbp 10 priority 110
 glbp 10 preempt
 glbp 10 load-balancing round-robin
 glbp 10 authentication md5 key-string GLBPKEY
 glbp 10 timers 3 10

interface s0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.12.1
end
```

### R3 설정 (AVF)

```
conf t
hostname R3

interface f0/0
 ip address 192.168.10.200 255.255.255.0
 no shutdown
 glbp 10 ip 192.168.10.1
 glbp 10 priority 100
 glbp 10 preempt
 glbp 10 load-balancing round-robin
 glbp 10 authentication md5 key-string GLBPKEY
 glbp 10 timers 3 10

interface s0/1
 ip address 10.0.13.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.13.1
end
```

### R1 설정 (상위 라우터)

```
conf t
hostname R1

interface s0/0
 ip address 10.0.12.1 255.255.255.252
 no shutdown

interface s0/1
 ip address 10.0.13.1 255.255.255.252
 no shutdown

ip route 192.168.10.0 255.255.255.0 10.0.12.2
ip route 192.168.10.0 255.255.255.0 10.0.13.2
end
```

### PC 설정

```
PC1> ip 192.168.10.10 255.255.255.0 192.168.10.1
PC2> ip 192.168.10.20 255.255.255.0 192.168.10.1
PC3> ip 192.168.10.30 255.255.255.0 192.168.10.1
```

### 확인

```
show glbp
show glbp brief
```

R2 출력 예시 (AVG + AVF):
```
FastEthernet0/0 - Group 10
  State is Active
    2 state changes, last state change 00:05:32
  Virtual IP address is 192.168.10.1
  Hello time 3 sec, hold time 10 sec
  Priority 110 (configured)
  Preemption enabled
  Active is local
  Standby is 192.168.10.200, priority 100
  Load balancing: round-robin
  Group members:
    192.168.10.100 (local)
    192.168.10.200

  Forwarder 1
    State is Active
    MAC address is 0007.b400.0a01
  Forwarder 2
    State is Listen
    MAC address is 0007.b400.0a02
```

### 로드밸런싱 방식 변경

```
-- 라운드 로빈 (기본)
glbp 10 load-balancing round-robin

-- 가중치 기반
glbp 10 load-balancing weighted
glbp 10 weighting 200        -- 가중치 설정 (높을수록 더 많은 트래픽)

-- 호스트 고정
glbp 10 load-balancing host-dependent
```

---

## GLBP Track 설정

```
conf t
track 10 interface s0/0 line-protocol

interface f0/0
 glbp 10 weighting track 10 decrement 50
end
```

업링크 장애 시 가중치가 낮아져 해당 라우터로의 트래픽 비율이 줄어듦.

---

# FHRP 프로토콜별 설정 명령어 비교

| 항목 | HSRP | VRRP | GLBP |
|------|------|------|------|
| 가상 IP | `standby [grp] ip [VIP]` | `vrrp [grp] ip [VIP]` | `glbp [grp] ip [VIP]` |
| Priority | `standby [grp] priority [값]` | `vrrp [grp] priority [값]` | `glbp [grp] priority [값]` |
| Preempt | `standby [grp] preempt` | 기본 활성 | `glbp [grp] preempt` |
| 타이머 | `standby [grp] timers [hello] [hold]` | `vrrp [grp] timers advertise [초]` | `glbp [grp] timers [hello] [hold]` |
| 인증 | `standby [grp] authentication md5 key-string [키]` | `vrrp [grp] authentication md5 key-string [키]` | `glbp [grp] authentication md5 key-string [키]` |
| Track | `standby [grp] track [인터페이스] [감소값]` | `vrrp [grp] track [obj] decrement [값]` | `glbp [grp] weighting track [obj] decrement [값]` |
| 로드밸런싱 | - | - | `glbp [grp] load-balancing [방식]` |
| 상태 확인 | `show standby brief` | `show vrrp brief` | `show glbp brief` |

---

## 실무에서 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| Active 복구 후 Standby로 남음 | Preempt 미설정 (HSRP/GLBP) | `preempt` 설정 |
| 업링크 다운인데 Active 유지 | Track 미설정 | Track으로 업링크 감시 |
| Active/Standby 형성 안 됨 | 인증 키 불일치 | 양쪽 동일한 key-string |
| 두 대 모두 Active | 네트워크 단절로 서로 상대를 못 봄 | 물리 연결 확인 |
| VRRP Priority 변경해도 Master 안 바뀜 | IP Address Owner가 존재 | Owner의 IP를 변경하거나 제거 |

---

## 정리

- HSRP는 Cisco 전용으로 Active/Standby 구조. Preempt와 Track을 수동 설정해야 함
- VRRP는 IEEE 표준으로 멀티벤더 환경에 적합. Preempt가 기본 활성이고 IP Address Owner 개념이 있음
- GLBP는 Cisco 전용이지만 유일하게 로드밸런싱을 지원. AVG가 ARP 응답 시 여러 AVF의 MAC을 분배하여 트래픽을 분산
- 세 프로토콜 모두 Track을 설정해야 업링크 장애 시 정상적인 Failover가 이루어짐
- 실무에서는 Cisco 환경이면 HSRP, 멀티벤더면 VRRP, 로드밸런싱이 필요하면 GLBP를 선택
