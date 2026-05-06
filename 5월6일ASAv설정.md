좋습니다! 여기 GitHub README 양식으로 정리한 내용입니다. 바로 복붙해서 사용하세요:

```markdown
# ASAv 방화벽 네트워크 설정 가이드

## 📌 목차
1. [ASAv 개요](#asav-개요)
2. [네트워크 토폴로지](#네트워크-토폴로지)
3. [기본 설정](#기본-설정)
4. [OSPF 라우팅](#ospf-라우팅)
5. [SSH/Telnet/HTTP 접속 설정](#sshtelnethtp-접속-설정)
6. [방화벽 ACL 및 인바운드 정책](#방화벽-acl-및-인바운드-정책)
7. [실습 과제 및 테스트](#실습-과제-및-테스트)
8. [클라우드/원격 관리 설정](#클라우드원격-관리-설정)

---

## ASAv 개요

**ASAv**는 Cisco ASA 방화벽의 기능을 가상 머신(VM) 형태로 구현한 소프트웨어입니다.
- 하드웨어 장비 없이도 서버나 GNS3 같은 시뮬레이터 위에서 동일한 보안 기능 수행 가능
- 소프트웨어 기반 방화벽

---

## 네트워크 토폴로지

```
┌─────────────┐
│    Web      │ (192.168.100.200)
│  (Linux)    │
└──────┬──────┘
       │ eth0
       │ (192.168.100.0/24)
       │
┌──────┴──────────┐
│       R1        │ (1.1.1.1 - f0/0)
│    (Router)     │ (192.168.100.254 - f0/1)
└────────┬────────┘
         │ (1.1.1.0/24)
         │
┌────────┴────────┐
│      ASAv       │ (1.1.1.2 - G0/0)
│   (Firewall)    │ (2.2.2.2 - G0/1)
└────────┬────────┘
         │ (2.2.2.0/24)
         │
┌────────┴────────┐
│       R2        │ (2.2.2.1 - f0/0)
│    (Router)     │ (192.168.200.254 - f0/1)
└────────┬────────┘
         │ (192.168.200.0/24)
         │
┌────────┴────────┐
│   WebTerm2      │ (192.168.200.200)
│  (Windows PC)   │
└─────────────────┘

관리용:
└─ Management 0/0: 10.10.10.10/24
└─ Cloud VM: 10.10.10.20/24 (Loopback)
```

---

## 기본 설정

### 1️⃣ Web (Linux) 설정

```bash
# IP 주소 설정
ip addr add 192.168.100.200/24 dev eth0
ip link set eth0 up

# 기본 경로 설정 (R1을 게이트웨이로)
ip route add default via 192.168.100.254

# 네트워크 설정 확인/변경
ls /etc/network
vi /etc/network/interfaces
```

### 2️⃣ PC (Windows) 설정

```
IP Address: 192.168.100.100
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.100.254

명령어: save
테스트: ping 192.168.100.200 (핑 정상)
```

### 3️⃣ R1 (Router 1) 설정

```
R1# conf t

# f0/1 인터페이스 (Web 쪽)
R1(config)# interface f0/1
R1(config-if)# ip address 192.168.100.254 255.255.255.0
R1(config-if)# no shutdown

# f0/0 인터페이스 (ASAv 쪽)
R1(config)# interface f0/0
R1(config-if)# ip address 1.1.1.1 255.255.255.0
R1(config-if)# no shutdown
```

### 4️⃣ R2 (Router 2) 설정

```
R2# conf t

# f0/1 인터페이스 (WebTerm2 쪽)
R2(config)# interface f0/1
R2(config-if)# ip address 192.168.200.254 255.255.255.0
R2(config-if)# no shutdown

# f0/0 인터페이스 (ASAv 쪽)
R2(config)# interface f0/0
R2(config-if)# ip address 2.2.2.1 255.255.255.0
R2(config-if)# no shutdown
```

---

## OSPF 라우팅

### 🔍 OSPF란?

**OSPF (Open Shortest Path First)**는 Link State 라우팅 프로토콜입니다.
- 모든 연결된 장비가 자신의 네트워크 정보를 광고(Advertise)
- 전체 네트워크 지도를 형성하여 최단 경로 계산
- 라우터들이 서로의 위치를 알게 됨

### R1 OSPF 설정

```
R1(config)# router ospf 1
R1(config-router)# network 192.168.100.0 0.0.0.255 area 0
R1(config-router)# network 1.1.1.0 0.0.0.255 area 0
```

### R2 OSPF 설정

```
R2(config)# router ospf 1
R2(config-router)# network 192.168.200.0 0.0.0.255 area 0
R2(config-router)# network 2.2.2.0 0.0.0.255 area 0
```

### ASAv OSPF 설정

```
ciscoasa(config)# router ospf 1
ciscoasa(config-router)# network 1.1.1.0 255.255.255.0 area 0
ciscoasa(config-router)# network 2.2.2.0 255.255.255.0 area 0
```

---

## SSH/Telnet/HTTP 접속 설정

### 🔐 ASAv 원격 접속 설정

```
# 관리자 패스워드 (Enable)
ciscoasa(config)# enable password 1234

