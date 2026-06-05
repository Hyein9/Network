# VLAN 실습

## 개념 정리

### VLAN이란
- 한 대의 스위치를 논리적으로 여러 대처럼 나누는 기술
- 물리적으로는 스위치 1대지만 논리적으로는 여러 개의 독립된 네트워크로 분리됨
- VLAN이 다르면 같은 스위치에 연결되어 있어도 서로 통신할 수 없음

### VLAN의 핵심 특성

**1. 브로드캐스트 도메인 분리**
- VLAN 10에서 ARP 브로드캐스트를 보내면 VLAN 20은 수신하지 못함
- VLAN 하나가 하나의 브로드캐스트 도메인

**2. VLAN 간 통신은 기본적으로 불가**
- VLAN 10과 VLAN 20은 직접 통신할 수 없음
- 통신하려면 라우터 또는 L3 스위치가 필요 (Inter-VLAN Routing)

**3. Access 포트 vs Trunk 포트**

| 구분 | 용도 | VLAN 수 | 연결 대상 |
|------|------|---------|----------|
| Access 포트 | 단말 장비 연결 | 1개만 소속 | PC, 서버 등 |
| Trunk 포트 | 스위치 간 연결 | 여러 VLAN 태그 전달 | 스위치, 라우터 |

**4. VLAN ID와 이름**
- VLAN ID: 1~4094 범위의 고유 번호
- 이름: 관리 편의를 위해 붙이는 식별 문자열

---

### VLAN 종류

**(1) Static VLAN (포트 기반)**
- 스위치의 특정 포트에 VLAN을 할당하는 방식
- 어떤 기기를 연결하든 해당 포트에 지정된 VLAN에 소속됨
- 단점: 모든 설정을 관리자가 직접 수행해야 함

**(2) Dynamic VLAN (MAC 주소 기반)**
- 연결된 기기의 MAC 주소를 인식해서 스위치가 자동으로 VLAN을 할당
- 노트북이나 모바일 기기처럼 자리를 이동하는 사용자가 많아지면서 개발됨
- 기기에 따라 VLAN이 달라질 수 있어서 Dynamic이라 부름

**(3) End-to-End VLAN**
- 물리적 위치와 관계없이 업무나 데이터 종류에 따라 VLAN을 할당
- 사용자가 위치를 이동해도 동일한 VLAN에 소속됨
- 모든 트래픽이 Core를 거치게 되어 대규모 엔터프라이즈 망에서는 권장하지 않음

**(4) Local VLAN**
- 층이나 물리적 위치에 따라 네트워크를 별도 VLAN으로 분리
- 사용자가 위치를 이동하면 다른 VLAN에 소속됨
- 장애 원인 파악이 쉽고, 네트워크 확장이나 이중화 구성이 용이해서 엔터프라이즈 망에서 권장
- 단점: 설정이 복잡하고 L3 스위치나 라우터 같은 추가 장비가 필요

---

### VLAN 동작 방식

| 방식 | 설명 |
|------|------|
| 포트 기반 (Port-based) | 스위치의 특정 포트를 특정 VLAN에 할당 |
| MAC 주소 기반 (MAC-based) | 기기의 MAC 주소를 인식하여 포트 이동과 관계없이 VLAN 할당 |
| 태그 기반 (Tagged) | 이더넷 프레임에 VLAN ID를 추가(IEEE 802.1Q)하여 서로 다른 스위치 간 VLAN 전달 |

### Trunk 포트와 802.1Q 태깅

```
태그 없는 프레임 (Access 포트)         태그 있는 프레임 (Trunk 포트)
+------+------+------+               +------+--------+------+------+
| Dst  | Src  | Data |               | Dst  | 802.1Q | Src  | Data |
| MAC  | MAC  |      |               | MAC  | Tag    | MAC  |      |
+------+------+------+               +------+--------+------+------+
                                              |
                                         VLAN ID 포함
```

- Trunk 포트는 프레임에 VLAN ID 태그를 붙여서 전송
- 수신 측 스위치가 태그를 확인하고 해당 VLAN으로 전달
- Access 포트로 나갈 때는 태그를 제거하고 전달

