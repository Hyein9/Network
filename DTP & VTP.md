# DTP & VTP 실습

## 개념 정리

### DTP (Dynamic Trunking Protocol)
- 스위치 포트 간 Trunk 연결을 자동으로 협상하는 Cisco 전용 프로토콜
- 양쪽 포트의 DTP 모드 조합에 따라 Access 또는 Trunk로 결정됨
- 보안상 수동으로 mode access 또는 mode trunk를 지정하는 것을 권장

### DTP 모드

| 모드 | 설명 | 기본 장비 |
|------|------|----------|
| Dynamic Auto | 소극적 협상. 상대가 Trunk 또는 Desirable이 아니면 Access | L2 스위치 기본값 |
| Dynamic Desirable | 적극적 협상. 상대가 Access가 아니면 Trunk | L3 스위치 기본값 |
| Trunk | 무조건 Trunk 동작 | 수동 설정 |
| Access | 무조건 Access 동작 | 수동 설정 |

### DTP 협상 결과표

```
               상대 포트
자신 포트       Auto        Desirable    Trunk       Access
─────────────────────────────────────────────────────────────
Auto           Access      Trunk        Trunk       Access
Desirable      Trunk       Trunk        Trunk       Access
Trunk          Trunk       Trunk        Trunk       제한적
Access         Access      Access       제한적       Access
```

핵심: Auto끼리 만나면 둘 다 소극적이라 Trunk가 성립되지 않고 Access로 동작.

### DTP 비활성화

보안상 DTP 협상을 아예 끄려면:
```
switchport nonegotiate
```
이 경우 반드시 수동으로 mode trunk 또는 mode access를 지정해야 함.

---

### VTP (VLAN Trunking Protocol)
- 스위치 간 VLAN 정보를 자동으로 동기화하는 Cisco 전용 프로토콜
- 하나의 스위치(Server)에서 VLAN을 생성하면 같은 도메인의 다른 스위치에 자동 전파
- 동기화 메시지는 Trunk 포트를 통해서만 전달됨

### VTP 동작 조건
- 같은 VTP Domain Name 사용
- 같은 VTP Password 사용
- 스위치 간 Trunk 연결 필수
- VTP는 기본적으로 동작하며 끌 수 없음

### VTP 모드 3가지

| 모드 | VLAN 생성 | 동기화 수신 | 동기화 전달 | VLAN 범위 | Revision |
|------|:--------:|:----------:|:----------:|----------|----------|
| Server | O | O | O | 1~1005 | 증가 |
| Client | X | O | O | - | 수신값 반영 |
| Transparent | O | X (무시) | O (전달만) | 1~4094 | 항상 0 |

- **Server**: VLAN을 직접 생성/수정/삭제 가능. 변경 시 Revision Number가 증가하며 동기화 전파
- **Client**: Server로부터 VLAN 정보를 수신만 함. 직접 VLAN 생성 불가
- **Transparent**: 자기만의 VLAN을 독립적으로 관리. VTP 광고는 다른 스위치에 전달(중계)하지만, 자신의 DB에는 반영하지 않음

### VTP 주의사항
- VLAN 설정 정보는 동기화되지만 포트 할당 정보는 동기화되지 않음
- 확장 VLAN(1006~4094)은 Transparent 모드에서만 사용 가능
- 보안상 Transparent 모드를 권장하는 경우가 많음
- Revision Number가 높은 스위치의 정보가 우선되므로, 신규 스위치 투입 시 주의 필요

### VTP 설정 권장 순서
1. Password 설정
2. Domain 설정
3. Mode 설정

스위치 전체 설정 순서:
1. VTP 모드 설정
2. Trunk 포트 설정
3. VLAN 생성
4. 포트에 VLAN 할당

---

## 실습 토폴로지

```
  +------------------+       Trunk        +------------------+       Trunk        +------------------+
  |       SW1        +------(f0/1)-------+       SW2        +------(f0/2)-------+       SW3        |
  |   VTP Server     |                   |  VTP Transparent  |                   |   VTP Client     |
  |                  |                   |                   |                   |                  |
  |  VLAN 10,20,     |                   |  VLAN 99          |                   |  VLAN 자동 수신   |
  |  30,40 생성       |                   |  (독립 관리)       |                   |  (생성 불가)       |
  +------------------+                   +------------------+                   +------------------+
         Domain: AWS                           Domain: AWS                           Domain: AWS
         Password: 0213                        Password: 0213                        Password: 0213
```

### VTP 동기화 흐름