# 원격 접속용 패스워드 (Telnet/SSH)
ciscoasa(config)# passwd 1234

# SSH 사용자 계정
ciscoasa(config)# username estcamp password 1234
ciscoasa(config)# aaa authentication ssh console LOCAL

# SSH 키 생성
ciscoasa(config)# crypto key generate rsa modulus 1024

# 인터페이스 이름 설정 (G0/0)
ciscoasa(config)# interface GigabitEthernet0/0
ciscoasa(config-if)# ip address 1.1.1.2 255.255.255.0
ciscoasa(config-if)# nameif inside
ciscoasa(config-if)# security-level 100
ciscoasa(config-if)# no shutdown

# 인터페이스 이름 설정 (G0/1)
ciscoasa(config)# interface GigabitEthernet0/1
ciscoasa(config-if)# ip address 2.2.2.2 255.255.255.0
ciscoasa(config-if)# nameif outside2
ciscoasa(config-if)# security-level 0
ciscoasa(config-if)# no shutdown

# HTTP 서버 활성화
ciscoasa(config)# http server enable

# 설정 저장 및 확인
ciscoasa(config)# write memory
ciscoasa# show firewall
ciscoasa# show route
ciscoasa# show running-config
```

### 🔐 R2 원격 접속 설정

```
R2(config)# ip http server
R2(config)# ip http secure-server
R2(config)# enable password 1234
R2(config)# passwd 1234
R2(config)# username estcamp password 1234
R2(config)# aaa authentication ssh console LOCAL
R2(config)# crypto key generate rsa modulus 1024
```

### 📋 접근 제어 설정

```
# SSH 접근 허용: 2.2.2.x 대역에서만
ciscoasa(config)# ssh 2.2.2.0 255.255.255.0 outside

# HTTP 접근 허용: WebTerm2(192.168.200.200)만
ciscoasa(config)# http 192.168.200.200 255.255.255.255 outside

# Telnet 접근 허용: R2(2.2.2.1)만 (테스트용)
ciscoasa(config)# telnet 2.2.2.1 255.255.255.255 outside
```

### 🎯 접근 제어 설명

| 명령어 | 의미 |
|--------|------|
| `ssh 2.2.2.0 255.255.255.0 outside` | "2.2.2.x 대역(R2 쪽)의 누구든 SSH로 접속 가능" |
| `http 192.168.200.200 255.255.255.255 outside` | "오직 WebTerm2(192.168.200.200)만 웹 브라우저로 관리 페이지 접속 가능" |
| `telnet 2.2.2.1 255.255.255.255 outside` | "R2(2.2.2.1)만 Telnet으로 접속 가능" (테스트용) |

### 🧪 원격 접속 테스트 (R2에서)

```bash
# SSH 테스트
ssh -l estcamp 2.2.2.2
# 결과: ASAv SSH 접속 성공 ✓

# Telnet 테스트
telnet 2.2.2.2
# 결과: ASAv Telnet 접속 성공 ✓
```

---

## 방화벽 ACL 및 인바운드 정책

### 📊 보안 레벨 개념

ASAv의 인터페이스는 **보안 레벨(0-100)**을 가집니다:
- **높은 레벨 → 낮은 레벨**: 기본 허용 (예: inside(100) → outside(0))
- **낮은 레벨 → 높은 레벨**: 기본 차단 (예: outside(0) → inside(100))

### 🚫 인바운드(Inbound) 정책

**외부(Lower Security) → 내부(Higher Security)로 들어오는 트래픽은 엄격하게 제어**

```
ciscoasa(config)# access-list outinside extended permit tcp 1.1.1.0 255.255.255.0 any eq www
ciscoasa(config)# access-group outinside in interface outside
```

### ⚠️ ICMP 핑 차단 이유

내부에서 외부로 나간 핑에 대해, 돌아오는 응답 패킷이 방화벽에서 차단되는 경우:
- **원인**: 외부에서 내부로 들어오는 ICMP 응답이 기본 차단됨
- **해결**: Stateful Inspection 활성화 필요 (아래 참고)

---

## 실습 과제 및 테스트

### ✅ 과제 1: 텔넷 테스트 (외부 R1 → 내부 R2 웹 서비스)

**목표**: 외부(R1)에서 내부(R2)의 웹 서비스(80번 포트)로 접속 허용

```
ciscoasa(config)# access-list outinside extended permit tcp 1.1.1.0 255.255.255.0 host 2.2.2.1 eq 80
ciscoasa(config)# access-group outinside in interface outside
```

**테스트** (R1에서 실행):
```
telnet 2.2.2.1 80
# 결과: Connected to 2.2.2.1 메시지 출력 ✓
```

---

### ✅ 과제 2: 외부 핑 테스트 (R1 → R2)

**목표**: 외부(R1)에서 내부(R2)로 핑 허용 (기본값: 차단)

```
ciscoasa(config)# icmp permit any outside
```

**테스트**:
```
# 설정 전 (R1에서)
ping 2.2.2.1
# 결과: Timeout (실패) ✗

