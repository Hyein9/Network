# VLAN 개념
VLAN = 한 대의 스위치를 ‘논리적으로 여러 대처럼’ 나누는 기술
👉 물리적으로는 스위치 1대인데
👉 논리적으로는 여러 개의 네트워크로 쪼개는 거임

🧠 왜 VLAN을 쓰냐?
❌ VLAN 없을 때

스위치에 꽂힌 모든 PC가 다 같은 네트워크

브로드캐스트 다 받음 → 느려짐

보안 구림 (부서 구분 안 됨)

⭕ VLAN 있을 때

같은 스위치라도 서로 다른 VLAN이면 완전 다른 네트워크

브로드캐스트 분리됨

보안 좋아짐

네트워크 구조 깔끔해짐

🏢 현실 비유로 이해해보자

스위치 = 건물
포트 = 방
VLAN = 부서

포트	VLAN
Fa0/1	영업부 (VLAN 10)
Fa0/2	개발부 (VLAN 20)
Fa0/3	영업부 (VLAN 10)
Fa0/4	개발부 (VLAN 20)

👉 같은 건물에 있어도
👉 부서 다르면 서로 말 안 통함

🧱 VLAN 핵심 개념 4개만 기억해라
#### 1️⃣ VLAN은 브로드캐스트 도메인 분리

VLAN 10에서 ARP 날리면
👉 VLAN 20은 못 받음

#### 2️⃣ VLAN끼리는 원래 통신 안 됨

👉 VLAN 10 ↔ VLAN 20
❌ 직접 통신 불가

✔ 통신하려면?
➡ 라우터 또는 L3 스위치 필요 (Inter-VLAN Routing)

#### 3️⃣ Access 포트 vs Trunk 포트
✅ Access 포트

PC 같은 일반 장비 연결

VLAN 하나만 소속
```
switchport mode access
switchport access vlan 10
```
✅ Trunk 포트

스위치 ↔ 스위치

스위치 ↔ 라우터

여러 VLAN 태그 붙여서 전달

``` switchport mode trunk ```

4️⃣ VLAN은 번호 + 이름
vlan 10
 name Sales


번호 = VLAN ID

이름 = 그냥 보기 좋게 붙이는 거

🧪 너가 한 실습 = VLAN 구조 예시
PC1 -- Fa0/1 -- VLAN 10 (192.168.10.0/24)
PC2 -- Fa0/2 -- VLAN 20 (192.168.20.0/24)
PC3 -- Fa0/3 -- VLAN 10
PC4 -- Fa0/4 -- VLAN 20


👉 PC1 ↔ PC3 통신 가능
👉 PC2 ↔ PC4 통신 가능
👉 PC1 ↔ PC2 ❌ (라우터 없으면 안 됨)

🧠 시험에 자주 나오는 핵심 문장

VLAN은 논리적 네트워크 분리 기술

VLAN은 브로드캐스트 도메인을 분리

서로 다른 VLAN 간 통신은 라우터 필요

Access 포트는 VLAN 1개

Trunk 포트는 VLAN 여러 개 전달

⚠️ 실습할 때 자주 터지는 실수

❌ VLAN 만들고 포트 할당 안 함
❌ Access/Trunk 헷갈림
❌ VLAN 간 통신 되길 기대함 (라우터 없는데)



##### 1️⃣ VLAN 초기화
```
Delete filename [vlan.dat]?
Delete flash:/vlan.dat? [confirm]
Switch# reload
```

👉 이거 = 스위치에 저장된 VLAN 정보 초기화
vlan.dat 삭제 + 재부팅 →
기존에 만들었던 VLAN 다 날아가고 VLAN 1만 남음

그래서:

“전에 하던 거 있는데 그게 안 보이네”
👉 정상임. VLAN 정보 초기화해서 사라진 거임.

##### 2️⃣ 재부팅 후 VLAN 상태 확인
Switch# sh vlan


결과:

VLAN 1만 존재

모든 포트가 VLAN 1에 들어가 있음

1002~1005는 Cisco 기본 VLAN (삭제 불가)

##### 3️⃣ 스위치 이름 바꿈
```
conf t
hostname SW1
```
##### 4️⃣ VLAN 생성
```
vlan 10
 name 192.168.10.0/24-Network

vlan 20
 name 192.168.20.0/240Network
```

