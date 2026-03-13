# RADIUS (Remote Authentication Dial-In User Service)

RADIUS는 네트워크에서 **AAA(Authentication, Authorization, Accounting)** 기능을 제공하는 프로토콜이다.  
주로 **기업 WiFi, VPN, ISP 인증 시스템**에서 사용된다.

```
RADIUS = Authentication + Authorization + Accounting
```

대표적으로 **네트워크 접속 사용자 인증을 중앙 서버에서 관리**하는 역할을 한다.

---

# 1. RADIUS가 필요한 이유

여러 네트워크 장비(AP, Switch, VPN 등)에 각각 사용자 계정을 만들면  
관리 포인트가 많아져 관리가 어려워진다.

RADIUS는 **중앙 인증 서버**를 통해 이를 해결한다.

## Without RADIUS

```
User → AP1 (local account)
User → AP2 (local account)
User → VPN (local account)

각 장비마다 계정 관리 필요
```

## With RADIUS

```
User → Network Device → RADIUS Server → Authentication Result
```

```
                +-------------------+
User ---------->| Network Device    |
                | (AP / VPN / SW)   |
                +---------+---------+
                          |
                          |
                          v
                +-------------------+
                |   RADIUS Server   |
                | Authentication DB |
                +-------------------+
```

---

# 2. RADIUS AAA 개념

## 2.1 Authentication (인증)

사용자가 **누구인지 확인하는 과정**

```
username: user1
password: password123
```

인증 방식 예시

```
- ID / Password
- Certificate
- OTP
- Token
```

---

## 2.2 Authorization (권한 부여)

인증된 사용자에게 **어떤 네트워크 권한을 줄지 결정**

```
Example Policy

user1 → VLAN 10
user2 → VLAN 20
guest → Internet only
```

예시

```
Access-Accept
VLAN-ID = 10
```

---

## 2.3 Accounting (사용 기록)

사용자의 **접속 정보를 기록**

```
Accounting Log Example

User: user1
Login Time: 2026-03-13 09:10
Logout Time: 2026-03-13 10:30
Traffic: 2.1GB
```

---

# 3. RADIUS 동작 과정

## Step 1

```
User connects to WiFi or VPN
```

## Step 2

```
Network Device → RADIUS Server

Access-Request
```

## Step 3

```
RADIUS checks user credentials
```

## Step 4

```
RADIUS → Network Device

Access-Accept
or
Access-Reject
```

## Step 5

```
Network Device allows or blocks access
```

전체 흐름

```
+--------+
|  User  |
+---+----+
    |
    v
+----------------+
| Network Device |
|  AP / VPN / SW |
+-------+--------+
        |
        | Access-Request
        v
+----------------+
|  RADIUS Server |
+-------+--------+
        |
        | Access-Accept / Reject
        v
+----------------+
| Network Device |
+----------------+
```

---

# 4. 주요 RADIUS 메시지

```
Access-Request
```

```
Access-Accept
```

```
Access-Reject
```

```
Accounting-Request
```

```
Accounting-Response
```

---

# 5. RADIUS 구성 요소

## RADIUS Client

네트워크 장비

```
- Access Point
- VPN Gateway
- Network Switch
```

## RADIUS Server

인증 처리 서버

```
- FreeRADIUS
- Microsoft NPS
- Cisco ISE
```

## Authentication Database

```
- Active Directory
- LDAP
- Local DB
```

---

# 6. 실제 사용 예

## Enterprise WiFi (802.1X)

```
User → WiFi AP → RADIUS → Authentication
```

## VPN Login

```
User → VPN Gateway → RADIUS
```

## ISP Internet Authentication

```
User → ISP Router → RADIUS
```

---

# 7. RADIUS 기본 포트

```
Authentication : UDP 1812
Accounting     : UDP 1813
```

구버전 포트

```
1645
1646
```

---

# 8. 요약

```
RADIUS는 네트워크 사용자 인증을 중앙 서버에서 관리하는 AAA 프로토콜이다.
```

```
Authentication → 사용자 확인
Authorization  → 권한 부여
Accounting     → 접속 기록


# Cisco RADIUS AAA Configuration

## Topology

```
Client (192.168.10.10)
        |
        |
      ESW1
   VLAN1: 192.168.10.254
      /   \
     /     \
   R1     RADIUS Server
192.168.10.1   192.168.10.100
```

- Client: 192.168.10.10
- Router (R1): 192.168.10.1
- Switch (ESW1 VLAN1): 192.168.10.254
- RADIUS Server: 192.168.10.100


# 1. R1 (Router) Configuration

R1에서 **RADIUS 기반 Telnet 로그인 인증**을 설정한다.

```
enable
configure terminal
```

## AAA 활성화

```
aaa new-model
```

## RADIUS 인증 그룹 생성

```
aaa authentication login TELNET group radius
```

## RADIUS 서버 등록

```
radius-server host 192.168.10.100 auth-port 1812 key aws
```

## RADIUS 패킷 Source Interface 지정

```
ip radius source-interface f0/0
```

## VTY 라인 설정 (Telnet 로그인 시 RADIUS 사용)

```
line vty 0 4
login authentication TELNET
transport input telnet
```

## 권한 레벨 설정 (선택)

```
privilege exec level 15 show running-config
```

---

# 2. R1 RADIUS 인증 테스트

```
test aaa group radius admin admin legacy
```

성공 시

```
User successfully authenticated
```

---

# 3. ESW1 (Switch) Configuration

ESW1에서도 동일하게 **RADIUS 로그인 인증**을 설정한다.

```
enable
configure terminal
```

## VLAN 인터페이스 설정

```
interface vlan 1
ip address 192.168.10.254 255.255.255.0
no shutdown
exit
```

## AAA 활성화

```
aaa new-model
```

## RADIUS 인증 설정

```
aaa authentication login TELNET group radius
```

## RADIUS 서버 등록

```
radius-server host 192.168.10.100 auth-port 1812 key aws
```

## Source Interface 설정

```
ip radius source-interface vlan 1
```

## VTY 설정

```
line vty 0 4
login authentication TELNET
transport input telnet
```

---

# 4. 테스트

Client에서 Telnet 접속 테스트

```
telnet 192.168.10.1
```

또는

```
telnet 192.168.10.254
```

로그인 시 **RADIUS 서버 인증을 통해 접속**된다.

---

# 5. 인증 흐름

```
Client
  |
  | Telnet Login
  v
Router / Switch
  |
  | RADIUS Access-Request
  v
RADIUS Server (192.168.10.100)
  |
  | Access-Accept / Reject
  v
Router / Switch
  |
  v
Login Success or Fail
```


# 6. 핵심 명령어 요약

```
aaa new-model
aaa authentication login TELNET group radius
radius-server host 192.168.10.100 auth-port 1812 key aws
ip radius source-interface f0/0
line vty 0 4
login authentication TELNET
```



# 7. Troubleshooting

디버그로 RADIUS 패킷 확인

```
debug radius authentication
```

또는

```
debug aaa authentication
```
