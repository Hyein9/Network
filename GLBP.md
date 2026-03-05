## 1. GLBP 
- HSRP나 VRRP와 같이 게이트웨이 이중화 기능을 제공하면서 동시에 별도의 설정 없이 로드밸런싱 기능을 제공
- 1개의 가상 IP 주소, 여러개의 MAC 주소를 사용(그룹당 4개)
- 3초마다 IP주소 224.0.0.102, UDP 포트 3222번으로 통신(Active와 Standby 각각)
- GLBP 그룹 하나 당 한개의 AVG 선출 나머지가 AVF 상태
- 시스코 전용 프로토콜

## 2. AVG(Active Virtual Gateway)
- 각 멤버들(AVF)에게 가상 MAC 주소 할당
- 게이트웨이 주소 ARP 요청에 응답
 : PC등이 보내는 게이트웨이 IP주소에 대한 ARP요청에 대하여 AVF에게 할당한 MAC주소들로 응답하여 자동으로 부하분산
- AVG 선출 우선순위
  A: GLBP 우선순위 값이 높은 라우터
  B: 인터페이스 IP주소가 높은 라우터

ex) 00007.b4ZZ.ZZXX 가상 MAC 형식
ZZ = 그룹번호
XX = AVF ID

