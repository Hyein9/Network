# GLBP (Gateway Load Balancing Protocol) 실습

## 개념 정리

### GLBP란
- HSRP나 VRRP와 같이 게이트웨이 이중화 기능을 제공하면서, 별도의 설정 없이 로드밸런싱 기능까지 제공하는 Cisco 전용 프로토콜
- HSRP/VRRP는 Active(Master) 1대만 트래픽을 전달하고 나머지는 대기 상태이지만, GLBP는 모든 라우터가 동시에 트래픽을 분산 처리
- 1개의 가상 IP 주소에 여러 개의 가상 MAC 주소를 사용 (그룹당 최대 4개)
- 3초마다 IP 224.0.0.102, UDP 포트 3222번으로 Hello 메시지 통신

### HSRP/VRRP vs GLBP 비교

| 구분 | HSRP / VRRP | GLBP |
|------|------------|------|
| 트래픽 전달 | Active(Master) 1대만 | 모든 라우터가 동시에 전달 |
| 로드밸런싱 | 지원 안 함 | 기본 지원 |
| 가상 IP | 1개 | 1개 |
| 가상 MAC | 1개 | 최대 4개 (AVF 수만큼) |
| 대기 라우터 활용 | 유휴 상태 (자원 낭비) | 모든 라우터가 트래픽 처리 |

```
HSRP/VRRP:                           GLBP:

  [PC1] [PC2] [PC3]                   [PC1] [PC2] [PC3]
     \    |    /                          \    |    /
      \   |   /                            \   |   /
    [VIP: .1]                            [VIP: .1]
        |                                /        \
     [Active]     [Standby]          [AVF1]      [AVF2]
     트래픽 전달   대기만 함           트래픽 전달  트래픽 전달
```

### GLBP 구성 요소

| 구성 요소 | 역할 |
|----------|------|
| AVG (Active Virtual Gateway) | 가상 IP를 관리하고 ARP 요청에 응답. 각 AVF에 가상 MAC을 할당 |
| AVF (Active Virtual Forwarder) | 실제로 트래픽을 포워딩하는 라우터. 그룹당 최대 4대 |

### AVG (Active Virtual Gateway) 상세

- 각 멤버(AVF)에게 가상 MAC 주소를 할당
- PC가 게이트웨이 IP에 대한 ARP 요청을 보내면, AVG가 각 AVF의 가상 MAC 중 하나를 응답
- ARP 응답 시 돌아가며 다른 MAC을 응답하므로 자동으로 부하 분산이 이루어짐

### AVG 선출 기준
1. GLBP Priority가 높은 라우터 (기본값 100)
2. Priority가 같으면 인터페이스 IP 주소가 높은 라우터

### GLBP Virtual MAC 구조

```
0007.b400.XXYY
          ││││
          ││└└── AVF ID (01, 02, 03, 04)
          └└──── GLBP 그룹 번호 (16진수)

예: 그룹 10, AVF 1번 = 0007.b400.0a01
    그룹 10, AVF 2번 = 0007.b400.0a02
```

### GLBP 로드밸런싱 동작 원리

```
  [PC1] ARP: 192.168.10.1의 MAC은?
     |
     v
  [AVG] 응답: 0007.b400.0a01 (R2의 가상 MAC)

  [PC2] ARP: 192.168.10.1의 MAC은?
     |
     v
  [AVG] 응답: 0007.b400.0a02 (R3의 가상 MAC)

  [PC3] ARP: 192.168.10.1의 MAC은?
     |
     v
  [AVG] 응답: 0007.b400.0a01 (R2의 가상 MAC) ← Round-Robin

  → 같은 가상 IP인데 PC마다 다른 MAC을 응답
  → PC1, PC3은 R2를 통해 통신
  → PC2는 R3을 통해 통신
  → 트래픽이 자연스럽게 분산
```

### 로드밸런싱 방식

| 방식 | 설명 |
|------|------|
| Round-Robin (기본) | ARP 요청마다 순서대로 다른 AVF의 MAC을 응답 |
| Weighted | 라우터별 가중치(weighting)에 따라 트래픽 비율 조절 |
| Host-Dependent | 출발지 MAC 기준으로 항상 같은 AVF에 매핑 (세션 유지 필요 시) |

### GLBP Weighting

AVF 역할을 수행할지 여부를 가중치로 제어한다.

| 항목 | 기본값 | 설명 |
|------|--------|------|
| Maximum | 100 | 가중치 최대값 |
| Lower Threshold | 1 | 이 값 이하로 떨어지면 AVF 역할 포기 |
| Upper Threshold | 100 | 이 값 이상으로 올라가면 AVF 역할 재개 |
| Track Decrement | 설정값 | 감시 대상 다운 시 감소할 가중치 |

