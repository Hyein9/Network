# 📡 RACL (Reflexive ACL) 실습 정리

---

# 1️⃣ 개념

- 내부 → 외부 트래픽 기준으로 세션 생성 (reflect)
- 외부 → 내부는 해당 세션 응답만 허용 (evaluate)
- 외부에서 시작하는 접속은 기본 차단

---

# 2️⃣ 토폴로지

```
R1 --- R2 --- R3 --- R4
```

- R1-R2: 1.1.12.0/24
- R2-R3: 1.1.23.0/24
- R3-R4: 1.1.34.0/24

---

# 3️⃣ IP 설정

## R1
```bash
conf t
int s0/0
 ip address 1.1.12.1 255.255.255.0
 no shutdown
exit
```

## R2
```bash
conf t
int s0/0
 ip address 1.1.12.2 255.255.255.0
 no shutdown
exit

int s0/1
 ip address 1.1.23.2 255.255.255.0
 no shutdown
exit
```

## R3
```bash
conf t
int s0/0
 ip address 1.1.23.3 255.255.255.0
 no shutdown
exit

int s0/1
 ip address 1.1.34.3 255.255.255.0
 no shutdown
exit
```

## R4
```bash
conf t
int s0/0
 ip address 1.1.34.4 255.255.255.0
 no shutdown
exit
```

---

# 4️⃣ OSPF 설정

## R1
```bash
router ospf 1
 network 1.1.12.0 0.0.0.255 area 0
```

## R2
```bash
router ospf 1
 network 1.1.12.0 0.0.0.255 area 0
 network 1.1.23.0 0.0.0.255 area 0
```

## R3
```bash
router ospf 1
 network 1.1.23.0 0.0.0.255 area 0
 network 1.1.34.0 0.0.0.255 area 0
```

## R4
```bash
router ospf 1
 network 1.1.34.0 0.0.0.255 area 0
```

---

# 5️⃣ Telnet 설정

## R1 / R4
```bash
conf t
line vty 0 4
 password cisco
 login
 transport input all
end
```

---

# 6️⃣ RACL 설정 (R2)

## OUT ACL (세션 생성)
```bash
conf t
ip access-list extended acl-out
 permit tcp any any reflect myracl
 permit udp any any reflect myracl
 permit icmp any any reflect myracl
 permit ip any any
exit
```

## IN ACL (세션 검사)
```bash
ip access-list extended acl-in
 permit ospf host 1.1.23.3 any
 evaluate myracl
exit
```

## 인터페이스 적용
```bash
int s0/1
 ip access-group acl-out out
 ip access-group acl-in in
end
```

---

# 7️⃣ 테스트

## 내부 → 외부
```bash
R1# ping 1.1.34.4
R1# telnet 1.1.34.4
```

## 외부 → 내부 (차단)
```bash
R4# telnet 1.1.12.1
```

---

# 8️⃣ 외부 허용 (옵션)

```bash
conf t
ip access-list extended acl-in
 permit tcp host 1.1.34.4 host 1.1.12.1 eq telnet
end
```

---

# 9️⃣ 트러블슈팅

```bash
show ip ospf neighbor
show ip route
show ip interface brief
show ip access-lists
```

---

# 🔥 핵심 요약

- reflect → 세션 생성
- evaluate → 세션 검사
- 내부 시작만 허용
- 외부 직접 접근 차단

---

# 💬 한 줄 정리

👉 내가 요청한 것만 응답 허용
