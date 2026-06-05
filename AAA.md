# AAA (Authentication, Authorization, Accounting) 실습

## 개념 정리

### AAA란
- 네트워크 장비에 대한 접근을 관리하고 기록하는 보안 프레임워크
- 라우터, 스위치, 서버 등 네트워크 장비에 접근할 때 누가(Authentication), 무엇을(Authorization), 언제(Accounting) 했는지를 제어하고 기록함

### AAA 3요소

| 요소 | 역할 | 설명 |
|------|------|------|
| Authentication (인증) | 사용자가 누구인지 확인 | 아이디/비밀번호, OTP, 인증서 등으로 신원 확인 |
| Authorization (권한 부여) | 무엇을 할 수 있는지 결정 | 인증된 사용자에게 허용된 명령어, 접근 범위를 제한 |
| Accounting (기록) | 활동 내역을 기록 | 접속 시간, 실행 명령어, 접속 IP 등을 로그로 남김 |

### AAA 동작 흐름

```
  사용자가 라우터에 접속 시도
          |
          v
  +-------+--------+
  | Authentication  |  사용자 이름/비밀번호 확인
  | (인증)          |  → 실패 시 접속 거부
  +-------+--------+
          |
          v (인증 성공)
  +-------+--------+
  | Authorization   |  권한 레벨 확인
  | (권한 부여)     |  → Admin이면 모든 명령 가능
  +-------+--------+  → User이면 show 명령만 가능
          |
          v
  +-------+--------+
  | Accounting      |  활동 기록
  | (기록)          |  → 접속 시간, 실행 명령어 로그 저장
  +-------+--------+
```

### AAA 인증 방식

| 방식 | 설명 |
|------|------|
| Local | 장비 자체에 저장된 사용자 DB로 인증. 소규모 환경에 적합 |
| RADIUS | 외부 AAA 서버로 인증. UDP 기반. 인증+권한을 한 번에 처리 |
| TACACS+ | Cisco 전용 외부 AAA 서버. TCP 기반. 인증/권한/기록을 개별 제어 가능 |

### RADIUS vs TACACS+ 비교

| 구분 | RADIUS | TACACS+ |
|------|--------|---------|
| 표준/벤더 | IETF 표준 (RFC 2865) | Cisco 전용 |
| 전송 프로토콜 | UDP (1812/1813) | TCP (49) |
| 암호화 범위 | 비밀번호만 암호화 | 전체 패킷 암호화 |
| AAA 분리 | 인증+권한 결합 | 인증/권한/기록 개별 제어 |
| 명령어 제어 | 제한적 | 명령어 단위로 세밀한 권한 제어 가능 |
| 주 사용처 | ISP, 무선 인증 | 네트워크 장비 관리 |

### Privilege Level (권한 레벨)

Cisco 장비에서 사용자에게 할당할 수 있는 권한 수준:

| 레벨 | 접근 범위 | 예시 |
|------|----------|------|
| 0 | 매우 제한적 | logout, enable, disable 등 최소 명령만 |
| 1 | User EXEC | show 명령 일부 (기본 접속 레벨) |
| 15 | Privileged EXEC | 모든 명령 사용 가능 (enable 후 상태) |
| 2~14 | 커스텀 | 관리자가 직접 명령어를 할당하여 사용 |

---

## 실습 토폴로지

### Local AAA 구성

```
  [Admin PC]        [User PC]
  192.168.10.10     192.168.10.20
       |                 |
       +--------+--------+
                |
              [SW1]
                |
              g0/0
         +----+----+
         |   R1    |
         | AAA     |
         | Server  |
         | (Local) |
         +---------+
       192.168.10.1
```

### TACACS+ 서버 연동 구성

```
  [Admin PC]        [User PC]                [TACACS+ Server]
  192.168.10.10     192.168.10.20            192.168.10.100
       |                 |                        |
       +--------+--------+------------------------+
                |
              [SW1]
                |
              g0/0              g0/1
         +----+----+-----------+----+
         |          R1               |
         |       AAA 적용            |
         |  Local + TACACS+          |
         +---------------------------+
       192.168.10.1            192.168.20.1
                                    |
                                  [R2]
                               (외부 라우터)
```

### IP 주소 구성

| 장비 | 인터페이스 | IP 주소 | 역할 |
|------|-----------|---------|------|
| R1 | g0/0 | 192.168.10.1/24 | LAN 게이트웨이 |
| R1 | g0/1 | 192.168.20.1/24 | 외부 연결 |
| R2 | g0/0 | 192.168.20.2/24 | 외부 라우터 |
| Admin PC | - | 192.168.10.10/24 | 관리자 접속용 |
| User PC | - | 192.168.10.20/24 | 일반 사용자 접속용 |
| TACACS+ Server | - | 192.168.10.100/24 | 인증 서버 |

