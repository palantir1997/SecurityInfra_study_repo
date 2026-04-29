# 📝 오늘 배운 내용 요약

## 1. Zabbix 모니터링 서버 구축

### Zabbix 포트 및 설정

**클라이언트 (236번 서버)**
* 방화벽 포트 10050/tcp 개방 (클라이언트 포트)
  - 서버의 질문에 응답하는 역할
  - `sudo firewall-cmd --permanent --add-port=10050/tcp`
* Zabbix Agent2 시작 및 활성화
  - `sudo systemctl start zabbix-agent2`
  - `sudo systemctl enable zabbix-agent2`

**서버 (235번 서버)**
* 방화벽 포트 10051/tcp 개방 (서버 포트)
  - 클라이언트들의 보고를 수신하는 역할
  - `sudo firewall-cmd --permanent --add-port=10051/tcp`
* 추가 포트 개방: HTTP(80), HTTPS(443), MariaDB(3306)
* 서비스 재시작
  - `sudo systemctl restart zabbix-server`
  - `sudo systemctl restart apache2`
  - `sudo systemctl restart mariadb`

### 클라이언트 에이전트 설정

**에이전트 설정 파일 수정**
* 파일 위치: `/etc/zabbix/zabbix_agent2.conf`
* 수정 항목 3가지
  1. **Server**: 235번(Zabbix 서버 IP)로 지정
  2. **ServerActive**: 235번(Zabbix 서버 IP)로 지정
  3. **Hostname**: 각 클라이언트 이름으로 설정 (예: db1)

### 동작 원리

**클라이언트-서버 구조**
* 서버(235번): 10051 포트로 에이전트들의 보고 수신
* 클라이언트(236번): 10050 포트로 서버의 질문에 응답

**⚠️ 주의사항**
* Grafana와 ClickHouse는 메모리 사용량이 많아 프로세스 종료 필요

---

## 2. GNS3 라우터 기본 설정 및 OSPF

### 라우터 기본 인터페이스 설정

**기본 명령어 구조**
```
enable
configure terminal
interface fastEthernet 0/0
ip address 10.10.10.1 255.255.255.0
no shutdown
exit
```

### OSPF (Open Shortest Path First) 라우팅 프로토콜

**1단계: 인터페이스 IP 설정**
* 각 라우터의 연결된 인터페이스에 IP 주소 할당
* 예시
  - R1 (f0/0 → R2): 10.10.10.1/24
  - R2 (f0/0 → R1): 10.10.10.2/24
  - R2 (f0/1 → R3): 10.10.20.2/24
  - R3 (f0/1 → R2): 10.10.20.3/24
  - R3 (f0/0 → R4): 10.10.30.3/24
  - R4 (f0/0 → R3): 10.10.30.4/24

**2단계: OSPF 활성화 및 네트워크 광고**
* 명령어: `router ospf 1`
* 각 라우터별 설정
  - R1: `network 10.10.10.0 0.0.0.255 area 0`
  - R2: `network 10.10.10.0 0.0.0.255 area 0` + `network 10.10.20.0 0.0.0.255 area 0`
  - R3: `network 10.10.20.0 0.0.0.255 area 0` + `network 10.10.30.0 0.0.0.255 area 0`
  - R4: `network 10.10.30.0 0.0.0.255 area 0`

**3단계: OSPF 동작 확인**
* `show ip ospf neighbor` - 이웃 라우터 확인
* `show ip route` - 라우팅 테이블 확인

---

## 3. 디버깅 및 패킷 분석

### Debug 명령어

**기본 디버깅**
* `debug ip ?` - IP 관련 디버그 옵션 확인
* `debug ip packet` - 모든 IP 패킷 추적
* `debug ip ospf hello` - OSPF Hello 패킷 주고받기 추적

**디버그 종료**
* `undebug all` 또는 `u all` - 모든 디버그 종료

---

## 4. ACL (Access Control List) - 방화벽 정책

### 1️⃣ 기본 ACL (Static ACL)

**Stateless 방식의 문제점**
* 나가는 패킷은 허용되지만, 되돌아오는 응답 패킷이 막힐 수 있음
* 예: R1의 Ping 요청은 성공하지만, R4의 응답이 R2에서 차단

```
ip access-list extended tcpes
 permit ospf host 10.10.20.3 any
 permit tcp any any established
 exit
ip access-group tcpes in
```

### 2️⃣ RACL (Reflexive ACL) - **세션 기반 제어**

**동작 원리**
* **나가는 패킷(Outgoing)**: 기억하기 (Reflect)
* **들어오는 패킷(Incoming)**: 기억난 패킷인지 확인 (Evaluate)

