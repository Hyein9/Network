## 실습
#### 1️⃣ R21, R22 라우터 기동

Dynamips VM에서 3725 라우터 부팅

기본 인터페이스 모두 shutdown 상태

FastEthernet 인터페이스 사용

2️⃣ IP 주소 설정 & 인터페이스 UP
✅ R21
```
conf t
int f0/0
 ip address 192.168.2.65 255.255.255.192
 no shutdown

int f0/1
 ip address 192.168.2.3 255.255.255.192
 no shutdown
```
확인:
```
(do) show ip int brief
```
✅ R22
```
conf t
int f0/0
 ip address 192.168.2.66 255.255.255.192
 no shutdown

int f0/1
 ip address 192.168.2.130 255.255.255.192
 no shutdown
```
확인:
```
do show ip int brief
```
#### 3️⃣ EIGRP 102 설정 (자동요약 끄기 + 네트워크 광고)
✅ R21
```
router eigrp 102
 no auto-summary
 network 192.168.2.0 0.0.0.63
 network 192.168.2.64 0.0.0.63

이웃 형성 확인:

do show ip route
```
✅ R22
```
router eigrp 102
 no auto-summary
 network 192.168.2.64 0.0.0.63
 network 192.168.2.128 0.0.0.63

이웃 형성 로그:

DUAL-5-NBRCHANGE: Neighbor 192.168.2.65 is up
```
#### 4️⃣ EIGRP 인증 (Key-Chain) 설정 🔐

포인트:

양쪽 라우터에서

같은 key-chain 이름

같은 key 번호

같은 key-string
→ 다 맞아야 이웃 유지됨

✅ R21
````
conf t
key chain R21-KEY
 key 21
  key-string cloud21

int f0/0
 ip authentication key-chain eigrp 102 R21-KEY
```
✅ R22
```
conf t
key chain R21-KEY
 key 21
  key-string cloud21

int f0/0
 ip authentication key-chain eigrp 102 R21-KEY
```
👉 적용 순간:
```
Neighbor down: keychain changed
Neighbor up: new adjacency
```
→ 정상 동작

5️⃣ 연결 테스트 (PING)
R21 → R22 쪽
do ping 192.168.2.2
R22 → 다른 네트워크
do ping 192.168.2.1

성공률 80% → 최초 ARP 때문 (정상)

🧠 네트워크 구조 요약
구간	네트워크	R21	R22
R21 ↔ R22	192.168.2.64/26	192.168.2.65	192.168.2.66
R21 LAN	192.168.2.0/26	192.168.2.3	–
R22 LAN	192.168.2.128/26	–	192.168.2.130
🚨 중간에 나온 에러 정리
메시지	이유	해결
Translating "conft"	오타	conf t
key21	문법 오류	key 21
Neighbor Down	Key-chain 적용 순간	정상 현상
ping 80%	ARP 초기 학습	정상
📌 실습 목적 정리

✔ IP 주소 설정
✔ EIGRP 인접 형성
✔ 서브넷 분리
✔ EIGRP 인증(Key-chain) 적용
✔ 인증 변경 시 인접 관계 재수립 확인

✨ 한 줄 요약

R21–R22 간 EIGRP 102를 설정하고,
Key-Chain 인증까지 적용해서
보안이 들어간 라우팅 인접 관계를 성공적으로 구성한 실습 🔐🚀