---

## 실습 1: Local AAA 인증 구성

외부 서버 없이 라우터 자체에 사용자 DB를 만들어 인증하는 방식.

### 1단계. 기본 인터페이스 설정

**R1**
```
enable
conf t
hostname R1

interface g0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface g0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
end
```

### 2단계. 로컬 사용자 생성

```
conf t
username admin privilege 15 secret admin123
username user1 privilege 1 secret user123
```

- `admin`: 관리자 계정. privilege 15로 모든 명령 사용 가능
- `user1`: 일반 사용자. privilege 1로 제한된 명령만 가능
- `secret`: 비밀번호를 MD5로 암호화하여 저장 (password보다 안전)

### 3단계. AAA 활성화 및 인증 설정

```
aaa new-model
aaa authentication login default local
aaa authorization exec default local
aaa accounting exec default start-stop group tacacs+
```

각 줄 설명:
- `aaa new-model`: AAA 프레임워크 활성화. 이 명령 이후 기존 line password 방식이 비활성화됨
- `aaa authentication login default local`: 로그인 시 로컬 DB로 인증
- `aaa authorization exec default local`: 인증 후 로컬 DB의 privilege 레벨로 권한 부여
- `aaa accounting exec default start-stop group tacacs+`: 접속 시작/종료를 TACACS+ 서버에 기록 (서버 없으면 로컬 로그)

### 4단계. 콘솔 및 VTY(원격 접속) 설정

```
line console 0
 login authentication default

line vty 0 4
 transport input ssh telnet
 login authentication default
end
```

### 5단계. SSH 설정 (원격 접속용)

```
conf t
ip domain-name lab.local
crypto key generate rsa modulus 2048
ip ssh version 2
end
```

### 동작 확인

Admin PC에서 R1으로 SSH 접속:
```
ssh -l admin 192.168.10.1
Password: admin123
```

접속 후:
```
R1# show privilege
Current privilege level is 15
```

user1로 접속 시:
```
ssh -l user1 192.168.10.1
Password: user123
```

접속 후:
```
R1> show privilege
Current privilege level is 1
```

user1은 enable 없이는 설정 변경이 불가능하다.

---

## 실습 2: TACACS+ 서버 연동

외부 TACACS+ 서버를 사용하여 중앙 집중식 인증을 구성하는 방식. 장비가 많을수록 효율적.

### 인증 흐름

```
  [Admin PC]                [R1]                  [TACACS+ Server]
       |                     |                          |
       |  SSH 접속 시도       |                          |
       +-------------------->|                          |
       |                     |  인증 요청 (TCP 49)       |
       |                     +------------------------->|
       |                     |                          |
       |                     |  인증 결과 (성공/실패)     |
       |                     |<-------------------------+
       |                     |                          |
       |  접속 허용/거부      |  권한 조회 요청           |
       |<--------------------+------------------------->|
       |                     |                          |
       |                     |  권한 정보 응답           |
       |                     |<-------------------------+
       |                     |                          |
       |  명령어 실행         |  Accounting 기록 전송     |
       |                     +------------------------->|
```

### 1단계. TACACS+ 서버 지정

```
conf t
tacacs-server host 192.168.10.100 key TACACSKEY
```

- `host`: TACACS+ 서버 IP
- `key`: 라우터와 서버 간 공유 키 (양쪽 동일해야 함)

### 2단계. AAA 인증 설정 (TACACS+ 우선, Local 백업)

```
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+
```

- `group tacacs+ local`: TACACS+ 서버에 먼저 인증 시도. 서버 장애 시 로컬 DB로 인증 (Fallback)
- `accounting commands 15`: privilege 15 레벨에서 실행한 명령어를 기록

### 3단계. 콘솔 및 VTY 설정

```
line console 0
 login authentication default

line vty 0 4
 transport input ssh telnet
 login authentication default
end
```

### TACACS+ 서버 장애 시 동작

```
  [R1] → TACACS+ 서버에 인증 요청
           |
           | 서버 응답 없음 (timeout)
           v
  [R1] → Local DB로 인증 시도 (Fallback)
           |
           | admin / admin123 확인
           v
        접속 허용
```

서버가 다운되어도 로컬 계정으로 접속할 수 있도록 `local`을 백업으로 설정하는 것이 중요하다.

---

## 실습 3: 커스텀 인증 목록 (Named Method List)