👉 VLAN 10, VLAN 20 생성 완료
※ 이름 오타 있음:

192.168.20.0/240Network  ❌
192.168.20.0/24-Network  ⭕

5️⃣ 포트에 VLAN 할당
```
int range f0/1 , f0/3
 switchport mode access
 switchport access vlan 10
```

👉 Fa0/1, Fa0/3 → VLAN 10
```
int range f0/2, f0/4
 switchport mode access
 switchport access vlan 20
```

👉 Fa0/2, Fa0/4 → VLAN 20

6️⃣ 최종 결과
``` sh vlan ```


결과:

VLAN	포트
10	Fa0/1, Fa0/3
20	Fa0/2, Fa0/4
1	나머지 전부
📌 시험/과제용 “VLAN 설정 명령어 한 방 정리”

이거 그대로 외우면 됨 👇
```
en
conf t

hostname SW1

vlan 10
 name VLAN10
exit

vlan 20
 name VLAN20
exit

int range f0/1, f0/3
 switchport mode access
 switchport access vlan 10
exit

int range f0/2, f0/4
 switchport mode access
 switchport access vlan 20
exit

end
show vlan
```
💡 실습할 때 꿀팁

✔ 저장 안 하면 전원 끄면 다 날아감:

copy run start


✔ 특정 포트 VLAN 확인:

show run interface f0/1


✔ 포트 상태 보기:

show interfaces status





f0/1번엔 태그 없음. vlan 10번으로 붙여서 도착하면 뗌


🔁 전체 흐름 한 줄 요약

VLAN 생성 → 트렁크 구성 → Native VLAN 삽질 → 스패닝트리 에러 → 라우터 서브인터페이스 구성 → VLAN 간 통신 성공

1️⃣ 트렁크 & VLAN allowed 조작 (SW1)
✔ 한 일
```
int f0/10
 switchport trunk allowed vlan remove 20
 switchport trunk allowed vlan add 20
```
✔ 결과
``` show interfaces trunk ```


VLAN 20 제거 → 다시 추가됨

트렁크 정상

👉 이 부분 정상 작업 + 이해 잘함

2️⃣ Native VLAN mismatch 에러 (SW1 ↔ SW2)
❌ 에러 로그
%CDP-4-NATIVE_VLAN_MISMATCH
%SPANTREE-2-BLOCK_PVID_LOCAL

❌ 원인

SW1: Native VLAN = 10

SW2: Native VLAN = 1
👉 서로 달라서 STP가 포트 차단함

✅ 해결

양쪽 동일하게 맞춤 (VLAN 1로 복구한 거 잘함)
```
int f0/10
 switchport trunk native vlan 1
```

👉 트렁크 양쪽 Native VLAN은 반드시 동일

3️⃣ 스위치에 IP 못 넣는 이유
❌ 시도
```
int f0/20
 ip address 192.168.10.1 255.255.255.0
```
❌ 에러 이유

👉 L2 스위치(2950)는 물리 포트에 IP 못 넣음

✅ 맞는 방법 (SVI)
```
int vlan 10
 ip address 192.168.10.254 255.255.255.0
 no shutdown
```

👉 관리용 IP는 VLAN 인터페이스(SVI) 에만 설정 가능

4️⃣ 라우터 서브인터페이스 (Router-on-a-Stick)
❌ 처음 에러
ip address ...
% Configuring IP routing ... only allowed if encapsulation set

❌ 원인

👉 encapsulation 먼저 안 함

✅ 정석 순서
```
int f0/0.1
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

int f0/0.2
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```

👉 이거 순서 진짜 시험에도 나옴

5️⃣ VLAN 간 통신 성공
✔ 라우터 라우팅 테이블
```
show ip route

C 192.168.10.0/24 → Fa0/0.1
C 192.168.20.0/24 → Fa0/0.2
```
✔ PC에서 핑 성공

VLAN 10 ↔ VLAN 10 OK

VLAN 10 → VLAN 20 OK

중간에 ARP 때문에 첫 핑 타임아웃 → 정상 현상

👉 구성 전부 맞음 👍

