# STP (Spanning Tree Protocol) 및 BPDU 실습

## 개념 정리

### STP란
- 스위치나 브리지 환경에서 루핑(Looping)을 방지하는 프로토콜
- 스위치를 이중화 구성하면 물리적으로 루프가 발생할 수 있는데, STP가 특정 포트를 논리적으로 차단하여 루프를 막음
- IEEE 802.1D 표준으로 정의

### 루프가 발생하면
```
  [SW1] -------> [SW2]
    |               |
    +---- 루프 -----+
         발생!
```
- 브로드캐스트 프레임이 무한 순환하여 네트워크 마비
- MAC 주소 테이블이 불안정해짐 (MAC flapping)
- CPU 과부하로 스위치 다운 가능

### BPDU (Bridge Protocol Data Unit)
- 스위치끼리 STP 정보를 교환하기 위해 주고받는 프레임
- Root Bridge ID, Path Cost, Bridge ID, Port ID 등의 정보를 포함
- Root Bridge는 매 2초(Hello Time)마다 BPDU를 전송하고, 나머지 스위치는 이를 수신

### Bridge ID 구조

```
+------------------+---------------------+
|    Priority      |    MAC Address      |
|   (16bit)        |    (48bit)          |
+------------------+---------------------+
     |
     +-- 기본값: 32768
         4096의 배수로 변경 가능
         sys-id-ext (VLAN 번호) 포함
```

Bridge ID = Priority + MAC Address. Priority가 낮을수록 우선순위가 높다.

---

## STP 포트 상태 (Port States)

STP는 포트 상태를 5단계로 관리하며, 루프 방지를 위해 단계적으로 전환한다.

### 상태 전이 흐름

```
Disabled ---> Blocking ---> Listening ---> Learning ---> Forwarding
               (20초)        (15초)        (15초)
                               |              |
                               v              v
                           상황 변경 시 Blocking으로 복귀 가능
```

### 각 상태 상세

| 상태 | 데이터 전송 | MAC 학습 | BPDU 송수신 | 시간 | 설명 |
|------|:---------:|:-------:|:----------:|------|------|
| Disabled | X | X | X | - | 포트 고장 또는 관리자가 shutdown 시킨 상태 |
| Blocking | X | X | 수신만 | 20초 | 스위치 최초 부팅 시 상태. STP 선정 과정이 이 단계에서 진행됨 |
| Listening | X | X | O | 15초 | Root Port나 Designated Port로 선정된 포트가 진입. 상황 변경 시 Blocking으로 복귀 가능 |
| Learning | X | O | O | 15초 | MAC 주소 테이블을 만들기 시작하는 단계. 아직 데이터 전송은 불가 |
| Forwarding | O | O | O | - | 실제 데이터 프레임을 주고받을 수 있는 최종 상태 |

Blocking(20초) + Listening(15초) + Learning(15초) = 총 약 50초가 소요된다. 이 시간 동안 해당 포트로 데이터 전송이 불가.

---

## STP 포트 역할 (Port Roles)

### Root Port (RP)
- Root Bridge로 가는 최단 경로 포트
- 각 일반 스위치마다 1개만 존재
- Forwarding 상태

선정 기준 (우선순위 순):
1. Root까지 Path Cost가 가장 낮음
2. 상대 Bridge ID가 가장 낮음
3. 상대 Port ID가 가장 낮음

### Designated Port (DP)
- 해당 네트워크 세그먼트의 대표 포트
- 각 링크(세그먼트)마다 1개 존재
- 해당 구간에서 Root까지 비용이 가장 낮은 스위치의 포트
- Forwarding 상태

### Blocked Port (Non-Designated Port)
- Root Port도 Designated Port도 아닌 포트
- 루프 방지를 위해 차단된 상태
- BPDU는 수신하지만 데이터는 전달하지 않음

### 포트 역할 예시

```
          [ Root Bridge ]
          (모든 포트 = DP)
            |         |
          (DP)      (DP)
            |         |
         [SW1]------[SW2]
          (RP)      (RP)
            \        /
           (DP)  (Blocked)
              \  /
             [SW3]
             (RP)
```

- Root Bridge: 모든 포트가 Designated Port
- SW1, SW2: Root 방향 포트가 Root Port
- SW3: Root로 가는 최적 경로가 Root Port, 반대편은 Blocked

---

## STP 선정 과정