기본 인증 외에 특정 접속 방식에 다른 인증 방법을 적용할 수 있다.

### 콘솔은 로컬, VTY는 TACACS+ 사용

```
conf t
aaa new-model

aaa authentication login CONSOLE-AUTH local
aaa authentication login VTY-AUTH group tacacs+ local

line console 0
 login authentication CONSOLE-AUTH

line vty 0 4
 transport input ssh telnet
 login authentication VTY-AUTH
end
```

- `CONSOLE-AUTH`: 콘솔 접속은 로컬 DB만 사용
- `VTY-AUTH`: 원격 접속은 TACACS+ 우선, 실패 시 로컬 DB

이렇게 접속 방식별로 다른 인증 정책을 적용할 수 있다.

---

## 실습 4: 명령어 권한 제어 (Authorization)

특정 사용자에게 제한된 명령어만 허용하는 설정.

### Privilege Level 커스텀 설정

```
conf t
username operator privilege 7 secret oper123

privilege exec level 7 show running-config
privilege exec level 7 show ip route
privilege exec level 7 show ip interface brief
privilege exec level 7 ping
```

operator 계정은 privilege 7로 접속하며, 위에서 지정한 명령어만 사용 가능. 다른 설정 명령은 실행할 수 없다.

### View 기반 권한 제어 (Parser View)

더 세밀한 제어가 필요하면 Parser View를 사용:

```
conf t
aaa new-model
enable secret class123

parser view MONITOR-VIEW
 secret monitor123
 commands exec include show
 commands exec include ping
 commands exec include traceroute
end
```

MONITOR-VIEW로 접속한 사용자는 show, ping, traceroute만 사용 가능.

```
R1# enable view MONITOR-VIEW
Password: monitor123
R1# show ip route       → 성공
R1# configure terminal  → 거부
```

---

## Accounting 로그 예시

TACACS+ 서버에 기록되는 로그:

```
User: admin
Source IP: 192.168.10.10
Login Time: 2025-06-05 09:00:15
Logout Time: 2025-06-05 09:25:42
Commands:
  09:00:20  show running-config
  09:05:11  configure terminal
  09:05:15  interface g0/0
  09:05:20  ip address 10.0.0.1 255.255.255.0
  09:25:40  exit
```

이 기록을 통해 누가, 언제, 어떤 작업을 했는지 추적할 수 있다. 보안 감사나 장애 원인 분석에 활용.

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show aaa sessions` | 현재 AAA 세션 확인 |
| `show aaa user all` | AAA 사용자 정보 |
| `show privilege` | 현재 접속 계정의 권한 레벨 |
| `show running-config \| section aaa` | AAA 관련 설정만 필터링 |
| `show running-config \| section line` | line 설정 확인 |
| `show users` | 현재 접속 중인 사용자 목록 |
| `debug aaa authentication` | AAA 인증 과정 실시간 디버깅 |
| `debug aaa authorization` | AAA 권한 부여 과정 디버깅 |
| `debug tacacs` | TACACS+ 통신 디버깅 |

---

## 설정 명령어 정리

```
-- AAA 활성화
aaa new-model

-- 인증 설정
aaa authentication login [이름|default] [group tacacs+|group radius|local|enable]

-- 권한 설정
aaa authorization exec [이름|default] [group tacacs+|local]

-- 기록 설정
aaa accounting exec [이름|default] start-stop group [tacacs+|radius]
aaa accounting commands [레벨] [이름|default] start-stop group [tacacs+|radius]

-- 로컬 사용자 생성
username [이름] privilege [레벨] secret [비밀번호]

-- TACACS+ 서버 지정
tacacs-server host [IP] key [공유키]

-- RADIUS 서버 지정
radius-server host [IP] key [공유키]

-- SSH 설정
ip domain-name [도메인]
crypto key generate rsa modulus 2048
ip ssh version 2
```

---

## 정리

- AAA는 인증(Authentication), 권한(Authorization), 기록(Accounting)으로 구성된 보안 프레임워크
- Local 인증은 장비 자체 DB를 사용하고, TACACS+/RADIUS는 외부 서버로 중앙 집중 관리
- TACACS+는 TCP 기반으로 전체 패킷 암호화, 명령어 단위 권한 제어가 가능하여 네트워크 장비 관리에 적합
- RADIUS는 UDP 기반으로 ISP나 무선 인증 환경에서 많이 사용
- 서버 장애 대비를 위해 `group tacacs+ local`처럼 로컬 인증을 백업으로 설정하는 것이 필수
- Privilege Level이나 Parser View를 활용하면 사용자별로 사용 가능한 명령어를 세밀하게 제어할 수 있음
