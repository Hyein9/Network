# NAT (Network Address Translation) 실습

## 개념 정리

### NAT란
- 사설 IP 주소를 공인 IP 주소로 변환하는 네트워크 주소 변환 기술
- 내부망에서는 공인 IP의 가용 범위를 줄이기 위해 사설 IP를 사용하고, 외부망(인터넷)에 나갈 때 공인 IP로 변환함
- 대표적인 NAT 장치가 IP 공유기

### 사설 IP 대역

| 클래스 | 대역 | 범위 |
|--------|------|------|
| A | 10.0.0.0/8 | 10.0.0.0 ~ 10.255.255.255 |
| B | 172.16.0.0/12 | 172.16.0.0 ~ 172.31.255.255 |
| C | 192.168.0.0/16 | 192.168.0.0 ~ 192.168.255.255 |

사설 IP는 인터넷에서 라우팅되지 않으므로 외부와 통신하려면 반드시 NAT가 필요하다.

### NAT의 특징
1. **공인 IP 절약**: 여러 내부 호스트가 소수의 공인 IP를 공유하여 사용할 수 있음
2. **보안 효과**: 외부에서는 내부의 사설 IP 대역을 알 수 없어 해킹 대상을 식별하기 어려움
3. **유연한 주소 관리**: 내부 네트워크 구조를 변경해도 외부에 영향을 주지 않음

### NAT 용어

| 용어 | 설명 |
|------|------|
| Inside Local | 내부 호스트의 실제 사설 IP |
| Inside Global | 내부 호스트가 외부에 보여지는 공인 IP (NAT 변환 후) |
| Outside Local | 외부 호스트가 내부에서 보이는 IP |
| Outside Global | 외부 호스트의 실제 공인 IP |

### NAT 종류

| 종류 | 설명 |
|------|------|
| Static NAT | 사설 IP 1개와 공인 IP 1개를 1:1로 고정 매핑 |
| Dynamic NAT | 사설 IP를 공인 IP Pool에서 동적으로 할당. Pool이 소진되면 대기 |
| PAT (Port Address Translation) | 하나의 공인 IP를 포트 번호로 구분하여 여러 호스트가 동시에 사용. Overload라고도 부름 |

### NAT 동작 흐름

```
내부 → 외부 (Outbound)

  [PC1]              [NAT Router]                [Internet]
  172.16.10.10       Inside    Outside           8.8.8.8
       |                |         |                  |
       |  src: 172.16.10.10       |                  |
       +----------->    |         |                  |
                   NAT 변환       |                  |
                   src → 10.100.105.241              |
                        |         +----------------->|
                        |    src: 10.100.105.241     |
                        |                            |

외부 → 내부 (Inbound, 응답)

                        |    dst: 10.100.105.241     |
                        |<---------------------------+
                   NAT 역변환                        |
                   dst → 172.16.10.10                |
       |<-----------    |                            |
       |  dst: 172.16.10.10                          |
```

---

## 실습 토폴로지

```
     [LAN1]                      [LAN2]
  172.16.100.0/24              172.16.10.0/24
       |                            |
     f0/0                         f0/1      f0/0                f0/1
  +--+---+                     +---+--------+---+          +----+----+
  |      |     172.16.10.0/24  |                |  10.100  |         |
  |  R1  +---------------------+      R2        +--105.0---+   ISP   |
  |      | .2                .1|   (NAT Router)  | .240  .1|  (GW)   |
  +------+                     +--------+--------+         +---------+
                                        |
                                   NAT 변환
                                Inside: f0/0
                                Outside: f0/1

  +------+    +------+
  | SW1  |    | SW2  |    ← L2 스위치 (별도 설정 불필요)
  +------+    +------+
```

### 상세 연결도

```
  [PC1]     [PC2]              [PC3]     [PC4]                     [ISP]
 .100.10   .100.11             .10.10    .10.11                   .105.1
    |         |                   |         |                        |
    +----+----+                   +----+----+                        |
         |                             |                             |
       [SW1]                         [SW2]                           |
         |                             |                             |
       f0/0                          f0/0                          f0/1
      [R1]                          [R2]                            |
  172.16.100.1                  172.16.10.1                  10.100.105.1
         |                             |                             |
       f0/1                       NAT Inside                    NAT Outside
  172.16.10.2                          |                             |
         |                         f0/0 (.1)                   f0/1 (.240)
         +-------- 172.16.10.0/24 -----+                             |
                                       +----- 10.100.105.0/24 ------+
```

