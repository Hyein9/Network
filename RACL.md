# Reflexive ACL (RACL) 실습

## 개념 정리

### Reflexive ACL이란
- 내부에서 외부로 나가는 트래픽을 기준으로 세션 정보를 자동 생성하고, 외부에서 들어오는 트래픽 중 해당 세션의 응답만 허용하는 동적 ACL
- 외부에서 먼저 시작하는 접속은 기본적으로 차단됨
- 일반 ACL은 정적으로 허용/차단을 정하지만, Reflexive ACL은 세션 상태를 추적하여 동적으로 판단

### 일반 ACL vs Reflexive ACL

| 구분 | 일반 ACL (Extended) | Reflexive ACL |
|------|-------------------|---------------|
| 동작 방식 | 정적 규칙으로 필터링 | 세션 기반 동적 필터링 |
| 응답 트래픽 | 별도 permit 규칙 필요 | 자동으로 응답 허용 |
| 외부 시작 접속 | 별도 deny 규칙 필요 | 기본 차단 |
| 보안 수준 | 낮음 (양방향 열어야 함) | 높음 (내부 시작만 허용) |

### Reflexive ACL 핵심 키워드

| 키워드 | 역할 | 적용 방향 |
|--------|------|----------|
| `reflect` | 나가는 트래픽의 세션 정보를 기록 | Outbound ACL |
| `evaluate` | 들어오는 트래픽이 기록된 세션의 응답인지 검사 | Inbound ACL |

### 동작 원리

```
내부 → 외부 (Outbound):

  [R1] ---> [R2] ---(s0/1 out: acl-out)---> [R3] ---> [R4]
                     |
                     | reflect myracl
                     | → 세션 정보 자동 생성
                     |   (출발지, 목적지, 프로토콜, 포트 기록)


외부 → 내부 (Inbound, 응답):

  [R4] ---> [R3] ---(s0/1 in: acl-in)---> [R2] ---> [R1]
                     |
                     | evaluate myracl
                     | → 기록된 세션의 응답인지 확인
                     | → 일치하면 허용, 불일치하면 차단


외부 → 내부 (외부에서 먼저 시작):

  [R4] ---> [R3] ---(s0/1 in: acl-in)---> [R2]  X  차단
                     |
                     | evaluate myracl
                     | → 매칭되는 세션 없음 → 차단
```

### Named Extended ACL
- Reflexive ACL은 반드시 Named Extended ACL에서만 사용 가능
- 번호 기반 ACL(access-list 100 ...)에서는 reflect/evaluate 사용 불가

---

## 실습 토폴로지

```
  [R1]            [R2]              [R3]            [R4]
  s0/0            s0/0  s0/1       s0/0  s0/1      s0/0
    |               |     |          |     |          |
    +--1.1.12.0/24--+     +--1.1.23.0/24--+     +--1.1.34.0/24--+
         .1    .2              .2    .3              .3    .4

                        RACL 적용 위치
                        R2의 s0/1
                        out: acl-out (reflect)
                        in:  acl-in  (evaluate)
```

### 상세 구조

```
    내부 영역                    경계                     외부 영역
  (Inside)                   (R2 s0/1)                  (Outside)

  [R1] ---- [R2] -------- | acl-out (out) | -------- [R3] ---- [R4]
 1.1.12.1   1.1.12.2      | acl-in  (in)  |        1.1.34.3   1.1.34.4
                 1.1.23.2  |               | 1.1.23.3
                           +───────────────+

  R1 → R4 ping    : acl-out에서 reflect로 세션 생성 → R4 응답 허용
  R4 → R1 telnet  : acl-in에서 evaluate 검사 → 세션 없음 → 차단
```

### IP 주소 구성

| 라우터 | 인터페이스 | IP 주소 |
|--------|-----------|---------|
| R1 | s0/0 | 1.1.12.1/24 |
| R2 | s0/0 | 1.1.12.2/24 |
| R2 | s0/1 | 1.1.23.2/24 |
| R3 | s0/0 | 1.1.23.3/24 |
| R3 | s0/1 | 1.1.34.3/24 |
| R4 | s0/0 | 1.1.34.4/24 |

---

## 1단계. IP 설정

**R1**
```
conf t
interface s0/0
 ip address 1.1.12.1 255.255.255.0
 no shutdown
end
```

**R2**
```
conf t
interface s0/0
 ip address 1.1.12.2 255.255.255.0
 no shutdown

interface s0/1
 ip address 1.1.23.2 255.255.255.0
 no shutdown
end
```

**R3**
```
conf t
interface s0/0
 ip address 1.1.23.3 255.255.255.0
 no shutdown

interface s0/1
 ip address 1.1.34.3 255.255.255.0
 no shutdown
end
```

**R4**
```
conf t
interface s0/0
 ip address 1.1.34.4 255.255.255.0
 no shutdown
end
```

---

## 2단계. OSPF 설정

모든 라우터 간 통신이 가능하도록 OSPF를 구성한다.

**R1**
```
router ospf 1
 network 1.1.12.0 0.0.0.255 area 0
```

**R2**
```
router ospf 1
 network 1.1.12.0 0.0.0.255 area 0
 network 1.1.23.0 0.0.0.255 area 0
```

**R3**
```
router ospf 1
 network 1.1.23.0 0.0.0.255 area 0
 network 1.1.34.0 0.0.0.255 area 0
```

**R4**
```
router ospf 1
 network 1.1.34.0 0.0.0.255 area 0
```

OSPF 이웃 확인:
```
show ip ospf neighbor
show ip route
```

모든 라우터에서 전체 네트워크 경로가 보여야 다음 단계로 진행.

