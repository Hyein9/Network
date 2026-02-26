# Port-Security
물리적포트에 연결되는 Mac주소를 제한하여 비인가된 장치의 네트워크 접근을 차단하는 기능
; Mac주소를 기반으로 학습할 수 있는 주소의 개수를 지정하거나, 특정 인터페이스를 통해 접속 가능한 Mac 주소의 개수를 제한하는 보안 기능.

# 설정 조건
- Dynamic Port 설정 불가(Trunk, Access 포트만 설정가능)
- Span의 목적지 포트엔느 적용 불가
- etherchannel에 설정불가
- 가능한 액세스에 설정을 권장함

학습가능한 최대 Mac주소 개수 설정과 Port-security 활성화
Switch(config-if)# switchport port-security maximum 3
Switch(config-if)# switchport port-security


1. 기본설정(sticky방식)
스위치가 Mac-address를 동적으로 학습되도록하며, 저장된 주소를 저장할 수 있다.
```
Switch(config)# int f0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 1
Switch(config-if)# switchport port-security violation shutdown
Switch(config-if)# switchport port-security mac-address sticky
```
2. 수동 MAC 주소 지정(Static)
수동으로 학습을 원하는 Mac-address를 지정해준다. NVram에 저장할 수 있다.
```
Switch(config)# int f0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 1
Switch(config-if)# switchport port-security mac-address 0011.2233.4455
Switch(config-if)# switchport port-security violation restrict
```
3. 동적 MAC 주소 지정(Dynamic)
따로 명령어를 사용하지 않고 스위치가 Mac-address를 동적으로 학습되도록 나눈다. 스위치에 의해 자동으로 학습된 주소는 NVram에 저장할 수 없다.
```
Switch(config)# int f0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 3
Switch(config-if)# switchport port-security violation restrict
```
port-security 설정
port-security를 설정할 인터페이스의 동작 모드를 설정한다. (여기서는 access 모드로 설정한다.)
``` 
SW(config)# int f0/23
SW(config-if)# switchport mode access
해당 인터페이스에 port-security를 활성화한다.
SW(config-if)# switchport port-security
접속가능한 IP 주소의 최대 개수를 지정한다.
SW(config-if)# switchport port-security maximum [number]
port-security 보안 규정을 위반한 경우, 동작할 모드를 설정한다.
SW(config-if)# switchport port-security violation [protect|restrict|shutdown]
```
출처: https://b1jou39.tistory.com/54 [돌끼랑:티스토리]



#Port-Security Violation Mode 3가지
Switch(config-if)#switchport port-security violation [protect | restrict | shutdown]

(1) Violation protect
위반한 장비 통신을 차단하고 허용된 호스트는 통신가능
;Port-security 조건을 위반한 MAC 주소를 가진 장비의 모든 Frame Drop + Log 기록 X + 보안 위반한 장비만 통신 불가 (shutdown X)
```
SW1(config)#interface ethernet 0/0
SW1(config-if)#switchport port-security
SW1(config-if)#switchport port-security maximum 2
SW1(config-if)#switchport port-security violation
SW1(config-if)#switchport port-security violation protect
```
(2)Violation shutdown
Port-Security가 동작하는 포트에서 위반했을 경우 포트 자체를 닫아버림.(가장 강경대응방법)
해당 포트는 Shutdown이 되며 err-disabled 상태로 넘어감.
이렇게 인터페이스의 에러가 생긴 경우에는 원인을 확인하고 다시 복구시켜야함.
Port-Security Default Violation
```
SW1(config)#interface ethernet 0/0
SW1(config-if)#switchport port-security
SW1(config-if)#switchport port-security mac-address 0050.7966.6804
SW1(config-if)#switchport port-security violation shutdown
```
-> Shutdown 모드에서 보안 위반으로 포트가 down되면 shutdown 명령어로 포트를 죽이고 no sh 명령을 통해 다시 활성화 가능

(3) Violation restrict (protect + log발생)
Port-Security가 동작하는 포트에서 위반한 MAC주소를 가진 장비의 모든 Frame을 Drop시킨다.
Drop과 동시에 위반한 MAC주소에 대해서 Log가 발생한다.
```
SW1(config)#interface ethernet 0/0
SW1(config-if)#switchport port-security
SW1(config-if)#switchport port-security maximum 2
SW1(config-if)#switchport port-security violation restrict
 ```
최대 2개까지 가능하게 했을때 0050.7966.6806의 MAC주소가 제한 되었다는 Log 입니다.
```*Mar 26 15:15:47.206: %PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 0050.7966.6806 on port Ethernet0/0.```


