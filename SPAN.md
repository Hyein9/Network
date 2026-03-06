# SPAN (Switch Port Analyzer) / Port Mirroring 실습 가이드

## 1. 실습 목적

SPAN(Switch Port Analyzer) 또는 Port Mirroring 기능을 사용하여 특정 포트의 트래픽을 다른 포트로 복제하여 패킷 분석 장비(Sniffer)가 트래픽을 분석할 수 있도록 구성한다.

이 실습에서는 다음을 수행한다.

* PC1 ↔ PC2 간 통신 발생
* Switch에서 해당 포트 트래픽을 미러링
* Sniffer PC에서 트래픽 캡처

---

# 2. 네트워크 토폴로지

R1: 192.168.10.1/24
PC1: 192.168.10.10/24
PC2: 192.168.10.20/24

Switch 포트 구성

| 장비      | 인터페이스 | 연결     |
| ------- | ----- | ------ |
| R1      | f0/0  | Switch |
| PC1     | e0    | f1/2   |
| PC2     | e0    | f1/3   |
| Sniffer | e0    | f1/10  |

---

# 3. IP 설정

## PC1

```
ip 192.168.10.10/24 192.168.10.1
```

## PC2

```
ip 192.168.10.20/24 192.168.10.1
```

## Sniffer PC

(패킷 캡처용이므로 IP 설정은 필수 아님)

```
ip 192.168.10.30/24 192.168.10.1
```

---

# 4. Router 설정

```
enable
configure terminal

interface f0/0
ip address 192.168.10.1 255.255.255.0
no shutdown

end
write memory
```

---

# 5. Switch 기본 설정

```
enable
configure terminal

hostname ESW1

interface range f1/2 - 3
switchport mode access
no shutdown

interface f1/10
switchport mode access
no shutdown

end
write memory
```

---

# 6. SPAN (Port Mirroring) 설정

PC1, PC2 포트의 트래픽을 Sniffer 포트로 복제한다.

Source Port

* f1/2
* f1/3

Destination Port

* f1/10

설정 명령어

```
enable
configure terminal

monitor session 1 source interface f1/2
monitor session 1 source interface f1/3
monitor session 1 destination interface f1/10

end
write memory
```

---

# 7. SPAN 설정 확인

```
show monitor session 1
```

출력 예시

```
Session 1
---------
Type                   : Local Session
Source Ports           :
    Both               : Fa1/2
    Both               : Fa1/3
Destination Ports      : Fa1/10
```

---

# 8. 통신 테스트

PC1에서 PC2로 Ping 실행

```
ping 192.168.10.20
```

---

# 9. Sniffer에서 패킷 캡처

Sniffer PC에서 Wireshark 또는 tcpdump 실행

예시

```
tcpdump -i eth0
```

또는

Wireshark 실행 후 인터페이스 캡처 시작

PC1 ↔ PC2 통신 패킷(ICMP)을 확인할 수 있다.

---

# 10. SPAN 동작 원리

Port Mirroring은 특정 포트의 트래픽을 복사하여 다른 포트로 전달하는 기능이다.

트래픽 흐름

```
PC1 ----> Switch ----> PC2
           |
           |
           v
        Sniffer
```

Switch는 f1/2, f1/3 포트의 트래픽을 복사하여 f1/10 포트로 전달한다.

따라서 Sniffer는 실제 통신에 참여하지 않아도 모든 패킷을 확인할 수 있다.

---

# 11. SPAN 삭제 방법

```
configure terminal
no monitor session 1
end
```

---

# 12. 전체 명령어 정리

Switch

```
enable
configure terminal

monitor session 1 source interface f1/2
monitor session 1 source interface f1/3
monitor session 1 destination interface f1/10

end
```

---

# 13. 실습 결과

* PC1 ↔ PC2 정상 통신
* Switch에서 트래픽 미러링 발생
* Sniffer에서 모든 패킷 캡처 확인

---

# 14. 정리

SPAN / Port Mirroring은 다음과 같은 상황에서 사용된다.

* 네트워크 트래픽 분석
* 보안 모니터링
* IDS / IPS 연동
* 패킷 분석

대표적인 패킷 분석 도구

* Wireshark
* tcpdump

이 기능을 통해 네트워크 내부 트래픽을 쉽게 분석할 수 있다.
