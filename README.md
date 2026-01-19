# Cisco-Packet-Tracer


✅ 1단계. 물리 연결 & 기본 확인

라우터 전원 ON

라우터 간 Serial DCE 케이블 연결

PC ↔ Router는 Copper Straight-Through

시리얼 인터페이스 존재 확인

show ip interface brief

✅ 2단계. 인터페이스 IP 설정 (라우터별)
🔹 R1
conf t
interface se0/0
ip address 1.1.1.1 255.0.0.0
clock rate 64000
no shutdown

interface fa0/0
ip address 192.168.10.2 255.255.255.0
no shutdown

🔹 R2
conf t
interface se0/0
ip address 1.1.1.2 255.0.0.0
no shutdown

interface se0/1
ip address 2.1.1.1 255.0.0.0
clock rate 64000
no shutdown

interface fa0/0
ip address 126.255.255.2 255.0.0.0
no shutdown

🔹 R3
conf t
interface se0/0
ip address 2.1.1.2 255.0.0.0
no shutdown

interface se0/1
ip address 3.1.1.1 255.0.0.0
clock rate 64000
no shutdown

interface fa0/0
ip address 191.255.255.2 255.0.0.0
no shutdown

🔹 R4
conf t
interface se0/0
ip address 3.1.1.2 255.0.0.0
no shutdown

interface fa0/0
ip address 223.255.255.2 255.255.255.0
no shutdown

✅ 3단계. PC IP 설정
PC	IP	Gateway
PC1	192.168.10.1	192.168.10.2
PC2	126.255.255.1	126.255.255.2
PC3	191.255.255.1	191.255.255.2
PC4	223.255.255.1	223.255.255.2
✅ 4단계. OSPF 설정 (가장 중요)
🔹 R1 (Area 10)
router ospf 1
network 1.0.0.0 0.255.255.255 area 10
network 192.168.10.0 0.0.0.255 area 10

🔹 R2 (Area 10 / 0 / 20 → ABR)
router ospf 1
network 1.0.0.0 0.255.255.255 area 10
network 2.0.0.0 0.255.255.255 area 0
network 126.0.0.0 0.255.255.255 area 20

🔹 R3 (Area 0 / 40 / 30 → ABR)
router ospf 1
network 2.0.0.0 0.255.255.255 area 0
network 3.0.0.0 0.255.255.255 area 40
network 191.255.0.0 0.255.255.255 area 30

🔹 R4 (Area 40)
router ospf 1
network 3.0.0.0 0.255.255.255 area 40
network 223.255.255.0 0.0.0.255 area 40

✅ 5단계. OSPF 이웃 확인
show ip ospf neighbor ✔ 이웃 라우터가 FULL 상태여야 함

show ip ospf database ✔ LSA들이 Area 간에 교환되는지 확인

정상 출력:

FULL/ -

✅ 6단계. 라우팅 테이블 확인
show ip route


✔ 표시 확인:

O → 같은 Area

O IA → 다른 Area

✅ 7단계. 통신 테스트
PC1 > ping 223.255.255.1


👉 모든 PC 간 ping 성공해야 정상
