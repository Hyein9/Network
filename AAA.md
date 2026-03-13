# Network AAA (Authentication, Authorization, Accounting)

## 1. AAA 개요
AAA는 네트워크 보안에서 **사용자의 접근을 관리하고 기록하는 보안 프레임워크**입니다.  
AAA는 다음 세 가지 요소로 구성됩니다.

- **Authentication (인증)**: 사용자나 관리자가 맞는지 증명하는 과정으로 일반적으로 사용자 이름과 패스워드가 맞는지 확인
- **Authorization (권한 부여)**: 인증된 사용자나 관리자에게 허가된 자원 또한 권한을 제공하는 과정
- **Accounting (계정 관리 / 기록)**: 사용자의 활동을 기록하고 추적, 사용자나 관리자의 접속 여부, 접속 후 행위, 접속 시간 등에 대해 기록을 남기고 후에 감사하기 위한 용도로 사용

주로 네트워크 장비(라우터, 스위치, 서버)에 접근할 때 사용됩니다.

---

# 2. Authentication (인증)

Authentication은 **사용자의 신원을 확인하는 과정**입니다.

### 예시
- 아이디 / 비밀번호 로그인
- OTP (One-Time Password)
- 생체 인증
- SSH Key 인증

### 네트워크 예시

관리자가 라우터에 접속할 때:
Username: admin
Password: ********

시스템은 입력된 정보가 **데이터베이스 또는 인증 서버와 일치하는지 확인**합니다.

---

# 3. Authorization (권한 부여)

Authorization은 **인증된 사용자가 어떤 작업을 할 수 있는지 결정**하는 과정입니다.

예를 들어:

| 사용자 | 권한 |
|------|------|
| Admin | 모든 설정 변경 가능 |
| Operator | 설정 조회만 가능 |
| Guest | 제한된 접근 |

### 예시

- Admin → 네트워크 설정 변경 가능  
- User → 설정 조회만 가능  

즉,
인증 → 성공
권한 확인 → 허용된 작업만 가능


---

# 4. Accounting (기록 / 감사)

Accounting은 **사용자의 활동을 기록하는 기능**입니다.

기록되는 정보 예:

- 로그인 시간
- 로그아웃 시간
- 실행한 명령어
- 접속한 IP

### 예시 로그


User: admin
Login Time: 2025-09-29 10:00
Command: show running-config
Logout Time: 2025-09-29 10:20


이 기능은 다음과 같은 목적에 사용됩니다.

- 보안 감사
- 문제 추적
- 사용 기록 관리

---

# 5. AAA 동작 흐름

AAA는 일반적으로 다음 순서로 동작합니다.


Authentication (사용자 인증)
↓

Authorization (권한 확인)
↓

Accounting (활동 기록)


예시:


User → Router Login
↓
Authentication → 사용자 확인
↓
Authorization → 권한 확인
↓
Accounting → 활동 기록


---

# 6. AAA에서 사용하는 프로토콜

대표적인 AAA 프로토콜:

| 프로토콜 | 설명 |
|------|------|
| RADIUS | 가장 많이 사용되는 AAA 프로토콜 |
| TACACS+ | Cisco에서 많이 사용하는 AAA 프로토콜 |
| LDAP | 사용자 디렉토리 기반 인증 |

---

# 7. AAA 사용 목적

AAA를 사용하는 주요 목적:

- 네트워크 보안 강화
- 사용자 접근 제어
- 사용자 활동 기록
- 중앙 집중식 인증 관리

---

# 8. 정리

| 요소 | 역할 |
|------|------|
| Authentication | 사용자가 누구인지 확인 |
| Authorization | 사용자가 무엇을 할 수 있는지 결정 |
| Accounting | 사용자의 활동을 기록 |

즉,

> **AAA = 사용자 인증 + 권한 관리 + 활동 기록**