🧠 지금 네 토폴로지 구조
```
PC (VLAN10) ┐
            ├─ SW1 ──(trunk)── SW2
PC (VLAN20) ┘          │
                       │ trunk
                       R1
                    (Router-on-a-Stick)
```

✅ 시험/실습용 명령어 치트키 (이거만 외워도 됨)
🔹 VLAN 생성
```
conf t
vlan 10
 name VLAN10
vlan 20
 name VLAN20
end
```
🔹 Access 포트
```
int f0/1
 switchport mode access
 switchport access vlan 10
```
🔹 Trunk 포트
```
int f0/10
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport trunk native vlan 1
```
🔹 Trunk 확인
```
show interfaces trunk
```
🔹 스위치 관리 IP (SVI)
```
int vlan 10
 ip address 192.168.10.254 255.255.255.0
 no shutdown
```
🔹 Router-on-a-Stick
```
int f0/0
 no shutdown

int f0/0.1
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

int f0/0.2
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```
🔹 저장
```copy run start```

⚠️ 네가 겪은 에러 = 실습자 90%가 겪는 실수
에러	원인
Native VLAN mismatch	트렁크 양쪽 Native VLAN 다름
Spanning-tree block	VLAN ID 불일치
IP address invalid	L2 스위치 포트에 IP 넣음
encapsulation 에러	dot1Q 먼저 안 함

👉 오히려 제대로 다 밟고 온 거라 실력 쌓임 😆




🧠 멀티레이어 스위치 핵심 개념

L2 스위치: VLAN 나누기만 가능 (라우팅 ❌)

멀티레이어 스위치(L3):
👉 VLAN 나누기 + VLAN 간 라우팅 ⭕

Router-on-a-Stick 필요 없음

성능 훨씬 좋음 (하드웨어 라우팅)

🧱 토폴로지 예시
PC1 (VLAN10) ─┐
               ├─ MLS (3560 같은 L3 스위치)
PC2 (VLAN20) ─┘


게이트웨이:

VLAN10 → 192.168.10.1

VLAN20 → 192.168.20.1
(이 IP를 멀티레이어 스위치가 직접 가짐)

✅ 멀티레이어 스위치 기본 세팅 (치트키)

⚠️ 3560 / 3650 / 3750 같은 L3 스위치 써야 함
2950, 2960 ❌ (L2 전용)

1️⃣ VLAN 생성
```
conf t
vlan 10
 name VLAN10
vlan 20
 name VLAN20
````
2️⃣ SVI에 IP 할당 (게이트웨이 역할)
```
int vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

int vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
```
3️⃣ ⭐ IP 라우팅 켜기 (이게 핵심)
ip routing
👉 이거 안 하면 L3 스위치가 라우팅 안 함 (제일 많이 까먹는 포인트)

4️⃣ PC 연결 포트 설정 (Access)
```
int f0/1
 switchport mode access
 switchport access vlan 10

int f0/2
 switchport mode access
 switchport access vlan 20
```
5️⃣ 확인
```show ip route```

정상이면:
```
C 192.168.10.0/24 is directly connected, Vlan10
C 192.168.20.0/24 is directly connected, Vlan20

show vlan
```
🖥 PC IP 설정
```
PC	IP	Gateway
PC1	192.168.10.10	192.168.10.1
PC2	192.168.20.10	192.168.20.1
```
🎯 테스트
``` PC1 > ping 192.168.20.10 ```


👉 바로 통신 됨 (라우터 없이 성공 🔥)

#### ⚠️ 멀티레이어 스위치에서 자주 터지는 실수
증상	원인
VLAN 간 통신 안 됨	ip routing 안 켬
SVI down	해당 VLAN에 포트 없음 or no shutdown 안 함
핑 안 됨	PC 게이트웨이 설정 안 함
IP 안 먹힘	L2 스위치로 실습함
🧠 Router-on-a-Stick vs L3 Switch 차이
구분	Router-on-a-Stick	멀티레이어 스위치
성능	느림 (단일 링크)	빠름 (하드웨어 라우팅)
구성	복잡	깔끔
장비	라우터 필요	스위치 하나로 끝
실무	소규모	중대형 네트워크
