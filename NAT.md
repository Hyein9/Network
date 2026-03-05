# NAT(Network Address Translation)
-사설 ip address를 public ip address로 변환시키는 통신망의 주소 변환기술
- 내부망은 public ip의 가용범위를 줄이기 위해 사설 ip를 사용
- 인터넷망(외부망)은 사실관계에 따른 ip실명을 원칙 : 국제표준에 제시된 국가별 공인 ip를 사용하도록 InterNIC에 의해 규정
- 외부망인 인터넷 망에서는 공인 ip를 사용

### NAT특징
1) 인터넷의 공인 ip주소를 절약
2) 공공망과 연결되는 사용자들의 고유한 사설망을 해킹으로부터 일정하게 보호 (외부에서는 사설 ip의 존재 및 주소대역을 식별하기가 어렵다.)
3) 인터넷의 공인IP주소 클래스는 한정되어있기 때문에 가급적 이를 공유할 수 있도록하기 위해서 등장한 기능이 바로 NAT이고 대표적인 장치가 IP공유기를 들 수 있다.
4) NAT를 사용하면 사설IP주소대역을 사용하면서 이를 공인 IP주소를 상호변환할 수 있도록 하여 공인 IP주소를 다수와 함께 동시에 사용할 수 있도록 함으로써 IP낭비를 현저히 줄 수 있다.

🧠 주소 정리 (먼저 이거 보고 가자)
내부망
```
LAN1: 172.16.100.0/24
PC-1, PC-2

LAN2: 172.16.10.0/24
PC-3, PC-4
```
```
NAT Router (R2)
Inside → 172.16.10.0/24
Outside → 10.100.105.0/24

ISP GW → 10.100.105.1

NAT Pool

10.100.105.241 ~ 243
```
🖥️ R1 설정 (내부 라우터)
```
enable
conf t
hostname R1

interface fa0/0
 ip address 172.16.100.1 255.255.255.0
 no shutdown

interface fa0/1
 ip address 172.16.10.2 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 172.16.10.1
end
wr
```
🌐 R2 설정 (NAT 핵심 ⭐)
```
enable
conf t
hostname R2

인터페이스 설정
interface fa0/0
 ip address 172.16.10.1 255.255.255.0
 ip nat inside
 no shutdown

interface fa0/1
 ip address 10.100.105.240 255.255.255.0
 ip nat outside
 no shutdown
```

#### ACL 설정 (내부망 지정)
```
access-list 1 permit 172.16.10.0 0.0.0.255
access-list 1 permit 172.16.100.0 0.0.0.255
```
#### NAT Pool 생성
```ip nat pool NAT_POOL 10.100.105.241 10.100.105.243 netmask 255.255.255.0```

#### Dynamic NAT 적용
```ip nat inside source list 1 pool NAT_POOL```

기본 경로
```
ip route 0.0.0.0 0.0.0.0 10.100.105.1
end
wr
```
🖧 SW1 / SW2

👉 설정 필요 없음
(기본 L2 스위치, VLAN 안 나뉘어 있음)

💻 VPCS 설정
PC-1
ip 172.16.100.10 255.255.255.0 172.16.100.1

PC-2
ip 172.16.100.11 255.255.255.0 172.16.100.1

PC-3
ip 172.16.10.10 255.255.255.0 172.16.10.1

PC-4
ip 172.16.10.11 255.255.255.0 172.16.10.1

✅ 확인 명령어 (시험 단골)
```
show ip route
show ip nat translations
show ip nat statistics
```

PC에서:
```
ping 8.8.8.8
ping www.google.com
```
