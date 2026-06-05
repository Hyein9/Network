# EIGRP 라우팅 및 Key-Chain 인증 실습

## 개념 정리

### EIGRP (Enhanced Interior Gateway Routing Protocol)
- Cisco에서 개발한 거리 벡터 기반의 하이브리드 라우팅 프로토콜
- Autonomous System(AS) 번호가 같은 라우터끼리 이웃(Neighbor) 관계를 맺고, 라우팅 정보를 교환함
- DUAL 알고리즘을 사용해서 최적 경로를 계산하고, 토폴로지가 변경되면 빠르게 수렴함
- `no auto-summary`를 설정하지 않으면 클래스풀 단위로 자동 요약되기 때문에, 서브넷 단위로 광고하려면 반드시 꺼줘야 함

### Key-Chain 인증
- EIGRP 이웃 간에 인증을 적용하면, 허가되지 않은 라우터가 라우팅 테이블에 개입하는 걸 막을 수 있음
- key-chain 이름, key 번호, key-string이 양쪽 라우터에서 전부 일치해야 이웃 관계가 유지됨
- 인증을 적용하는 순간 기존 이웃 관계가 한 번 끊겼다가 재수립되는데, 이건 정상 동작임

### 서브넷 구성
- 이번 실습에서는 192.168.2.0/24를 /26 단위로 나눠서 사용
- /26이면 서브넷당 호스트 62개 사용 가능 (2^6 - 2)

| 서브넷 | 네트워크 주소 | 사용 가능 범위 |
|--------|-------------|---------------|
| 1번째 | 192.168.2.0/26 | .1 ~ .62 |
| 2번째 | 192.168.2.64/26 | .65 ~ .126 |
| 3번째 | 192.168.2.128/26 | .129 ~ .190 |

---

## 실습 환경

- Dynamips VM에서 Cisco 3725 라우터 2대 (R21, R22) 사용
- FastEthernet 인터페이스 사용
- 부팅 직후 인터페이스는 전부 shutdown 상태

---

## 1. IP 주소 설정 및 인터페이스 활성화

**R21**
```
conf t
int f0/0
 ip address 192.168.2.65 255.255.255.192
 no shutdown

int f0/1
 ip address 192.168.2.3 255.255.255.192
 no shutdown
```

**R22**
```
conf t
int f0/0
 ip address 192.168.2.66 255.255.255.192
 no shutdown

int f0/1
 ip address 192.168.2.130 255.255.255.192
 no shutdown
```

설정 후 `show ip int brief`로 인터페이스 상태 확인.

---

## 2. EIGRP 102 설정

AS 번호 102로 EIGRP를 구성하고, 자동 요약을 끈 뒤 각 서브넷을 광고한다.

**R21**
```
router eigrp 102
 no auto-summary
 network 192.168.2.0 0.0.0.63
 network 192.168.2.64 0.0.0.63
```

**R22**
```
router eigrp 102
 no auto-summary
 network 192.168.2.64 0.0.0.63
 network 192.168.2.128 0.0.0.63
```

R22에서 `DUAL-5-NBRCHANGE: Neighbor 192.168.2.65 is up` 로그가 뜨면 이웃 관계가 정상적으로 형성된 것이다. `show ip route`로 EIGRP로 학습된 경로가 들어왔는지 확인.

---

## 3. EIGRP 인증 (Key-Chain) 설정

양쪽 라우터에 동일한 key-chain을 만들고, 연결 인터페이스에 적용한다.

**R21**
```
conf t
key chain R21-KEY
 key 21
  key-string cloud21

int f0/0
 ip authentication key-chain eigrp 102 R21-KEY
```

**R22**
```
conf t
key chain R21-KEY
 key 21
  key-string cloud21

int f0/0
 ip authentication key-chain eigrp 102 R21-KEY
```

적용하는 순간 `Neighbor down: keychain changed` 이후 다시 `Neighbor up`이 뜬다. 인증 정보가 바뀌면서 이웃 관계를 재수립하는 과정이라 정상적인 동작이다.

---

## 4. 연결 테스트

```
R21# ping 192.168.2.66
R22# ping 192.168.2.3
```

첫 번째 ping에서 성공률이 80%로 나오는 경우가 있는데, 최초 ARP 요청 때 첫 패킷이 드롭되는 것이라 문제 없다. 두 번째부터 100% 나오면 정상.

---

## 네트워크 구조

```
[R21 LAN]            [R21] ---- [R22]            [R22 LAN]
192.168.2.0/26    f0/1  f0/0    f0/0  f0/1    192.168.2.128/26
  .3                .65    .66                .130
                  192.168.2.64/26
```

| 구간 | 네트워크 | R21 | R22 |
|------|---------|-----|-----|
| R21 - R22 연결 | 192.168.2.64/26 | .65 | .66 |
| R21 LAN | 192.168.2.0/26 | .3 | - |
| R22 LAN | 192.168.2.128/26 | - | .130 |

---

## 실습 중 발생한 오류

| 메시지 | 원인 | 해결 |
|--------|------|------|
| Translating "conft" | `conf t` 오타 | 띄어쓰기 확인 |
| key21 인식 안됨 | `key 21`로 띄어써야 함 | 문법 수정 |
| Neighbor Down | Key-chain 적용 시 재수립 과정 | 정상 동작 |
| ping 성공률 80% | ARP 초기 학습 | 정상 동작 |

---

## 정리

이번 실습에서는 두 대의 라우터에 IP를 설정하고 EIGRP로 라우팅 이웃 관계를 구성한 뒤, Key-Chain 인증까지 적용하는 과정을 진행했다. 인증을 적용했을 때 이웃 관계가 한 번 끊겼다 다시 올라오는 동작을 직접 확인할 수 있었고, 서브넷을 /26으로 나눠서 각 구간에 할당하는 구조도 이해할 수 있었다.