### 1단계. Root Bridge 선출
- 모든 스위치가 BPDU를 교환하여 Bridge ID가 가장 낮은 스위치를 Root Bridge로 선출
- Bridge ID = Priority(기본 32768) + MAC Address
- Priority가 같으면 MAC Address가 낮은 쪽이 Root Bridge

### 2단계. Root Port 선정
- Root Bridge가 아닌 모든 스위치에서 Root Bridge까지 Cost가 가장 낮은 포트를 Root Port로 선정

### 3단계. Designated Port 선정
- 각 세그먼트에서 Root Bridge까지 Cost가 가장 낮은 스위치의 포트가 Designated Port
- Root Bridge의 모든 포트는 Designated Port

### Path Cost 기준

| 대역폭 | Cost |
|--------|------|
| 10 Gbps | 2 |
| 1 Gbps | 4 |
| 100 Mbps | 19 |
| 10 Mbps | 100 |

---

## STP 프로토콜 종류

| 프로토콜 | 표준 | 특징 |
|---------|------|------|
| CST (Common Spanning Tree) | 802.1D | 물리적 연결 기반으로 하나의 포트를 Block. 전체 네트워크에 STP 인스턴스 1개 |
| PVST (Per VLAN Spanning Tree) | Cisco 전용 | VLAN별로 STP 인스턴스를 운영. CST와 호환 안 됨 |
| PVST+ | Cisco 전용 | PVST의 확장판. 802.1Q 트렁크 지원. CST와 호환 가능 |
| RSTP (Rapid STP) | 802.1w | STP의 개선 버전. 수렴 속도가 수 밀리초~수 초로 빠름 |
| PVRST (Per VLAN Rapid STP) | Cisco 전용 | RSTP + VLAN별 인스턴스 |
| MSTP (Multiple STP) | 802.1s | 여러 VLAN을 그룹으로 묶어 STP 운영. BPDU 부하 감소 |

---

## STP vs RSTP 비교

| 구분 | STP (802.1D) | RSTP (802.1w) |
|------|-------------|---------------|
| 수렴 속도 | 30~50초 | 수 밀리초 ~ 수 초 |
| 포트 상태 | 5단계 (Disabled/Blocking/Listening/Learning/Forwarding) | 3단계 (Discarding/Learning/Forwarding) |
| 백업 포트 | Blocked Port (구분 없음) | Alternate Port / Backup Port (명시적) |
| 링크 업 시 | 타이머 대기 | 즉시 협상 |
| 토폴로지 변경 | 느린 반응 | 빠른 핸드셰이크 |

### RSTP 포트 역할

| 포트 | 역할 | 평소 상태 | 설명 |
|------|------|----------|------|
| Root Port (RP) | Root로 가는 최단 경로 | Forwarding | 주력 출구 |
| Designated Port (DP) | 세그먼트 대표 | Forwarding | 해당 구간 통신 담당 |
| Alternate Port (AP) | RP의 백업 | Discarding | RP 장애 시 즉시 Forwarding 승격 |
| Backup Port (BP) | 같은 스위치 이중 연결 시 예비 | Discarding | 드문 케이스 |

### RSTP 포트 역할 예시

```
        [ Root Bridge ]
             |
           (DP)
             |
          [SW1]
          /    \
       (RP)   (AP)  ← RP 장애 시 즉시 승격
        |      |
      [SW2]--[SW3]
           (DP)
```

- Alternate Port는 평소 Discarding 상태지만 BPDU는 계속 수신
- Root Port가 다운되면 대기 없이 바로 Forwarding으로 전환
- STP에서는 이 과정에 30~50초 걸리지만 RSTP에서는 거의 즉시 처리됨

---

## 실습: Root Bridge 선출 및 변경

### 토폴로지

```
        +-------+
        |  SW2  |
        +---+---+
       f0/1 | f0/2
            |
     f0/2   |   f0/2
  +----+----+----+----+
  |   SW1   |   SW3   |
  +----+----+----+----+
     f0/1        f0/1
       \          /
        \        /
     f0/1  f0/2
      +---+---+
      | Switch|
      +-------+
```

### 스위치별 MAC Address 및 Bridge ID

| 스위치 | MAC Address | 기본 Priority | Bridge ID |
|--------|------------|--------------|-----------|
| SW1 | 0004.9A76.8182 | 32769 | 32769 + MAC |
| SW2 | 0090.21D3.D807 | 32769 | 32769 + MAC |
| SW3 | 0004.9A66.7111 | 32769 | 32769 + MAC |