```
  [SW1 - Server]                [SW2 - Transparent]           [SW3 - Client]
       |                              |                             |
  VLAN 10,20,30,40 생성               |                             |
  Revision: 4                         |                             |
       |                              |                             |
       +--- VTP 광고 (Trunk) -------->|                             |
                                      | 자기 DB 반영 안 함            |
                                      | 그대로 전달 (중계)            |
                                      +--- VTP 광고 (Trunk) ------->|
                                                                    |
                                                              VLAN 10,20,30,40
                                                              자동 생성됨
                                                              Revision: 4
```

---

## 1단계. SW1 설정 (VTP Server)

```
enable
conf t
hostname SW1

vtp mode server
vtp domain AWS
vtp password 0213

interface range f0/1 - 2
 switchport mode trunk

vlan 10
 name AWS
vlan 20
 name 505
vlan 30
vlan 40

end
write
```

확인:
```
show vtp status
show vlan brief
```

정상 출력:
```
VTP Version                     : 2
Configuration Revision          : 4
VTP Operating Mode              : Server
VTP Domain Name                 : AWS
```

VLAN 10, 20, 30, 40이 생성되어 있어야 함.

---

## 2단계. SW2 설정 (VTP Transparent)

```
enable
conf t
hostname SW2

vtp mode transparent
vtp domain AWS
vtp password 0213

interface range f0/1 - 2
 switchport mode trunk

vlan 99

end
write
```

확인:
```
show vtp status
show vlan brief
```

정상 결과:
- VTP Operating Mode: Transparent
- Configuration Revision: 0 (항상 0)
- VLAN 99는 SW2에만 존재
- SW1의 VLAN 10, 20, 30, 40은 SW2의 VLAN 목록에 나타나지 않음

---

## 3단계. SW3 설정 (VTP Client)

```
enable
conf t
hostname SW3

vtp mode client
vtp domain AWS
vtp password 0213

interface range f0/1 - 2
 switchport mode trunk

end
write
```

확인:
```
show vtp status
show vlan brief
```

정상 결과:
- SW1에서 만든 VLAN 10, 20, 30, 40이 자동으로 생성됨
- SW2의 VLAN 99는 존재하지 않음 (Transparent는 자기 VLAN을 전파하지 않음)
- Client 모드에서 직접 VLAN 생성을 시도하면 에러 발생:
```
SW3(config)# vlan 10
VTP VLAN configuration not allowed when device is in CLIENT mode.
```

---

## DTP 실습

### DTP 모드별 설정

**Dynamic Auto (L2 스위치 기본값)**
```
interface f0/1
 switchport mode dynamic auto
```

**Dynamic Desirable (L3 스위치 기본값)**
```
interface f0/1
 switchport mode dynamic desirable
```

**수동 Trunk (권장)**
```
interface f0/1
 switchport mode trunk
 switchport nonegotiate
```

**수동 Access (권장)**
```
interface f0/1
 switchport mode access
 switchport nonegotiate
```

### DTP 협상 확인

```
show interfaces f0/1 switchport
```

출력에서 확인할 항목:
```
Administrative Mode: dynamic auto
Operational Mode: trunk (또는 static access)
Negotiation of Trunking: On (또는 Off)
```

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show vtp status` | VTP 모드, 도메인, Revision 확인 |
| `show vtp password` | VTP 패스워드 확인 |
| `show vlan brief` | VLAN 목록 및 포트 할당 |
| `show interfaces trunk` | Trunk 포트 상태 |
| `show interfaces f0/1 switchport` | 특정 포트의 DTP/Trunk 상태 |

---

## 실습에서 자주 발생하는 문제

| 문제 | 원인 | 해결 |
|------|------|------|
| Client에서 VLAN 안 생김 | Trunk 연결 안 됨 | 스위치 간 Trunk 설정 확인 |
| VTP 동기화 안 됨 | Domain 또는 Password 불일치 | 양쪽 동일하게 설정 |
| 신규 스위치가 기존 VLAN 덮어씀 | 신규 스위치의 Revision이 더 높음 | 투입 전 Transparent로 변경 후 Revision 초기화 |
| Client에서 VLAN 생성 시도 시 에러 | Client 모드는 VLAN 생성 불가 | Server 또는 Transparent로 변경 |

---

## 정리

- DTP는 스위치 포트 간 Trunk를 자동 협상하는 프로토콜이지만, 보안상 수동 설정(mode trunk/access + nonegotiate)을 권장
- VTP는 Server에서 생성한 VLAN을 같은 도메인의 스위치에 자동 전파하는 프로토콜
- Server는 VLAN 생성/전파, Client는 수신만, Transparent는 독립 관리 + 중계만 수행
- VTP 사용 시 Domain, Password, Trunk 연결이 모두 일치해야 동기화가 이루어짐
- Revision Number 관리에 주의하지 않으면 의도치 않게 VLAN 정보가 덮어씌워질 수 있음