### IP 주소 구성

**라우터**

| 라우터 | 인터페이스 | IP 주소 | NAT 역할 |
|--------|-----------|---------|----------|
| R1 | f0/0 | 172.16.100.1/24 | - |
| R1 | f0/1 | 172.16.10.2/24 | - |
| R2 | f0/0 | 172.16.10.1/24 | Inside |
| R2 | f0/1 | 10.100.105.240/24 | Outside |

**PC**

| PC | IP | Gateway |
|----|----|---------|
| PC1 | 172.16.100.10/24 | 172.16.100.1 |
| PC2 | 172.16.100.11/24 | 172.16.100.1 |
| PC3 | 172.16.10.10/24 | 172.16.10.1 |
| PC4 | 172.16.10.11/24 | 172.16.10.1 |

**NAT Pool**

| 항목 | 값 |
|------|-----|
| Pool 이름 | NAT_POOL |
| 공인 IP 범위 | 10.100.105.241 ~ 10.100.105.243 |
| Netmask | 255.255.255.0 |
| ISP Gateway | 10.100.105.1 |

---

## 1단계. R1 설정 (내부 라우터)

```
enable
conf t
hostname R1

interface f0/0
 ip address 172.16.100.1 255.255.255.0
 no shutdown

interface f0/1
 ip address 172.16.10.2 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 172.16.10.1
end
write
```

R1은 NAT 설정 없이 내부 네트워크 간 라우팅만 수행한다. 기본 경로를 R2(172.16.10.1)로 설정하여 외부로 나가는 트래픽을 R2에 전달.

---

## 2단계. R2 설정 (NAT Router)

### 인터페이스 설정

```
enable
conf t
hostname R2

interface f0/0
 ip address 172.16.10.1 255.255.255.0
 ip nat inside
 no shutdown

interface f0/1
 ip address 10.100.105.240 255.255.255.0
 ip nat outside
 no shutdown
```

- `ip nat inside`: 이 인터페이스를 NAT 내부로 지정
- `ip nat outside`: 이 인터페이스를 NAT 외부로 지정

### ACL 설정 (NAT 대상 네트워크 지정)

```
access-list 1 permit 172.16.10.0 0.0.0.255
access-list 1 permit 172.16.100.0 0.0.0.255
```

NAT 변환 대상이 되는 내부 네트워크를 ACL로 정의한다. 두 개의 내부 대역 모두 포함.

### NAT Pool 생성

```
ip nat pool NAT_POOL 10.100.105.241 10.100.105.243 netmask 255.255.255.0
```

공인 IP 3개(241, 242, 243)를 풀로 묶는다. 내부 호스트가 외부로 나갈 때 이 풀에서 IP를 할당받음.

### Dynamic NAT 적용

```
ip nat inside source list 1 pool NAT_POOL
```

ACL 1에 매칭되는 출발지를 NAT_POOL의 공인 IP로 변환한다.

### 기본 경로 설정

```
ip route 0.0.0.0 0.0.0.0 10.100.105.1
end
write
```

---

## 3단계. PC IP 설정

```
PC1: ip 172.16.100.10 255.255.255.0 gateway 172.16.100.1
PC2: ip 172.16.100.11 255.255.255.0 gateway 172.16.100.1
PC3: ip 172.16.10.10  255.255.255.0 gateway 172.16.10.1
PC4: ip 172.16.10.11  255.255.255.0 gateway 172.16.10.1
```

SW1, SW2는 기본 L2 스위치이므로 별도 설정 불필요.

---

## 4단계. 동작 확인

### 라우팅 테이블 확인

```
show ip route
```

### NAT 변환 테이블 확인

```
show ip nat translations
```

PC에서 외부로 ping을 보낸 뒤 이 명령을 실행하면 변환된 IP를 확인할 수 있다:

```
Pro  Inside global      Inside local       Outside local      Outside global
---  10.100.105.241     172.16.10.10       ---                ---
---  10.100.105.242     172.16.100.10      ---                ---
```

### NAT 통계 확인

```
show ip nat statistics
```

변환 횟수, Pool 사용량 등을 확인 가능.

### 통신 테스트

```
PC1> ping 10.100.105.1
PC3> ping 10.100.105.1
```

