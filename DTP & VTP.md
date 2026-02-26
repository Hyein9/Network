## DTP(Dynamic Trunking Protocol)
switchport mode dynamic auto  switchport mode dynamic auto  Access

switchport mode dynamic auto  switchport mode dynamic auto


### DTP-Auto
소극적 협상으로, 상대편이 trunk 또는 Desirable 이 아닐경우 Access 로 설정된다.
L2 스위치에서 기본값이며, Access 친화적이다.
설정하는 방법은 아래와 같다
// 특정 인터페이스에 접속 후
Switch(config-if)# switchport mode dynamic auto


### DTP-Desirable
적극적 협상으로, 상대편이 Access 가 아니라면 Trunk 로 설정된다.
L3 스위치에서 기본값이며, Trunk 친화적이다.
설정하는 방법은 아래와 같다.
// 특정 인터페이스에 접속 후
Switch(config-if)# switchport mode dynamic desirable
switchport mode dynamic desirable     switchport mode dynamic desirable    Trunk


##VTP(Vlan trunking Protocol)
1. need to use same domain & same password
2. mode
   1) Server : VLAN 정보를 직접 생성, 전송할 수 있고 (VLAN 1 ~ 1005)까지만 관리가 가능하다.
   2) Client : 같은 domain에 password가 설정된 VTP server로부터 VLAN 정보를 받아 저장한다.  
               직접 VLAN 정보를 설정할 수 없다.
   3) Transparent : 자신만의 VLAN 정보를 설정할 수 있고, 다른 스위치에게 자신의 VLAN 정보를 전송하지 않으며, VTP server로부터 받은 정보로 동기화하지 않지만 받은 정보는 다른 스위치에게 전송한다.
                    VLAN(1~4094) 생성 및 관리 가능하고 Revision number 값은 항상 0 입니다.


VTP 특징
VLAN 설정 정보는 받아오지만 스위치 Port 정보들은 받아오지 않는다.
스위치의 기본 설정으로는 Server로 되어 있고 Domain은 설정되어 있지 않다.
확장 VLAN (1006~4094)은 Transparent 모드에서만 사용이 가능하다.
VTP 설정하다 문제가 발생할 수 있기에 (패스워드 설정 -> 도메인 설정 -> 모드 설정) 순으로 해야 안전하다.
스위치 순서에도 중요하기에 (VTP 모드 설정 -> Trunk port 설정 -> VLAN 설정 -> Port에 VLAN 설정) 순으로 해줘야 안전하다.
보안상의 문제로 VTP Transparent mode를 권장합니다.


VTP 설정 조건
스위치에서는 VTP가 기본적으로 동작되며 끌 수 없는 프로토콜입니다.
VTP를 사용하려면 스위치들의 VLAN domain name을 같도록 설정해야지만 VLAN 정보를 동기화 할 수 있습니다.
VTP 동기화 메시지를 주고 받으려면 스위치간에 연결된 포트를 Trunk Port로 설정을 해줘야 합니다.
VLAN 정보가 무분별하게 동기화 될 수 있기에 VTP password를 설정해주세요. 스위치간에 VTP password가 맞지않으면 동기화 되지 않습니다.


실습
✅ 전체 구성 개요

SW1 → VTP Server (VLAN 생성 담당)

SW2 → VTP Transparent (VTP 영향 X, 자기 VLAN 따로 관리)

SW3 → VTP Client (VLAN 생성 ❌, 서버에서 받아오기)

VTP 정보:

Domain: AWS

Password: 0213

VLAN: 10, 20, 30, 40

🔵 SW1 (VTP Server + VLAN 생성)
en
conf t
hostname SW1

vtp mode server
vtp domain AWS
vtp password 0213

vlan 10
 name AWS
vlan 20
 name 505
vlan 30
vlan 40

end
wr


확인:

show vtp status
show vlan brief


✔️ 결과

VLAN 10,20,30,40 생성됨

Revision 증가

SW3으로 VLAN 전파됨

🟡 SW2 (VTP Transparent + 자기 VLAN 따로 관리)
en
conf t
hostname SW2

vtp mode transparent
vtp domain AWS
vtp password 0213

interface range f0/1 - 2
 switchport mode trunk

vlan 99

end
wr

확인:
show vtp status
show vlan brief


✔️ 결과
VLAN 99는 SW2에만 존재
SW1, SW3에는 VLAN 99 안 생김
Transparent는 VTP 광고 전달만 하고, DB 동기화 안 함

🟢 SW3 (VTP Client + VLAN 수신 전용)
en
conf t
hostname SW3

vtp mode client
vtp domain AWS
vtp password 0213

end
wr

확인:
show vtp status
show vlan brief

✔️ 결과
SW1에서 만든 VLAN 10,20,30,40 자동 생성됨
vlan 10 직접 만들려고 하면 → ❌ 에러 (네가 본 메시지 정상임)