### Native VLAN
- Trunk 포트에서 태그를 붙이지 않고 전송하는 VLAN
- 기본값은 VLAN 1
- 양쪽 스위치의 Native VLAN이 반드시 동일해야 함 (불일치 시 STP가 포트를 차단)

---

## 실습 1: VLAN 기본 구성

### 토폴로지

```
  [PC1]      [PC2]      [PC3]      [PC4]
 VLAN 10    VLAN 20    VLAN 10    VLAN 20
    |          |          |          |
  f0/1       f0/2       f0/3       f0/4
  +----+------+----------+----------+----+
  |                SW1                    |
  +---------------------------------------+
```

### IP 주소 구성

| PC | IP | Gateway | VLAN | 연결 포트 |
|----|----|---------|------|----------|
| PC1 | 192.168.10.10 | 192.168.10.1 | VLAN 10 | f0/1 |
| PC2 | 192.168.20.10 | 192.168.20.1 | VLAN 20 | f0/2 |
| PC3 | 192.168.10.11 | 192.168.10.1 | VLAN 10 | f0/3 |
| PC4 | 192.168.20.11 | 192.168.20.1 | VLAN 20 | f0/4 |

### 통신 가능 여부

```
PC1 (VLAN 10) <---> PC3 (VLAN 10)   통신 가능 (같은 VLAN)
PC2 (VLAN 20) <---> PC4 (VLAN 20)   통신 가능 (같은 VLAN)
PC1 (VLAN 10) <-X-> PC2 (VLAN 20)   통신 불가 (라우터 없음)
```

---

### 1단계. VLAN 초기화

기존 VLAN 정보가 남아 있을 수 있으므로 초기화 후 시작.

```
delete vlan.dat
reload
```

재부팅 후 확인:
```
show vlan
```
VLAN 1만 존재하고 모든 포트가 VLAN 1에 들어가 있으면 정상. 1002~1005는 Cisco 기본 VLAN으로 삭제 불가.

### 2단계. 스위치 이름 변경 및 VLAN 생성

```
conf t
hostname SW1

vlan 10
 name VLAN10
exit

vlan 20
 name VLAN20
exit
```

### 3단계. 포트에 VLAN 할당

```
int range f0/1, f0/3
 switchport mode access
 switchport access vlan 10
exit

int range f0/2, f0/4
 switchport mode access
 switchport access vlan 20
exit
end
```

### 4단계. 설정 확인

```
show vlan
```

정상 출력:
```
VLAN  Name        Status    Ports
----  ----------  ------    -------------------------
1     default     active    Fa0/5 ~ Fa0/24 (나머지 전부)
10    VLAN10      active    Fa0/1, Fa0/3
20    VLAN20      active    Fa0/2, Fa0/4
```

### 5단계. 저장

```
copy run start
```

저장하지 않으면 전원이 꺼질 때 설정이 전부 사라진다.

---

## 실습 2: Trunk 구성 및 Inter-VLAN Routing (Router-on-a-Stick)

### 토폴로지

```
  [PC1]      [PC2]                               [R1]
 VLAN 10    VLAN 20                            (Router-on-a-Stick)
    |          |                                   |
  f0/1       f0/2                                f0/0
  +----+------+-----+        f0/10        +-------+-------+
  |        SW1       +-------(trunk)------+      SW2      |
  +------------------+                    +---------------+
```

### 상세 연결도

```
  [PC1]        [PC2]
 192.168.10.10  192.168.20.10
  VLAN 10       VLAN 20
     |             |
   f0/1          f0/2        f0/10              f0/20          f0/0
  +-+--------------+-----------+--+    +---------+--------------+--+
  |           SW1              |  |    |         SW2               |
  +----------------------------+--+    +---------+-----------------+
                               |                 |
                            (trunk)           (trunk)
                            VLAN 10,20        VLAN 10,20
                               |                 |
                               +----- f0/10 -----+
                                                 |
                                               f0/0 (no shutdown)
                                                 |
                                          +------+------+
                                          |     R1      |
                                          |  f0/0.1     |  서브인터페이스
                                          |  VLAN 10    |  192.168.10.1
                                          |  f0/0.2     |
                                          |  VLAN 20    |  192.168.20.1
                                          +-------------+
```

