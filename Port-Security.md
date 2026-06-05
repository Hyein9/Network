# Port-Security 실습

## 개념 정리

### Port-Security란
- 스위치의 물리적 포트에 연결되는 MAC 주소를 제한하여 비인가 장치의 네트워크 접근을 차단하는 보안 기능
- MAC 주소를 기반으로 학습할 수 있는 주소의 개수를 지정하거나, 특정 인터페이스를 통해 접속 가능한 MAC 주소를 제한함

### 설정 조건
- Dynamic Port(자동 협상 상태)에서는 설정 불가. Trunk 또는 Access 포트에서만 설정 가능
- SPAN(미러링)의 목적지 포트에는 적용 불가
- EtherChannel에 설정 불가
- 실무에서는 Access 포트에 설정하는 것을 권장

### MAC 주소 학습 방식 3가지

| 방식 | 설명 | NVRAM 저장 | 특징 |
|------|------|:---------:|------|
| Sticky | 스위치가 동적으로 학습한 MAC을 running-config에 저장 | copy run start 시 가능 | 재부팅 후에도 유지 가능 |
| Static | 관리자가 수동으로 MAC 주소를 지정 | O | 특정 장비만 허용할 때 사용 |
| Dynamic | 스위치가 자동으로 학습 (별도 명령 없음) | X | 재부팅하면 학습 내용 초기화 |

### Violation Mode 3가지

보안 규정을 위반했을 때(허용되지 않은 MAC 주소가 접속했을 때) 스위치가 취하는 동작을 정의한다.

| 모드 | 위반 프레임 | 포트 상태 | 로그 기록 | 위반 카운터 |
|------|:---------:|:--------:|:--------:|:----------:|
| Protect | Drop | 유지 (up) | X | X |
| Restrict | Drop | 유지 (up) | O | O |
| Shutdown | Drop | err-disabled (down) | O | O |

**Protect**: 위반 장비의 프레임만 조용히 버림. 로그도 남기지 않아서 관리자가 위반 사실을 인지하기 어려움.

**Restrict**: 위반 장비의 프레임을 버리면서 로그를 남김. 허용된 장비는 정상 통신 가능.

**Shutdown**: 포트 자체를 닫아버리는 가장 강력한 대응. 포트가 err-disabled 상태가 되며, 복구하려면 관리자가 shutdown 후 no shutdown을 수행해야 함. Port-Security의 기본 Violation 모드.

---

## 토폴로지

```
  [PC1]        [PC2]        [PC3]
 (허용 장비)  (허용 장비)  (비인가 장비)
     |            |            |
   f0/1         f0/2         f0/1 (PC1 대신 연결 시도)
  +--+------------+------------+--+
  |              SW1              |
  |        Port-Security 적용     |
  +-------------------------------+
```

### 시나리오

- f0/1: PC1만 허용 (maximum 1)
- f0/2: MAC 2개까지 허용 (maximum 2)
- PC3이 f0/1에 연결을 시도하면 Violation 발생

---

## 실습 1: Sticky 방식 (동적 학습 + 저장)

스위치가 처음 연결된 장비의 MAC 주소를 자동으로 학습하고, 이후 해당 MAC만 허용한다.

### 설정

```
conf t
int f0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
end
```

### 동작 흐름

```
1. PC1을 f0/1에 연결
   → 스위치가 PC1의 MAC 주소를 자동 학습
   → running-config에 저장됨

2. PC1 정상 통신

3. PC1을 뽑고 PC3(비인가)을 f0/1에 연결
   → MAC 주소 불일치
   → Violation 발생 (shutdown 모드)
   → 포트가 err-disabled 상태로 전환
```

### 위반 발생 시 복구

```
conf t
int f0/1
 shutdown
 no shutdown
end
```

원인을 확인한 뒤 포트를 껐다가 다시 켜야 한다.

### Sticky MAC 확인

```
show port-security interface f0/1
show port-security address
show running-config interface f0/1
```

running-config에서 아래처럼 학습된 MAC이 보인다:
```
switchport port-security mac-address sticky 0050.7966.6801
```