```
정상:   Weighting = 100 (> Lower 1)  → AVF 역할 수행
장애:   Weighting = 40  (> Lower 1)  → AVF 역할 유지
심각:   Weighting = 0   (< Lower 1)  → AVF 역할 포기, 다른 AVF가 대체
```

### GLBP 타이머

| 타이머 | 기본값 |
|--------|--------|
| Hello | 3초 |
| Hold | 10초 |

### GLBP 상태

| 상태 | 설명 |
|------|------|
| Active | AVG 또는 AVF 역할 수행 중 |
| Standby | AVG 백업 (AVG 장애 시 승격) |
| Listen | AVG/Standby 아닌 나머지. 모니터링만 |

---

## 실습 1: 기본 GLBP 구성

### 토폴로지

```
                        [Internet / 상위 네트워크]
                                  |
                               [R1]
                              /    \
                         s0/0        s0/1
                   10.0.12.0/30    10.0.13.0/30
                        /              \
                    s0/0                s0/1
                   [R2]                [R3]
                   f0/0                f0/0
              192.168.10.100      192.168.10.200
                Priority 110         Priority 100
                AVG + AVF1            AVF2
                     \                /
                      \ VIP: .1     /
                       \          /
                   +----+--------+----+
                   |     L2 Switch    |
                   +----+--------+----+
                        |        |
                      [PC1]    [PC2]
                      .10       .20
                   GW: 192.168.10.1
```

### 상세 연결도

```
                       [R1]
                  s0/0       s0/1
                 .1            .1
                 |              |
            10.0.12.0/30   10.0.13.0/30
                 |              |
                .2             .2
              s0/0           s0/1
              [R2]           [R3]
             f0/0           f0/0
             .100           .200
               |              |
               +--- 192.168 --+
               |    .10.0/24  |
               |              |
          +----+--------------+----+
          |        L2 Switch       |
          +----+------+------+----+
               |      |      |
             [PC1]  [PC2]  [PC3]
              .10    .20    .30
```

### IP 주소 구성

| 장비 | 인터페이스 | IP 주소 | 역할 |
|------|-----------|---------|------|
| R1 | s0/0 | 10.0.12.1/30 | 상위 라우터 |
| R1 | s0/1 | 10.0.13.1/30 | 상위 라우터 |
| R2 | s0/0 | 10.0.12.2/30 | R1 연결 (업링크) |
| R2 | f0/0 | 192.168.10.100/24 | LAN (AVG + AVF1) |
| R3 | s0/1 | 10.0.13.2/30 | R1 연결 (업링크) |
| R3 | f0/0 | 192.168.10.200/24 | LAN (AVF2) |
| VIP | - | 192.168.10.1 | 가상 게이트웨이 |
| PC1 | - | 192.168.10.10/24 | GW: 192.168.10.1 |
| PC2 | - | 192.168.10.20/24 | GW: 192.168.10.1 |
| PC3 | - | 192.168.10.30/24 | GW: 192.168.10.1 |

---

### 1단계. 인터페이스 IP 설정

**R1 (상위 라우터)**
```
conf t
hostname R1

interface s0/0
 ip address 10.0.12.1 255.255.255.252
 clock rate 64000
 no shutdown

interface s0/1
 ip address 10.0.13.1 255.255.255.252
 clock rate 64000
 no shutdown

ip route 192.168.10.0 255.255.255.0 10.0.12.2
ip route 192.168.10.0 255.255.255.0 10.0.13.2
end
```

**R2 (AVG + AVF1)**
```
conf t
hostname R2

interface f0/0
 ip address 192.168.10.100 255.255.255.0
 no shutdown

interface s0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.12.1
end
```

**R3 (AVF2)**
```
conf t
hostname R3

interface f0/0
 ip address 192.168.10.200 255.255.255.0
 no shutdown

interface s0/1
 ip address 10.0.13.2 255.255.255.252
 no shutdown

ip route 0.0.0.0 0.0.0.0 10.0.13.1
end
```

---

### 2단계. GLBP 설정

**R2 (Priority 110 → AVG로 선출)**
```
conf t
interface f0/0
 glbp 10 ip 192.168.10.1
 glbp 10 priority 110
 glbp 10 preempt
 glbp 10 load-balancing round-robin
 glbp 10 authentication md5 key-string GLBPKEY
 glbp 10 timers 3 10
end
```

**R3 (Priority 100 → AVF)**
```
conf t
interface f0/0
 glbp 10 ip 192.168.10.1
 glbp 10 priority 100
 glbp 10 preempt
 glbp 10 load-balancing round-robin
 glbp 10 authentication md5 key-string GLBPKEY
 glbp 10 timers 3 10
end
```

### 설정 항목 설명