### VLAN 간 라우팅 흐름

```
PC1 (VLAN 10, 192.168.10.10)이 PC2 (VLAN 20, 192.168.20.10)에게 ping:

  [PC1] ---> [SW1 f0/1]
                |
                | VLAN 10 프레임
                v
          [SW1 f0/10] ---(trunk, 802.1Q tag: VLAN 10)---> [SW2]
                                                             |
                                                          (trunk)
                                                             v
                                                      [R1 f0/0.1]
                                                       VLAN 10 수신
                                                             |
                                                       라우팅 처리
                                                       목적지: 192.168.20.0/24
                                                             |
                                                             v
                                                      [R1 f0/0.2]
                                                       VLAN 20으로 전송
                                                             |
                                                          (trunk)
                                                             v
                                                          [SW2]
                                                             |
                                                          [SW1]
                                                             |
                                                        [PC2] 도착
```

---

### 1단계. Trunk 포트 설정 (SW1, SW2)

**SW1**
```
conf t
int f0/10
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport trunk native vlan 1
end
```

**SW2**
```
conf t
int f0/10
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport trunk native vlan 1

int f0/20
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport trunk native vlan 1
end
```

Trunk 확인:
```
show interfaces trunk
```

### Native VLAN 불일치 에러

양쪽 스위치의 Native VLAN이 다르면 아래 에러가 발생한다:
```
%CDP-4-NATIVE_VLAN_MISMATCH
%SPANTREE-2-BLOCK_PVID_LOCAL
```
STP가 해당 포트를 차단하므로 통신이 안 됨. 양쪽 Native VLAN을 동일하게 맞춰야 한다.

### Trunk에서 VLAN 제어

```
switchport trunk allowed vlan remove 20    -- VLAN 20 제거
switchport trunk allowed vlan add 20       -- VLAN 20 다시 추가
```

---

### 2단계. 라우터 서브인터페이스 설정 (Router-on-a-Stick)

L2 스위치만으로는 VLAN 간 통신이 불가능하므로, 라우터에 서브인터페이스를 만들어 각 VLAN의 게이트웨이 역할을 하게 한다.

주의: encapsulation dot1Q를 먼저 설정한 뒤에 IP를 할당해야 한다. 순서가 바뀌면 에러가 발생함.

**R1**
```
conf t
int f0/0
 no shutdown

int f0/0.1
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

int f0/0.2
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
end
```

라우팅 테이블 확인:
```
show ip route
```

정상 출력:
```
C    192.168.10.0/24 is directly connected, FastEthernet0/0.1
C    192.168.20.0/24 is directly connected, FastEthernet0/0.2
```

### 3단계. 통신 테스트

```
PC1> ping 192.168.20.10
```

VLAN 10에서 VLAN 20으로 ping이 성공하면 Inter-VLAN Routing 구성 완료. 첫 번째 ping에서 타임아웃이 나는 건 ARP 학습 과정이라 정상.

---

## 실습 3: 멀티레이어 스위치 (L3 Switch) Inter-VLAN Routing

### 토폴로지

```
  [PC1]                [PC2]
 192.168.10.10        192.168.20.10
  VLAN 10              VLAN 20
     |                    |
   f0/1                 f0/2
  +--+--------------------+--+
  |      MLS (3560)          |
  |   멀티레이어 스위치       |
  |                          |
  |  VLAN 10 SVI             |
  |  192.168.10.1            |
  |                          |
  |  VLAN 20 SVI             |
  |  192.168.20.1            |
  +--------------------------+
```

### Router-on-a-Stick vs L3 Switch 비교