**설정 예시**
```
! 나가는 트래픽 기록 (Reflect)
ip access-list extended racl
 permit tcp any any reflect racltest
 permit udp any any reflect racltest
 permit icmp any any reflect racltest
 exit

! 들어오는 트래픽 확인 (Evaluate)
ip access-list extended test
 permit ospf host 10.10.20.3 any
 evaluate racltest
 exit

! ACL 적용
ip access-group racl out
ip access-group test in
```

**장점**: 내부 사용자는 끊김없이 인터넷 사용, 외부에서 내부 접근 불가

### 3️⃣ Dynamic ACL - **인증 기반 접근 제어**

**설정 절차**
```
! 1. 사용자 인증 설정
username est password 1234
line vty 0 4
login local
autocommand access-enable host timeout 10

! 2. ACL 작성 (OSPF 허용 + Telnet 허용 + 동적 규칙)
ip access-list extended dacl
 permit ospf host 10.10.20.3 any
 permit tcp any host 10.10.20.2 eq telnet
 dynamic DACL permit ip any any
 exit

! 3. ACL 적용
ip access-group dacl in
```

**동작**
* 정책상 모든 통신이 차단되지만, Telnet으로 인증 후 10분간 접근 허용
* 인증 정보가 동적으로 임시 규칙 생성

---

## 5. ZFW (Zone-Based Policy Firewall) - **고급 방화벽**

### 동작 원리

**구성 5단계**
1. 존(Zone) 생성
2. 존 멤버 지정
3. 존 쌍(Zone-Pair) 설정
4. 트래픽 분류(Class-Map) 정의
5. 정책(Policy-Map) 적용

### 설정 예시

**1단계: 존 생성 및 멤버 지정**
```
zone security inside
zone security outside

interface fa0/0
zone-member security inside

interface fa0/1
zone-member security outside
```

**특징**: 같은 존 내 통신은 허용, 다른 존 간 통신은 정책에 따름

**2단계: 존 쌍 생성**
```
zone-pair security in-out source inside destination outside
```

**3단계: 트래픽 분류 (Class-Map)**
```
ip access-list extended acltest
 permit ip any any
 exit

class-map type inspect classtest
 match access-group name acltest
 exit
```

**Match의 의미**: "이 Class-Map에 들어오는 패킷 중, access-group 조건과 일치하는 것들만 분류"

**4단계: 정책 설정 (Policy-Map)**
```
policy-map type inspect policytest
 class type inspect classtest
  inspect  ! 세션 감시
  exit
 exit
```

**5단계: 정책 적용**
```
zone-pair security in-out
 service-policy type inspect policytest
 exit
```

### 동작 확인

**세션 모니터링**
```
show policy-map type inspect zone-pair sessions
```
* 활성 세션 목록 표시
* 방화벽이 통신을 감시 중임을 확인 가능

---

## 6. 실습 시나리오: R5-R6 추가 연결

### R5-R6 인터페이스 설정

```
! R5 설정
R5(config)# interface fa0/0
R5(config-if)# ip address 10.10.50.5 255.255.255.0
R5(config-if)# no shutdown

! R6 설정
R6(config)# interface fa0/0
R6(config-if)# ip address 10.10.60.6 255.255.255.0
R6(config-if)# no shutdown
```

### R6 → R1 트래픽 허용 (방화벽 R3 기준)

```
! R3에서 R6 접근 허용 ACL 생성
R3(config)# ip access-list extended R6_TO_R1_ANY
R3(config-ext-nacl)# permit ip host 10.10.60.6 host 10.10.10.1
R3(config-ext-nacl)# exit

! R6 클래스 맵 생성
R3(config)# class-map type inspect match-any R6_ALLOW_CLASS
R3(config-cmap)# match access-group name R6_TO_R1_ANY
R3(config-cmap)# exit

! 정책 맵에 포함
R3(config)# policy-map type inspect HTTP_POLICY
R3(config-pmap)# class type inspect R6_ALLOW_CLASS
R3(config-pmap-c)# pass  ! inspect 대신 pass 사용 (Any 허용)
R3(config-pmap-c)# exit
```

### 최종 통합 설정 (R3)

```
! 클래스 맵 통합
class-map type inspect match-any HTTP_CLASS
 match protocol http
 match access-group name HTTP_ONLY
 match access-group name HTTP_ACL

class-map type inspect match-all ICMP_CLASS
 match access-group name ICMP_ACL

class-map type inspect match-any R6_ALLOW_CLASS
 match access-group name R6_TO_R1_ANY

! 정책 맵 통합
policy-map type inspect HTTP_POLICY
 class type inspect R6_ALLOW_CLASS
  pass
 class type inspect HTTP_CLASS
  inspect
 class type inspect ICMP_CLASS
  pass
 class class-default
  drop

! 존 페어 적용
zone-pair security IN_TO_OUT source INSIDE destination OUTSIDE
 service-policy type inspect HTTP_POLICY
```

