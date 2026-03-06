# GRE over IPsec Lab
## Topology
```PC1 --- SW1 --- R1 ===== R2 ===== R3 --- SW2 --- PC2

LAN1 : 192.168.10.0/24
LAN2 : 192.168.20.0/24

R1 WAN : 10.10.10.1
R3 WAN : 10.10.20.1

Tunnel Network : 172.16.10.0/30
```
### 1. PC IP 설정
```
PC1
IP 192.168.10.10
Mask 255.255.255.0
Gateway 192.168.10.1

PC2
IP 192.168.20.10
Mask 255.255.255.0
Gateway 192.168.20.1
```
### 2. R1 기본 설정
```
enable
conf t

hostname R1

interface g0/0
ip address 192.168.10.1 255.255.255.0
no shutdown

interface s0/0
ip address 10.10.10.1 255.255.255.252
no shutdown

exit
```
### 3. R2 기본 설정
```
enable
conf t

hostname R2

interface s0/0
ip address 10.10.10.2 255.255.255.252
no shutdown

interface s0/1
ip address 10.10.20.2 255.255.255.252
no shutdown

exit
```
### 4. R3 기본 설정
```
enable
conf t

hostname R3

interface g0/0
ip address 192.168.20.1 255.255.255.0
no shutdown

interface s0/1
ip address 10.10.20.1 255.255.255.252
no shutdown

exit
```
### 5. Static Route 설정
```
R1
ip route 192.168.20.0 255.255.255.0 10.10.10.2
R2
ip route 192.168.10.0 255.255.255.0 10.10.10.1
ip route 192.168.20.0 255.255.255.0 10.10.20.1
R3
ip route 192.168.10.0 255.255.255.0 10.10.20.2
```
### 6. GRE Tunnel 생성
```
R1
interface tunnel1
ip address 172.16.10.1 255.255.255.252
tunnel source 10.10.10.1
tunnel destination 10.10.20.1
no shutdown

R3
interface tunnel1
ip address 172.16.10.2 255.255.255.252
tunnel source 10.10.20.1
tunnel destination 10.10.10.1
no shutdown
```
### 7. 암호화 트래픽 정의 (ACL)
```
R1
access-list 100 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255

R3
access-list 100 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```
8. ISAKMP Policy (Phase1)
```
R1
crypto isakmp policy 10
authentication pre-share
exit
R3
crypto isakmp policy 10
authentication pre-share
exit
```
9. Pre Shared Key 설정
R1
crypto isakmp key 0 aws address 10.10.20.1
R3
crypto isakmp key 0 aws address 10.10.10.1
10. IPsec Transform Set
R1
crypto ipsec transform-set IPSEC esp-aes 256
R3
crypto ipsec transform-set IPSEC esp-aes 256
11. Crypto Map 생성
R1
crypto map GRE 10 ipsec-isakmp
set peer 10.10.20.1
set transform-set IPSEC
match address 100
exit
R3
crypto map GRE 10 ipsec-isakmp
set peer 10.10.10.1
set transform-set IPSEC
match address 100
exit
12. Tunnel Interface에 적용
R1
interface tunnel1
crypto map GRE
R3
interface tunnel1
crypto map GRE
13. VPN 확인
show crypto isakmp sa
show crypto ipsec sa

정상 상태 예시

local crypto endpt.: 10.10.10.1
remote crypto endpt.: 10.10.20.1

패킷 증가 확인

#pkts encrypt
#pkts decrypt
14. 테스트

PC1 → PC2

ping 192.168.20.10
전체 설정 흐름
1 PC IP 설정
2 Router 기본 IP 설정
3 Static Route
4 GRE Tunnel 생성
5 ACL (암호화 트래픽 정의)
6 ISAKMP Policy
7 Pre-shared Key
8 Transform Set
9 Crypto Map
10 Interface 적용
11 Ping 테스트