| 명령어 | 설명 |
|--------|------|
| `glbp 10 ip 192.168.10.1` | GLBP 그룹 10, 가상 IP 설정 |
| `glbp 10 priority 110` | 우선순위 설정 (높을수록 AVG 선출 우선) |
| `glbp 10 preempt` | Priority가 높은 라우터가 복구 시 AVG를 다시 가져옴 |
| `glbp 10 load-balancing round-robin` | 로드밸런싱 방식 (기본값) |
| `glbp 10 authentication md5 key-string` | MD5 인증 (양쪽 키 동일 필수) |
| `glbp 10 timers 3 10` | Hello 3초, Hold 10초 |

---

### 3단계. PC 설정

```
PC1> ip 192.168.10.10 255.255.255.0 192.168.10.1
PC2> ip 192.168.10.20 255.255.255.0 192.168.10.1
PC3> ip 192.168.10.30 255.255.255.0 192.168.10.1
```

---

### 4단계. 상태 확인

```
R2# show glbp
```

R2 출력 예시 (AVG + AVF1):
```
FastEthernet0/0 - Group 10
  State is Active
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

  There are 2 forwarders (1 active)

  Forwarder 1
    State is Active
    MAC address is 0007.b400.0a01    ← R2의 가상 MAC

  Forwarder 2
    State is Listen
    MAC address is 0007.b400.0a02    ← R3의 가상 MAC
```

```
R3# show glbp
```

R3 출력 예시 (AVF2):
```
FastEthernet0/0 - Group 10
  State is Standby
  Virtual IP address is 192.168.10.1
  Priority 100 (default)
  Active is 192.168.10.100, priority 110

  Forwarder 2
    State is Active
    MAC address is 0007.b400.0a02
```

간략 확인:
```
show glbp brief
```

---

### 5단계. 로드밸런싱 동작 확인

각 PC에서 게이트웨이로 ping을 보낸 후 ARP 테이블 확인:

```
PC1> ping 192.168.10.1
PC1> show arp
```

PC별로 게이트웨이 MAC이 다르게 표시됨:
```
PC1: 192.168.10.1 → 0007.b400.0a01 (R2로 통신)
PC2: 192.168.10.1 → 0007.b400.0a02 (R3로 통신)
PC3: 192.168.10.1 → 0007.b400.0a01 (R2로 통신, Round-Robin)
```

같은 가상 IP(192.168.10.1)인데 PC마다 다른 MAC을 사용하므로 트래픽이 분산된다.

---

## 실습 2: Weighting + Track을 이용한 AVF 제어

### 시나리오

R2의 업링크(s0/0)가 다운되면 R2의 Weighting을 낮춰서 AVF 역할을 포기하게 한다. 모든 트래픽이 R3으로 전환.

### Track 동작 흐름

```
정상 상태:
  [R2] Weighting: 100 (> Lower 1) → AVF1 역할 수행
  [R3] Weighting: 100 (> Lower 1) → AVF2 역할 수행
  → 트래픽 분산

업링크(s0/0) 장애:
  [R2] Weighting: 100 - 80 = 20
       Track으로 Decrement 80 적용
       Lower Threshold를 30으로 설정했다면
       20 < 30 → AVF 역할 포기
  [R3] Weighting: 100 → AVF 역할 유지
  → 모든 트래픽이 R3으로 전환

업링크 복구:
  [R2] Weighting: 100 (복구)
       Upper Threshold(100) 이상 → AVF 역할 재개
  → 다시 트래픽 분산
```

### R2 설정

```
conf t
track 10 interface s0/0 line-protocol

interface f0/0
 glbp 10 weighting 100 lower 30 upper 60
 glbp 10 weighting track 10 decrement 80
end
```

- `weighting 100`: 초기 가중치 100
- `lower 30`: 30 이하로 떨어지면 AVF 역할 포기
- `upper 60`: 60 이상으로 올라가면 AVF 역할 재개
- `track 10 decrement 80`: s0/0 다운 시 가중치 80 감소

### 장애 테스트

**업링크 shutdown**
```
R2(config)# interface s0/0
R2(config-if)# shutdown
```

확인:
```
R2# show glbp
```

Forwarder 1의 State가 Active → Listen으로 변경되고, R3이 모든 트래픽을 처리.

**업링크 복구**
```
R2(config)# interface s0/0
R2(config-if)# no shutdown
```

Weighting이 100으로 복구되어(> upper 60) AVF 역할 재개.

---

## 실습 3: Weighted 로드밸런싱

Round-Robin 대신 가중치 비율로 트래픽을 분배하는 방식.

### 시나리오

R2에 트래픽 70%, R3에 30%를 분배.

### R2 설정

```
conf t
interface f0/0
 glbp 10 load-balancing weighted
 glbp 10 weighting 70
end
```

### R3 설정

```
conf t
interface f0/0
 glbp 10 load-balancing weighted
 glbp 10 weighting 30
end
```