| 구분 | Router-on-a-Stick | 멀티레이어 스위치 |
|------|-------------------|----------------|
| 성능 | 느림 (단일 링크 병목) | 빠름 (하드웨어 라우팅) |
| 구성 | 복잡 (서브인터페이스) | 깔끔 (SVI만 설정) |
| 장비 | 별도 라우터 필요 | 스위치 하나로 해결 |
| 적용 | 소규모 네트워크 | 중대형 네트워크 |

주의: 3560, 3650, 3750 같은 L3 스위치를 사용해야 한다. 2950, 2960은 L2 전용이라 IP 라우팅이 불가능.

### L2 스위치에서 물리 포트에 IP를 설정하면

```
int f0/20
 ip address 192.168.10.1 255.255.255.0
```
위 명령은 L2 스위치에서 에러가 발생한다. L2 스위치의 관리용 IP는 VLAN 인터페이스(SVI)에만 설정 가능.

---

### 1단계. VLAN 생성

```
conf t
vlan 10
 name VLAN10

vlan 20
 name VLAN20
```

### 2단계. SVI에 IP 할당 (게이트웨이 역할)

```
int vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

int vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```

### 3단계. IP 라우팅 활성화

```
ip routing
```

이 명령을 빠뜨리면 L3 스위치가 라우팅을 수행하지 않아서 VLAN 간 통신이 안 된다. 가장 많이 실수하는 부분.

### 4단계. PC 연결 포트 설정

```
int f0/1
 switchport mode access
 switchport access vlan 10

int f0/2
 switchport mode access
 switchport access vlan 20
end
```

### 5단계. 확인 및 테스트

라우팅 테이블 확인:
```
show ip route
```

정상 출력:
```
C    192.168.10.0/24 is directly connected, Vlan10
C    192.168.20.0/24 is directly connected, Vlan20
```

VLAN 확인:
```
show vlan
```

통신 테스트:
```
PC1> ping 192.168.20.10
```
라우터 없이 바로 통신이 됨.

### 멀티레이어 스위치에서 자주 발생하는 문제

| 증상 | 원인 |
|------|------|
| VLAN 간 통신 안 됨 | `ip routing` 안 켬 |
| SVI가 down 상태 | 해당 VLAN에 포트가 없거나 no shutdown 안 함 |
| ping 안 됨 | PC 게이트웨이 설정 안 함 |
| IP 설정 안 됨 | L2 스위치로 실습함 (2950/2960) |

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show vlan` | VLAN 목록 및 포트 할당 확인 |
| `show interfaces trunk` | Trunk 포트 상태, 허용 VLAN 확인 |
| `show interfaces status` | 포트 상태 (speed, duplex, VLAN) |
| `show run interface f0/1` | 특정 포트 설정 확인 |
| `show ip route` | 라우팅 테이블 (L3 스위치용) |
| `copy run start` | 설정 저장 |

---

## 실습에서 자주 발생하는 에러 정리

| 에러 | 원인 | 해결 |
|------|------|------|
| Native VLAN mismatch | 트렁크 양쪽 Native VLAN 다름 | 양쪽 동일하게 설정 |
| Spanning-tree block | VLAN ID 불일치로 STP 차단 | Native VLAN 맞춤 |
| L2 스위치 포트에 IP 넣음 | L2 스위치는 물리 포트에 IP 불가 | SVI 사용 |
| encapsulation 에러 | dot1Q 설정 전에 IP 할당 시도 | encapsulation 먼저 설정 |
| VLAN 간 통신 안 됨 | 라우터/L3 스위치 없음 | Inter-VLAN Routing 구성 필요 |

---

## 정리

- VLAN은 하나의 스위치를 논리적으로 분리하여 브로드캐스트 도메인을 나누는 기술
- Access 포트는 VLAN 1개, Trunk 포트는 여러 VLAN의 트래픽을 태그로 구분하여 전달
- 서로 다른 VLAN 간 통신은 라우터(Router-on-a-Stick) 또는 L3 스위치(멀티레이어)가 필요
- L3 스위치는 `ip routing` 활성화와 SVI 설정만으로 VLAN 간 라우팅이 가능하여 대규모 환경에 적합
