# SPAN (Switch Port Analyzer) / Port Mirroring 실습

## 개념 정리

### SPAN이란
- 스위치의 특정 포트를 지나가는 트래픽을 다른 포트로 복제하여 패킷 분석 장비(Sniffer)가 모니터링할 수 있게 하는 기능
- Port Mirroring이라고도 부르며, 네트워크에 영향을 주지 않고 트래픽을 분석할 수 있음
- Sniffer 장비는 실제 통신에 참여하지 않아도 모든 패킷을 확인 가능

### SPAN이 필요한 이유
- 스위치는 MAC 주소 테이블을 기반으로 목적지 포트에만 프레임을 전달하므로, 다른 포트에서는 해당 트래픽을 볼 수 없음
- 허브와 달리 스위치 환경에서는 패킷 캡처를 위해 SPAN 설정이 필수

### SPAN 구성 요소

| 구성 요소 | 역할 |
|----------|------|
| Source Port | 모니터링 대상 포트. 이 포트의 트래픽이 복제됨 |
| Destination Port | Sniffer가 연결된 포트. 복제된 트래픽을 수신 |
| Monitor Session | Source와 Destination을 묶는 세션 단위 설정 |

### SPAN 트래픽 방향

| 방향 | 설명 |
|------|------|
| Both (기본) | 수신(rx) + 송신(tx) 양방향 트래픽 복제 |
| rx (Receive) | 해당 포트로 들어오는 트래픽만 복제 |
| tx (Transmit) | 해당 포트에서 나가는 트래픽만 복제 |

### SPAN 종류

| 종류 | 설명 |
|------|------|
| Local SPAN | 같은 스위치 내에서 Source → Destination 미러링 |
| RSPAN (Remote SPAN) | 서로 다른 스위치 간에 VLAN을 통해 미러링 |
| ERSPAN (Encapsulated RSPAN) | GRE 터널을 통해 원격 미러링. L3 환경에서도 가능 |

### SPAN 주의사항
- Destination Port는 일반 통신에 사용할 수 없음 (Sniffer 전용)
- Source Port와 Destination Port는 같은 VLAN일 필요 없음
- 하나의 Destination Port에 여러 Source Port를 지정할 수 있음
- 과도한 Source 포트를 지정하면 Destination Port에 트래픽 과부하 발생 가능

---

## 실습 토폴로지

```
                      [R1]
                   192.168.10.1
                       |
                     f1/1
                       |
  +--------------------+--------------------+
  |                  [ESW1]                  |
  |                                          |
  |   f1/2          f1/3          f1/10      |
  +----+------+------+------+------+---------+
       |             |             |
     [PC1]         [PC2]       [Sniffer]
   .10.10          .10.20       .10.30
                              (Wireshark)
```

### 상세 트래픽 흐름

```
  [PC1] <-------> [PC2]    정상 통신
  f1/2             f1/3
     \             /
      \           /
       \         /
        [ESW1]
           |
           | SPAN: f1/2, f1/3의 트래픽을
           |       f1/10으로 복제
           |
         f1/10
           |
       [Sniffer]
     (패킷 캡처)

  Sniffer는 통신에 참여하지 않지만
  PC1 ↔ PC2의 모든 패킷을 수신
```

### IP 주소 구성

| 장비 | 인터페이스 | IP 주소 | 역할 |
|------|-----------|---------|------|
| R1 | f0/0 | 192.168.10.1/24 | 게이트웨이 |
| PC1 | - | 192.168.10.10/24 | 통신 대상 1 |
| PC2 | - | 192.168.10.20/24 | 통신 대상 2 |
| Sniffer | - | 192.168.10.30/24 | 패킷 분석 장비 |

### 스위치 포트 구성

| 스위치 포트 | 연결 장비 | SPAN 역할 |
|------------|----------|-----------|
| f1/1 | R1 | - |
| f1/2 | PC1 | Source Port |
| f1/3 | PC2 | Source Port |
| f1/10 | Sniffer | Destination Port |

---

## 1단계. 라우터 설정

**R1**
```
enable
conf t

interface f0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
end
write
```

---

## 2단계. PC IP 설정

```
PC1: ip 192.168.10.10/24 192.168.10.1
PC2: ip 192.168.10.20/24 192.168.10.1
Sniffer: ip 192.168.10.30/24 192.168.10.1
```

Sniffer의 IP는 필수가 아니지만, 관리 편의를 위해 설정.