AVG가 ARP 응답 시 R2의 MAC을 약 70% 비율로, R3의 MAC을 약 30% 비율로 응답한다.

### Host-Dependent 로드밸런싱

세션 유지가 필요한 환경에서는 출발지 MAC 기준으로 항상 같은 AVF에 매핑:

```
interface f0/0
 glbp 10 load-balancing host-dependent
```

동일한 PC는 항상 같은 AVF를 사용하므로 세션이 끊기지 않음.

---

## GLBP 장애 시나리오 정리

```
시나리오 1: AVF 장애 (R3 다운)

  R3 다운 전                    R3 다운 후
  PC1 → R2 (AVF1)              PC1 → R2 (AVF1)
  PC2 → R3 (AVF2)              PC2 → R2 (AVF1)  ← R2가 R3의 MAC도 대리 응답
  트래픽 분산                    R2가 모든 트래픽 처리


시나리오 2: AVG 장애 (R2 다운)

  R2 다운 전                    R2 다운 후
  R2 = AVG + AVF1               R3 = AVG (승격) + AVF2
  R3 = AVF2                     R3이 모든 역할 수행
  트래픽 분산                    R3이 단독 처리


시나리오 3: AVF Track 발동 (R2 업링크 다운)

  업링크 다운 전                 업링크 다운 후
  R2 = AVG + AVF1               R2 = AVG (유지, Weighting 감소)
  R3 = AVF2                     R2 AVF 역할 포기
  트래픽 분산                    R3이 모든 트래픽 처리
                                 (R2는 AVG만 유지)
```

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show glbp` | GLBP 전체 상태 (AVG/AVF, Forwarder, MAC, Weighting) |
| `show glbp brief` | GLBP 간략 상태 |
| `show glbp detail` | GLBP 상세 정보 (타이머, Track 포함) |
| `show track` | Track 객체 상태 |

---

## 설정 명령어 정리

```
-- 기본 설정
glbp [그룹] ip [가상IP]
glbp [그룹] priority [값]
glbp [그룹] preempt
glbp [그룹] timers [hello] [hold]
glbp [그룹] authentication md5 key-string [키]

-- 로드밸런싱
glbp [그룹] load-balancing [round-robin|weighted|host-dependent]

-- Weighting
glbp [그룹] weighting [값] lower [하한] upper [상한]
glbp [그룹] weighting track [객체번호] decrement [감소값]

-- Track 객체
track [번호] interface [인터페이스] line-protocol
```

---

## FHRP 3종 최종 비교

| 구분 | HSRP | VRRP | GLBP |
|------|------|------|------|
| 표준 | Cisco 전용 | IEEE 표준 | Cisco 전용 |
| Active 명칭 | Active | Master | AVG |
| Standby 명칭 | Standby | Backup | AVF |
| 로드밸런싱 | X | X | O (기본 지원) |
| 가상 MAC 수 | 1개 | 1개 | 최대 4개 |
| Hello 주기 | 3초 | 1초 | 3초 |
| Preempt 기본 | 비활성 | 활성 | 비활성 |
| 프로토콜 | UDP 1985 | IP 112 | UDP 3222 |
| Multicast | 224.0.0.2 / .102 | 224.0.0.18 | 224.0.0.102 |
| 적용 환경 | Cisco 단일벤더 | 멀티벤더 | Cisco + 로드밸런싱 필요 |

---

## 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| 로드밸런싱 안 됨 | load-balancing 미설정 또는 라우터 1대만 동작 | 양쪽 GLBP 설정 확인 |
| AVG 복구 후 Standby로 남음 | preempt 미설정 | `glbp [그룹] preempt` 추가 |
| 업링크 다운인데 AVF 유지 | Track/Weighting 미설정 | Track + Weighting lower 설정 |
| 인증 실패로 그룹 형성 안 됨 | key-string 불일치 | 양쪽 동일한 키 확인 |
| PC가 항상 같은 라우터 사용 | host-dependent 모드 | round-robin으로 변경 |

---

## 정리

- GLBP는 게이트웨이 이중화와 로드밸런싱을 동시에 제공하는 Cisco 전용 프로토콜
- AVG가 ARP 응답 시 각 AVF의 가상 MAC을 돌아가며 응답하여 트래픽을 자동 분산
- 그룹당 최대 4대의 AVF가 동시에 트래픽을 처리할 수 있어 자원 활용도가 높음
- Weighting과 Track을 연동하면 업링크 장애 시 AVF 역할을 자동으로 포기/재개하여 블랙홀을 방지
- Round-Robin(기본), Weighted(비율 조절), Host-Dependent(세션 유지) 3가지 로드밸런싱 방식 지원
- 실무에서 Cisco 환경 + 로드밸런싱이 필요한 경우 HSRP보다 GLBP가 효율적