설정을 유지하려면 저장:
```
copy run start
```

---

## 실습 2: Static 방식 (수동 MAC 지정)

관리자가 직접 허용할 MAC 주소를 지정한다. 특정 장비만 접속을 허용할 때 사용.

### 설정

```
conf t
int f0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address 0050.7966.6804
 switchport port-security violation restrict
end
```

### 동작 흐름

```
1. 0050.7966.6804 MAC을 가진 장비가 f0/1에 연결
   → 허용된 MAC이므로 정상 통신

2. 다른 MAC 주소를 가진 장비가 f0/1에 연결
   → 위반 프레임 Drop
   → 로그 발생 (restrict 모드)
   → 포트는 유지 (shutdown 안 됨)
```

### 위반 로그 예시

```
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred,
caused by MAC address 0050.7966.6806 on port FastEthernet0/1
```

---

## 실습 3: Dynamic 방식 (자동 학습, 저장 안 됨)

별도의 MAC 지정 명령 없이 스위치가 자동으로 학습한다. 재부팅하면 학습 내용이 초기화됨.

### 설정

```
conf t
int f0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation protect
end
```

### 동작 흐름

```
1. 장비 3대까지 f0/1에 연결 가능
   → MAC 주소 3개까지 자동 학습

2. 4번째 장비가 연결 시도
   → 위반 프레임 Drop (protect 모드)
   → 로그 없음, 포트 유지
   → 관리자가 위반 사실을 인지하기 어려움
```

---

## Violation Mode 비교 실습

### 동일 포트에 Violation 모드만 변경하면서 테스트

```
                   Protect          Restrict         Shutdown
                  +--------+      +--------+      +--------+
비인가 장비 접속 → | Drop   |      | Drop   |      | Drop   |
                  | 포트 up |      | 포트 up |      | 포트   |
                  | 로그 X  |      | 로그 O  |      | down   |
                  +--------+      +--------+      +--------+
                   조용히 차단       차단 + 알림      포트 폐쇄
```

실무에서는 보안 요구 수준에 따라 선택:
- 일반 사무실: restrict (차단하면서 로그로 모니터링)
- 서버실/보안 구역: shutdown (강력한 차단)
- 공용 공간: protect (접속만 막고 포트 유지)

---

## 확인 명령어 정리

| 명령어 | 용도 |
|--------|------|
| `show port-security` | Port-Security 설정 요약 (전체 포트) |
| `show port-security interface f0/1` | 특정 포트의 상세 보안 상태 |
| `show port-security address` | 학습된 MAC 주소 목록 |
| `show interfaces status` | 포트 상태 (err-disabled 확인) |
| `show running-config interface f0/1` | 해당 포트 설정 확인 (sticky MAC 포함) |

### show port-security interface 출력 예시

```
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Sticky MAC Addresses       : 1
Last Source Address         : 0050.7966.6801
Security Violation Count   : 0
```

---

## 설정 명령어 정리

```
-- 포트를 Access 모드로 설정 (필수)
switchport mode access

-- Port-Security 활성화
switchport port-security

-- 최대 MAC 주소 개수 제한
switchport port-security maximum [숫자]

-- Violation 모드 설정
switchport port-security violation [protect|restrict|shutdown]

-- Sticky 학습 활성화
switchport port-security mac-address sticky

-- 수동 MAC 주소 지정
switchport port-security mac-address [MAC주소]

-- err-disabled 포트 복구
shutdown
no shutdown
```

---

## 정리

- Port-Security는 MAC 주소 기반으로 스위치 포트의 접근을 제한하는 보안 기능
- 학습 방식은 Sticky(동적 학습 + 저장), Static(수동 지정), Dynamic(자동 학습) 3가지
- Violation 모드는 Protect(조용히 차단), Restrict(차단 + 로그), Shutdown(포트 폐쇄) 3가지
- Shutdown이 기본 모드이며 가장 강력한 대응. 복구 시 shutdown/no shutdown 필요
- Access 포트에서만 설정을 권장하며, 반드시 `switchport mode access` 후 `switchport port-security`를 활성화해야 함