---

## 3단계. 스위치 기본 설정

**ESW1**
```
enable
conf t
hostname ESW1

interface range f1/2 - 3
 switchport mode access
 no shutdown

interface f1/10
 switchport mode access
 no shutdown
end
write
```

---

## 4단계. SPAN 설정

PC1(f1/2)과 PC2(f1/3) 포트의 트래픽을 Sniffer(f1/10)로 복제한다.

```
enable
conf t

monitor session 1 source interface f1/2
monitor session 1 source interface f1/3
monitor session 1 destination interface f1/10
end
write
```

### 특정 방향만 미러링할 경우

```
-- 수신 트래픽만 복제
monitor session 1 source interface f1/2 rx

-- 송신 트래픽만 복제
monitor session 1 source interface f1/2 tx

-- 양방향 (기본값)
monitor session 1 source interface f1/2 both
```

---

## 5단계. SPAN 설정 확인

```
show monitor session 1
```

정상 출력:
```
Session 1
---------
Type                   : Local Session
Source Ports           :
    Both               : Fa1/2
    Both               : Fa1/3
Destination Ports      : Fa1/10
```

- Type: Local Session (같은 스위치 내 미러링)
- Source Ports: 미러링 대상 (Both = 양방향)
- Destination Ports: Sniffer 연결 포트

---

## 6단계. 통신 테스트 및 패킷 캡처

### PC1에서 PC2로 ping

```
PC1> ping 192.168.10.20
```

### Sniffer에서 패킷 캡처

Wireshark 실행 후 인터페이스 캡처 시작, 또는:
```
tcpdump -i eth0
```

캡처되는 패킷 예시:
```
ICMP Echo Request:  192.168.10.10 → 192.168.10.20
ICMP Echo Reply:    192.168.10.20 → 192.168.10.10
ARP Request:        Who has 192.168.10.20? Tell 192.168.10.10
ARP Reply:          192.168.10.20 is at xx:xx:xx:xx:xx:xx
```

Sniffer는 통신에 직접 참여하지 않지만 PC1 ↔ PC2 간의 모든 패킷(ICMP, ARP 등)을 확인할 수 있다.

---

## SPAN 삭제

```
conf t
no monitor session 1
end
```

특정 Source만 제거:
```
no monitor session 1 source interface f1/2
```

---

## SPAN 활용 사례

```
  사례 1: 보안 모니터링 (IDS/IPS 연동)

  [서버] ---- [스위치] ---- [인터넷]
                  |
                SPAN
                  |
               [IDS/IPS]
               침입 탐지


  사례 2: 네트워크 장애 분석

  [PC] ---- [스위치] ---- [서버]
                |
              SPAN
                |
           [Wireshark]
           패킷 분석으로
           장애 원인 파악


  사례 3: 트래픽 통계 수집

  [전체 네트워크]
        |
      [스위치]
        |
      SPAN
        |
   [NetFlow/sFlow]
   트래픽 패턴 분석
```

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show monitor session 1` | SPAN 세션 설정 확인 (Source/Destination) |
| `show monitor session all` | 모든 SPAN 세션 확인 |
| `show interfaces status` | 포트 상태 확인 |
| `show running-config \| section monitor` | SPAN 관련 설정만 필터링 |

---

## 설정 명령어 정리

```
-- SPAN 세션 생성 (Source 지정)
monitor session [번호] source interface [인터페이스] [rx|tx|both]

-- SPAN 세션 생성 (Destination 지정)
monitor session [번호] destination interface [인터페이스]

-- VLAN 단위로 Source 지정
monitor session [번호] source vlan [VLAN번호] [rx|tx|both]

-- SPAN 세션 삭제
no monitor session [번호]

-- 특정 Source만 제거
no monitor session [번호] source interface [인터페이스]
```

---

## 정리

- SPAN은 스위치 포트의 트래픽을 다른 포트로 복제하여 패킷 분석 장비가 모니터링할 수 있게 하는 기능
- Source Port(모니터링 대상)와 Destination Port(Sniffer 연결)를 지정하여 세션을 구성
- Destination Port는 Sniffer 전용이므로 일반 통신에 사용할 수 없음
- 네트워크 트래픽 분석, 보안 모니터링(IDS/IPS), 장애 분석 등에 활용
- Wireshark나 tcpdump 같은 패킷 분석 도구와 함께 사용하여 네트워크 내부 트래픽을 분석할 수 있음