---

# 💭 오늘의 회고

## 1. 배운 점

* **Zabbix 아키텍처 이해**: 클라이언트-서버 구조에서 포트 역할 명확화
  - 클라이언트: 질문에 응답 (10050)
  - 서버: 보고 수신 (10051)

* **OSPF 라우팅 프로토콜**: 동적 라우팅으로 자동 경로 학습
  - Area 기반 계층적 구조
  - Hello 메시지로 이웃 관계 형성

* **ACL의 진화 과정**: Stateless → Reflexive → Dynamic → ZFW
  - 각 방식의 장단점 이해
  - 보안 수준에 따른 선택 기준

* **ZFW의 강력함**: 세션 기반 Stateful Firewall
  - 존 기반으로 명확한 트래픽 흐름 제어
  - Inspect로 양방향 세션 추적

---

## 2. 어려운 점/개선할 점

* **ACL 방향 이해**: In/Out 방향 설정이 직관적이지 않음
  - 응답 패킷의 방향을 고려해야 함
  - 더 많은 실습으로 패킷 흐름 시각화 필요

* **ZFW 설정의 복잡성**: 클래스 맵, 정책 맵, 존 쌍이 얽혀있음
  - 순서대로 단계별 설정이 중요
  - 한 번에 이해하기 어려워 반복 실습 필요

* **메모리 관리**: Grafana/ClickHouse의 리소스 낭비
  - 불필요한 서비스 종료 필요
  - 프로덕션 환경에서의 최적화 전략 부족

---

## 3. 액션 플랜

- [ ] OSPF 다중 Area 구성 실습
- [ ] 복합 ACL 시나리오 3개 이상 직접 설계해보기
- [ ] ZFW에서 다양한 프로토콜(FTP, DNS) 허용/차단 정책 추가
- [ ] Zabbix 커스텀 모니터링 항목 추가 (디스크, 네트워크)
- [ ] GNS3 토폴로지 6개 라우터 이상으로 확장

---

## 4. 함께 나누고 싶은 점

* OSPF와 BGP의 선택 기준이 무엇인가요?
  - 실무에서는 언제 OSPF를 쓰고 언제 BGP를 쓰나요?

* ZFW에서 Logging을 추가하면 성능 영향이 큰가요?
  - 프로덕션 환경의 Best Practice가 궁금합니다.

* Zabbix의 자동 Discovery 기능 활용 방법?
  - 수동 설정 없이 자동으로 에이전트를 찾을 수 있나요?

---


**위 가이드에서 꼭 기억할 3가지:**

## 📌 RACL 핵심
```
출발: R1 → R4 핑
           ↓ "아, R1이 처음 시작했네" (Reflect에 기록)
돌아옴: R4 → R1 응답
           ↓ "맞다, 이건 기록된 세션이다!" (Evaluate 통과) ✅

역방향: R4 → R1 핑 (처음 시도)
           ↓ "R4가? 기록이 없는데?" (기록 없음) ❌ 차단!
```

**RACL은 "내부에서 시작한 것"만 기억합니다!**

---

## 📌 ZFW 핵심  
```
5단계 구성:
1️⃣ Zone 만들기: inside, outside
2️⃣ Zone-Pair: inside→outside 방향 정의
3️⃣ Class-Map: HTTP인지 ICMP인지 분류
4️⃣ Policy-Map: inspect/pass/drop 결정
5️⃣ 적용: Zone-Pair에 정책 붙이기
```

**ZFW는 "이것은 HTTP"라고 명확히 판정한 후 정책을 적용합니다!**

---

## ⚡ 시험 팁

**"반대 방향도 허용해야 한다"** → ZFW 선택! 🎯
- RACL: 불가능 (내부에서만 시도 가능)
- ZFW: 클래스맵 수정으로 가능

**"이건 HTTP만, 저건 FTP만"** → ZFW! 🎯
- RACL: 복잡 (포트로 일일이 분석)
- ZFW: 클래스맵 2개 + 정책맵 수정으로 끝

----

show zone security          ← 존이 제대로 만들어졌는지
show zone-pair security     ← 존 쌍이 제대로 연결되었는지
show policy-map type inspect zone-pair sessions  ← 세션이 정말 추적되는지