# 설정 후 (R1에서)
ping 2.2.2.1
# 결과: Reply (성공) ✓
```

---

### ✅ 과제 3: 내부 핑 테스트 (R2 → R1)

**목표**: 내부(R2)에서 외부(R1)로 핑 및 응답 허용

**원리**: 높은 레벨(100) → 낮은 레벨(0)은 기본 허용이지만, **돌아오는 응답 패킷을 위해 Stateful Inspection 활성화 필요**

```
ciscoasa(config)# policy-map global_policy
ciscoasa(config-pmap)# class inspection_default
ciscoasa(config-pmap-c)# inspect icmp
```

**테스트** (R2에서):
```
ping 1.1.1.1
# 결과: Reply (성공) ✓
```

---

### 📌 인내관 정책 요약

| 방향 | 내용 | 설정 |
|------|------|------|
| 외부 → 내부 | 기본 차단 | ACL로 명시적 허용 필요 |
| 내부 → 외부 | 기본 허용 | Policy-map으로 상태 검사 활성화 |
| ICMP 응답 | 돌아오는 응답 | `inspect icmp` 필수 |

---

## 클라우드/원격 관리 설정

### 🌐 Management 인터페이스 설정 (ASAv)

```
ciscoasa(config)# interface management0/0
ciscoasa(config-if)# ip address 10.10.10.10 255.255.255.0
ciscoasa(config-if)# no shutdown
```

### ☁️ 클라우드 VM 설정

```
# Loopback으로 관리용 IP 설정
10.10.10.20/24
```

### 🖥️ Windows PC 라우팅 설정 (관리자 권한)

```cmd
# 라우팅 테이블 확인
route print

# ASAv 관리 네트워크로의 경로 추가
route add 10.10.10.0 mask 255.255.255.0 10.10.10.10

# R2 네트워크로의 경로 추가
route add 192.168.200.0 mask 255.255.255.0 192.168.200.254

# 확인
route print
```

---

## 📌 핵심 개념 정리

### OSPF vs SSH 설정 비유

| 구분 | 역할 | 비유 |
|------|------|------|
| **OSPF** | R2에게 ASAv의 IP(2.2.2.2)로 가는 경로 제공 | 🗺️ 내비게이션 (길 찾기) |
| **SSH 설정** | ASAv의 2.2.2.x 대역 접근을 명시적으로 허용 | 📋 출입 허가 명단 (문 열기) |

**결론**: OSPF가 없으면 도착지를 몰라서 갈 수 없고, SSH 설정이 없으면 도착했는데 문이 닫혀있는 것!

---

## 🔧 유용한 명령어

```
# ASAv
ciscoasa# show firewall              # 방화벽 상태 확인
ciscoasa# show running-config        # 현재 설정 확인
ciscoasa# show route                 # 라우팅 테이블 확인
ciscoasa# show ospf neighbor         # OSPF 이웃 확인
ciscoasa# write memory               # 설정 저장

# 라우터
R#(config)# do show running-config   # 현재 설정 확인
R#(config)# do show ip route         # 라우팅 테이블 확인
R#(config)# do show ip ospf neighbor # OSPF 이웃 확인
```

---

## 📚 학습 체크리스트

- [ ] OSPF 개념 이해 (Link State, 라우팅 광고)
- [ ] 네트워크 토폴로지 파악
- [ ] R1, R2, ASAv 기본 IP 설정
- [ ] OSPF 라우팅 설정 및 이웃 확인
- [ ] SSH/Telnet/HTTP 접근 제어 설정
- [ ] ACL과 Security Level의 관계 이해
- [ ] Inbound/Outbound 정책 차이 인식
- [ ] Stateful Inspection의 필요성 이해
- [ ] Management 인터페이스를 통한 원격 관리

---

**마지막 수정**: 2026년 5월 6일
```

이 README를 GitHub에 올릴 때 파일명은 `README.md`로 하고 복붙하면 됩니다! 🚀