---

## 3단계. Telnet 설정

테스트를 위해 R1과 R4에 Telnet 접속을 허용한다.

**R1, R4 공통**
```
conf t
line vty 0 4
 password cisco
 login
 transport input all
end
```

RACL 적용 전 테스트:
```
R1# telnet 1.1.34.4     → 성공
R4# telnet 1.1.12.1     → 성공
```

양방향 모두 통신 가능한 상태를 먼저 확인.

---

## 4단계. Reflexive ACL 설정 (R2)

### Outbound ACL (나가는 트래픽에서 세션 생성)

```
conf t
ip access-list extended acl-out
 permit tcp any any reflect myracl
 permit udp any any reflect myracl
 permit icmp any any reflect myracl
 permit ip any any
exit
```

- `reflect myracl`: 나가는 TCP/UDP/ICMP 트래픽의 세션 정보를 myracl이라는 이름으로 기록
- `permit ip any any`: 나머지 IP 트래픽도 허용 (나가는 방향은 제한 없음)

### Inbound ACL (들어오는 트래픽에서 세션 검사)

```
ip access-list extended acl-in
 permit ospf host 1.1.23.3 any
 evaluate myracl
exit
```

- `permit ospf`: R3과의 OSPF 이웃 관계를 유지하기 위해 OSPF 트래픽 허용. 이 규칙이 없으면 OSPF가 끊어져 라우팅이 깨짐
- `evaluate myracl`: myracl에 기록된 세션의 응답 트래픽만 허용. 나머지는 암묵적 deny로 차단

### 인터페이스에 적용

```
interface s0/1
 ip access-group acl-out out
 ip access-group acl-in in
end
```

---

## 5단계. 테스트

### 내부 → 외부 (허용)

```
R1# ping 1.1.34.4
```
성공. acl-out에서 reflect로 ICMP 세션이 생성되고, R4의 응답이 acl-in의 evaluate를 통과.

```
R1# telnet 1.1.34.4
```
성공. TCP 세션이 reflect로 기록되어 응답 트래픽 허용.

### 외부 → 내부 (차단)

```
R4# telnet 1.1.12.1
```
차단. R4에서 시작한 접속이므로 myracl에 매칭되는 세션이 없어 acl-in에서 차단됨.

```
R4# ping 1.1.12.1
```
차단. 외부에서 먼저 시작한 ICMP이므로 세션 없음.

### 테스트 결과 요약

| 방향 | 테스트 | 결과 | 이유 |
|------|--------|------|------|
| R1 → R4 ping | 내부 시작 | 허용 | reflect로 세션 생성 → 응답 허용 |
| R1 → R4 telnet | 내부 시작 | 허용 | reflect로 세션 생성 → 응답 허용 |
| R4 → R1 telnet | 외부 시작 | 차단 | 매칭 세션 없음 → evaluate 불통과 |
| R4 → R1 ping | 외부 시작 | 차단 | 매칭 세션 없음 → evaluate 불통과 |

---

## 6단계. 외부 접속 선택적 허용 (옵션)

특정 외부 호스트의 Telnet 접속만 허용하려면 acl-in에 명시적 permit을 추가한다.

```
conf t
ip access-list extended acl-in
 permit tcp host 1.1.34.4 host 1.1.12.1 eq telnet
end
```

이 규칙 추가 후:
```
R4# telnet 1.1.12.1     → 허용 (명시적 permit)
R4# ping 1.1.12.1       → 여전히 차단 (ICMP는 허용 안 함)
```

---

## Reflexive ACL 세션 확인

```
show ip access-lists
```

내부에서 외부로 통신한 뒤 확인하면 myracl에 동적으로 생성된 세션 항목이 보인다:

```
Reflexive IP access list myracl
  permit icmp host 1.1.34.4 host 1.1.12.1 (5 matches) (time left 295 sec)
  permit tcp host 1.1.34.4 eq telnet host 1.1.12.1 eq 49160 (12 matches) (time left 295 sec)
```

세션은 기본 300초(5분) 후 자동 만료. 타임아웃 변경:
```
ip reflexive-list timeout 120
```

---

## 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| RACL 적용 후 OSPF 끊어짐 | acl-in에서 OSPF 허용 안 함 | `permit ospf host [R3 IP] any` 추가 |
| 내부 → 외부도 차단됨 | acl-out에 reflect 없거나 적용 방향 잘못 | out 방향에 acl-out 확인 |
| 세션이 생성 안 됨 | Named ACL이 아닌 번호 ACL 사용 | Named Extended ACL로 변경 |
| 외부 응답이 차단됨 | evaluate 이름 불일치 | reflect와 evaluate의 이름(myracl) 동일한지 확인 |

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show ip access-lists` | ACL 목록 + Reflexive 세션 항목 확인 |
| `show ip interface s0/1` | 인터페이스에 적용된 ACL 확인 |
| `show ip ospf neighbor` | OSPF 이웃 상태 (RACL로 끊어지지 않았는지) |
| `show ip route` | 라우팅 테이블 |

---

## 정리

- Reflexive ACL은 내부에서 시작한 통신의 응답만 허용하고, 외부에서 먼저 시작하는 접속은 차단하는 상태 기반 필터링
- reflect 키워드로 세션을 기록하고, evaluate 키워드로 응답 여부를 검사
- Named Extended ACL에서만 사용 가능하며, reflect와 evaluate의 이름이 반드시 동일해야 함
- OSPF 등 라우팅 프로토콜 트래픽은 별도로 permit 해야 이웃 관계가 유지됨
- 기본적으로 세션은 300초 후 만료되며, `ip reflexive-list timeout`으로 변경 가능