---

## NAT 변환 흐름 상세

```
PC3 (172.16.10.10)이 외부(10.100.105.1)로 ping:

  [PC3]
  src: 172.16.10.10
  dst: 10.100.105.1
     |
     v
  [R2 f0/0] (ip nat inside)
     |
     | NAT 변환 테이블 확인
     | ACL 1 매칭: 172.16.10.0/24 → permit
     | NAT_POOL에서 공인 IP 할당: 10.100.105.241
     |
     | src: 172.16.10.10 → 10.100.105.241 (변환)
     v
  [R2 f0/1] (ip nat outside)
  src: 10.100.105.241
  dst: 10.100.105.1
     |
     v
  [ISP] 수신
     |
     | 응답: dst → 10.100.105.241
     v
  [R2 f0/1] (ip nat outside)
     |
     | NAT 역변환
     | dst: 10.100.105.241 → 172.16.10.10
     v
  [R2 f0/0] (ip nat inside)
     |
     v
  [PC3] 수신
```

---

## Dynamic NAT의 한계와 PAT

### Dynamic NAT의 문제
- Pool에 공인 IP가 3개뿐이므로 동시에 4번째 호스트가 외부 접속을 시도하면 NAT 변환 실패
- Pool이 소진되면 대기해야 함

### PAT (Port Address Translation) - Overload

하나의 공인 IP를 포트 번호로 구분하여 여러 내부 호스트가 동시에 사용하는 방식. 가정용 공유기가 사용하는 방식이 바로 PAT.

**Pool 기반 PAT:**
```
ip nat inside source list 1 pool NAT_POOL overload
```

**인터페이스 기반 PAT (가장 일반적):**
```
ip nat inside source list 1 interface f0/1 overload
```

외부 인터페이스의 IP 하나로 모든 내부 호스트가 포트 번호를 달리하여 동시에 외부 접속 가능.

### Dynamic NAT vs PAT 비교

| 구분 | Dynamic NAT | PAT (Overload) |
|------|------------|----------------|
| 변환 방식 | IP 1:1 매핑 | IP + 포트 번호로 N:1 매핑 |
| 동시 접속 | Pool 개수만큼 | 이론상 65,535개 (포트 수) |
| 공인 IP 사용 | 여러 개 필요 | 1개로 충분 |
| 사용 사례 | 서버 등 고정 매핑 필요 시 | 일반적인 인터넷 공유 |

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show ip nat translations` | 현재 NAT 변환 테이블 확인 |
| `show ip nat statistics` | NAT 통계 (변환 횟수, Pool 사용량) |
| `show ip route` | 라우팅 테이블 확인 |
| `show access-lists` | ACL 설정 및 매칭 확인 |
| `clear ip nat translation *` | NAT 변환 테이블 초기화 |
| `debug ip nat` | NAT 변환 과정 실시간 확인 (디버깅) |

---

## 설정 명령어 정리

```
-- 인터페이스에 NAT Inside/Outside 지정
interface [인터페이스]
 ip nat inside
 ip nat outside

-- NAT 대상 ACL 설정
access-list [번호] permit [네트워크] [와일드카드]

-- NAT Pool 생성
ip nat pool [이름] [시작IP] [끝IP] netmask [마스크]

-- Dynamic NAT 적용
ip nat inside source list [ACL번호] pool [Pool이름]

-- PAT (Pool 기반 Overload)
ip nat inside source list [ACL번호] pool [Pool이름] overload

-- PAT (인터페이스 기반 Overload)
ip nat inside source list [ACL번호] interface [인터페이스] overload

-- Static NAT (1:1 고정)
ip nat inside source static [사설IP] [공인IP]
```

---

## 정리

- NAT는 사설 IP를 공인 IP로 변환하여 내부 호스트가 인터넷과 통신할 수 있게 하는 기술
- 인터페이스에 ip nat inside/outside를 지정하고, ACL로 변환 대상을 정의한 뒤 Pool 또는 인터페이스에 매핑
- Dynamic NAT는 Pool의 공인 IP 개수만큼만 동시 접속 가능하므로, 실무에서는 PAT(Overload)를 주로 사용
- PAT는 하나의 공인 IP로 포트 번호를 구분하여 다수의 호스트가 동시에 외부 접속 가능
- `show ip nat translations`로 변환 상태를 확인하는 것이 기본 검증 방법
