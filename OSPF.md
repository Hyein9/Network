```markdown
# Cisco Packet Tracer를 활용한 다중 Area OSPF 라우팅 구축 가이드

본 문서는 Cisco Packet Tracer 환경에서 4대의 라우터와 4대의 PC를 사용하여 다중 Area OSPF(Open Shortest Path First) 라우팅 프로토콜을 설정하고 검증하는 과정을 담은 기술 가이드입니다. 

OSPF는 대규모 네트워크에서 널리 사용되는 링크 상태(Link-State) 라우팅 프로토콜입니다. 네트워크 규모가 커질 수록 라우팅 테이블이 비대해지고 SPF 연산 부담이 증가하므로, 이를 방지하기 위해 네트워크를 여러 영역(Area)으로 분할하여 관리합니다. 본 실습에서는 중심이 되는 Backbone Area(Area 0)를 기반으로 Regular Area(Area 10, Area 20, Area 30, Area 40)들을 유기적으로 연결하고, 영역 간 라우팅을 수행하는 ABR(Area Border Router)의 역할을 이해할 수 있도록 구성되었습니다.

---

## 1. 네트워크 토폴로지 개요 및 설계

### 1.1 라우터 및 IP 주소 할당 구조
* **R1 (Area 10):** 내부망(192.168.10.0/24)을 가지며 R2와 Serial 연결(1.1.1.1/8)을 형성합니다.
* **R2 (ABR - Area 10 / Area 0 / Area 20):** 3개의 영역에 걸쳐 있는 경계 라우터입니다. R1 방향(Area 10), R3 방향(Area 0), 그리고 내부망(126.0.0.0/8 - Area 20)을 연결합니다.
* **R3 (ABR - Area 0 / Area 40 / Area 30):** R2 방향(Area 0), R4 방향(Area 40), 그리고 내부망(191.255.0.0/16 - Area 30)을 연결하는 경계 라우터입니다.
* **R4 (Area 40):** 내부망(223.255.255.0/24)을 가지며 R3와 Serial 연결(3.1.1.2/8)을 형성합니다.

### 1.2 OSPF 영역(Area) 구성도 안내
* **Area 10:** R1(전체), R2(Serial 0/0)
* **Area 0 (Backbone):** R2(Serial 0/1), R3(Serial 0/0)
* **Area 20:** R2(FastEthernet 0/0)
* **Area 30:** R3(FastEthernet 0/0)
* **Area 40:** R3(Serial 0/1), R4(전체)

---

## 2. 1단계: 물리 연결 및 기본 확인

1.  **장치 배치 및 전원 확인:** 라우터와 PC 장비를 배치한 후, 라우터의 전원이 켜져 있는지 확인합니다. (필요 시 Serial 인터페이스 카드가 장착되어 있지 않다면 WIC-2T 등의 모듈을 전원을 끈 상태에서 장착 후 재부팅합니다.)
2.  **케이블 연결:**
    * 라우터 간 연결: `Serial DCE` 케이블을 사용하여 연결합니다. (DCE 측 인터페이스에는 `clock rate` 설정이 필요합니다.)
    * PC와 라우터 간 연결: `Copper Straight-Through` 케이블을 사용하여 라우터의 FastEthernet 포트와 PC의 NIC 포트를 연결합니다.
3.  **인터페이스 상태 확인:** CLI 창에서 아래 명령어를 통해 시리얼 및 이더넷 인터페이스의 존재 여부와 초기 상태를 확인합니다.
    ```text
    show ip interface brief
    ```

---

## 3. 2단계: 라우터별 인터페이스 IP 및 활성화 설정

각 라우터의 인터페이스에 IP 주소를 할당하고 서브넷 마스크를 지정합니다. DCE 케이블이 연결된 포트에는 `clock rate 64000`을 적용하며, 모든 인터페이스는 `no shutdown` 명령어로 활성화합니다.

### R1 설정
```text
configure terminal
interface serial 0/0
 ip address 1.1.1.1 255.0.0.0
 clock rate 64000
 no shutdown
exit

interface fastethernet 0/0
 ip address 192.168.10.2 255.255.255.0
 no shutdown
exit

```