Priority가 동일하므로 MAC Address가 가장 낮은 SW3(0004.9A66.7111)이 Root Bridge로 선출.

---

### 1단계. 현재 STP 상태 확인

```
show spanning-tree
```

출력 예시 (SW1):
```
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0004.9A66.7111
             Cost        19
             Port        2(FastEthernet0/2)

  Bridge ID  Priority    32769
             Address     0004.9A76.8182
```

- Root ID: Root Bridge의 정보 (SW3의 MAC이 표시됨)
- Bridge ID: 자기 자신의 정보
- Root Port: f0/2 (Root Bridge 방향)

### 2단계. Root Bridge 수동 변경

`spanning-tree vlan 1 root primary` 명령을 사용하면 현재 Root Bridge보다 Priority를 낮게 자동 조정한다. Priority는 4096의 배수 단위로 변경됨.

**SW2에서 실행:**
```
conf t
spanning-tree vlan 1 root primary
end
show spanning-tree
```

결과:
```
Root ID    Priority    24577    ← 기존 32769보다 낮아짐
           Address     0090.21D3.D807
           This bridge is the root   ← SW2가 Root Bridge가 됨
```

### Root Primary 반복 시 Priority 변화

`root primary`를 각 스위치에서 반복 실행하면, 현재 Root보다 항상 낮은 값으로 설정되기 때문에 Priority가 4096씩 줄어든다:

```
실행 순서    스위치     Priority 변화
1번째       SW2       32769 → 24577
2번째       SW3       32769 → 20481
3번째       SW1       32769 → 16385
4번째       SW2       24577 → 12289
5번째       SW3       20481 → 8193
6번째       SW1       16385 → 4097
7번째       SW2       12289 → 1       ← 최솟값 (Priority 0 + sys-id-ext 1)
```

Priority가 0이 되면 더 이상 낮출 수 없다. 여러 스위치가 모두 0이 되면 MAC Address로 Root Bridge가 결정됨.

### 3단계. Root Secondary 설정

Root Bridge가 다운될 때를 대비해서 백업 Root Bridge를 지정할 수 있다. Priority를 28672로 설정함.

```
conf t
spanning-tree vlan 1 root secondary
end
show spanning-tree
```

결과:
```
Bridge ID  Priority    28673  (priority 28672 sys-id-ext 1)
```

현재 Root Bridge(Priority가 더 낮은 스위치)가 정상이면 Secondary는 Root가 되지 않지만, Root Bridge가 다운되면 Secondary가 Root Bridge를 대신한다.

---

## STP 타이머

| 타이머 | 기본값 | 설명 |
|--------|--------|------|
| Hello Time | 2초 | BPDU 전송 주기 |
| Max Age | 20초 | BPDU를 수신하지 못하면 토폴로지 변경으로 판단하는 시간 |
| Forward Delay | 15초 | Listening → Learning, Learning → Forwarding 각 단계의 대기 시간 |

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show spanning-tree` | STP 전체 상태 (Root ID, Bridge ID, 포트 역할/상태) |
| `show spanning-tree vlan [번호]` | 특정 VLAN의 STP 상태 |
| `show spanning-tree summary` | STP 요약 정보 |
| `show version` | 스위치 모델, IOS 버전, MAC Address 확인 |

---

## 설정 명령어 정리

```
-- Root Bridge로 지정 (Primary)
spanning-tree vlan [VLAN번호] root primary

-- 백업 Root Bridge로 지정 (Secondary)
spanning-tree vlan [VLAN번호] root secondary

-- Priority 직접 지정 (4096의 배수)
spanning-tree vlan [VLAN번호] priority [값]
```

---

## 정리

- STP는 스위치 이중화 환경에서 루프를 방지하기 위해 특정 포트를 차단하는 프로토콜
- BPDU를 통해 Bridge ID를 교환하고, 가장 낮은 Bridge ID를 가진 스위치가 Root Bridge로 선출됨
- Root Port는 Root Bridge로 가는 최적 경로, Designated Port는 세그먼트 대표, 나머지는 Blocked
- STP는 수렴에 30~50초가 걸리는 반면, RSTP는 Alternate Port를 활용해 거의 즉시 복구 가능
- 실무에서는 RSTP 또는 PVST+를 주로 사용하며, `root primary`로 Root Bridge를 의도적으로 지정하는 것이 일반적
