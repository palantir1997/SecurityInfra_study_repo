# 오늘의 보안 스터디 노트

---

## 1. Graylog (SIEM) 설치 및 설정

### 시스템 사전 설정

```bash
# 타임존 설정
sudo timedatectl set-timezone Asia/Seoul

# 메모리 맵 설정
sudo sysctl -w vm.max_map_count=262144
sudo echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-greylog-datanode.conf
sudo sysctl -p
```

### MongoDB 7.0 설치

```bash
# GPG 키 등록
sudo curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# 키 파일 존재 확인
ls -l /usr/share/keyrings/mongodb-server-7.0.gpg

# APT 소스 등록
sudo echo "deb [ arch=amd64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# 설치 확인
cat /etc/apt/sources.list.d/mongodb-org-7.0.list

# 패키지 설치
sudo apt install gnupg
sudo apt clean
sudo apt-get install -y \
  mongodb-org=7.0.14 \
  mongodb-org-database=7.0.14 \
  mongodb-org-server=7.0.14 \
  mongodb-org-mongos=7.0.14 \
  mongodb-org-tools=7.0.14 \
  mongodb-mongosh
```

### Graylog 6.1 설치

```bash
# 저장소 패키지 다운로드 및 설치
wget https://packages.graylog2.org/repo/packages/graylog-6.1-repository_latest.deb
sudo dpkg -i graylog-6.1-repository_latest.deb

# Graylog Datanode 설치
sudo apt install -y graylog-datanode
```

### 비밀번호 시크릿 키 생성 (password_secret)

```bash
# 96자리 랜덤 문자열 생성
sudo < /dev/urandom tr -dc A-Z-a-z-0-9 | head -c${1:-96}; echo;
```

> **생성 예시:**  
> `97RAjy-6TXOYItHS4m7YnzqQ3e22Sn5VmACIGCdXE1J5E0RaxZ8Awm5zwmVrH0D0gUZRZnv0lv477g1s9URopVt6VEtYFfAS`  
> → `/etc/graylog/datanode/datanode.conf`의 `password_secret` 항목에 입력

```bash
# Datanode 서비스 시작
sudo systemctl enable --now graylog-datanode
```

### 관리자 패스워드 해시 생성 (root_password_sha2)

```bash
# admin 비밀번호의 SHA-256 해시 생성
sudo echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1
```

> **입력:** `admin`  
> **출력:** `8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918`

### server.conf 주요 설정

```bash
sudo vi /etc/graylog/server/server.conf
```

| 라인 | 설정 항목 | 값 |
|------|----------|----|
| 57번 | `password_secret` | 위에서 생성한 96자리 랜덤 키 |
| 68번 | `root_password_sha2` | SHA-256 해시값 |
| 106번 | `http_bind_address` | `0.0.0.0:9000` |

```bash
# Graylog 서버 시작
sudo systemctl enable --now graylog-server

# 로그 확인 (초기 접속 정보 확인)
sudo cat /var/log/graylog-server/server.log
```

> **접속 정보 예시:**  
> `Initial configuration is accessible at 0.0.0.0:9000, with username 'admin' and password 'FUUhHkXYSt'.`

---

## 2. Splunk 설치 및 설정

### 네트워크 연결 확인

```bash
ping 8.8.8.8
```

### 전용 계정 생성

```bash
# Splunk 전용 사용자 생성 (홈 디렉터리: /opt/splunk, 쉘: /bin/sh)
sudo useradd -d /opt/splunk -s /bin/sh splunk

# 비밀번호 설정
sudo passwd splunk
```

> Splunk 설치 및 프로세스 실행 전용 계정을 분리해 권한을 최소화하는 보안 모범 사례

### .deb 패키지 다운로드

```bash
wget -O splunk-9.3.0-51ccf43db5bd-linux-2.6-amd64.deb \
  "https://download.splunk.com/products/splunk/releases/9.3.0/linux/splunk-9.3.0-51ccf43db5bd-linux-2.6-amd64.deb"
```

### 패키지 설치

```bash
sudo dpkg -i splunk-9.3.0-51ccf43db5bd-linux-2.6-amd64.deb
```

### Splunk 최초 실행

```bash
sudo /opt/splunk/bin/splunk start
```

> **실행 시 진행 사항:**
> 1. 라이선스 동의 → `y` 입력 (두 번)
> 2. 관리자 계정 정보 입력 (ID / 비밀번호 설정)

### 부팅 시 자동 실행 등록

```bash
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
```

> `splunk` 계정 권한으로 부팅 시 자동 시작되도록 systemd에 등록

### 웹 대시보드 접속

```
http://<Splunk_IP>:8000
```

> 브라우저에서 위 주소로 접속 → 설정한 관리자 계정으로 로그인 → 대시보드 화면 진입

---

## 3. CTF - Momentum2 머신 공략

### 정보 수집 (Reconnaissance)

```bash
# 네트워크 내 호스트 스캔
sudo nmap -sn 172.16.11.0/24

# 타깃 포트 및 서비스 상세 스캔
sudo nmap -sV -O -sC -v 172.16.11.224

# 웹 디렉터리 브루트포싱
sudo gobuster dir \
  -u http://172.16.11.224 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,log,bak,html
```

### 백업 파일 발견 및 분석

```bash
# 백업 파일 요청
sudo curl http://172.16.11.224/ajax.php.bak
```

> **분석 포인트:**  
> - 상태 코드 `200 OK`이지만 응답 크기 `0` → 서버 내부 처리 로직만 존재  
> - PHP는 서버에서 실행되어 소스가 노출되지 않으므로 개발자가 남긴 백업 파일을 노린다  
> - 백업 파일 확장자 탐색: `.bak`, `.old`, `.save`, `.tmp`

### 취약점 공략 전략

```
1. 쿠키 브루트포싱
   - 쿠키 패턴: &G6u@B6uDXMq&Ms[A-Z]
   - 마지막 대문자 한 자리(A~Z)를 변경하며 시도
   - Burp Suite Intruder 기능 활용

2. 파일 업로드 우회
   - 올바른 쿠키 확인 후 POST 파라미터에 secure=val1d 추가
   - PHP 웹쉘 업로드 시도
   - 응답에 1이 출력되면 업로드 성공
```

### 대문자 브루트포싱 스크립트

```python
# upper.py - A~Z 대문자 목록 생성
a = 65
for i in range(26):
    print(chr(a + i))
```

```bash
sudo vi upper.py
```

### 리버스쉘 업로드

```bash
# 웹쉘 목록 확인
sudo ls /usr/share/webshells/php/

# 리버스쉘 복사 후 수정
sudo cp /usr/share/webshells/php/php-reverse-shell.php ./php-reverse-shell.php
```

> **수정 위치:**  
> - `49번 라인`: 칼리 리눅스 IP 주소 입력  
> - `50번 라인`: 리스닝 포트 입력

### 발견된 주요 경로

| 경로 | 설명 |
|------|------|
| `/dashboard.html` | 파일 업로드 기능 존재 |
| `/owls` | 파일 업로드 관련 처리 |
| `/ajax.php.bak` | 백업 파일 (핵심 로직 포함) |

---

## 4. 프록시 개념 정리

### Burp Suite 동작 방식

```
Kali 브라우저 ──> [ Burp Suite 프록시 ] ──> 타깃 웹 서버
```

> Burp Suite는 브라우저와 서버 사이의 길목을 지키며 모든 패킷을 가로채는 **중간자(Man-in-the-Middle) 검문소** 역할

### HTTP 프록시 vs SOCKS 프록시 비교

| 구분 | HTTP 프록시 | SOCKS 프록시 |
|------|------------|-------------|
| 동작 계층 | 7계층 (응용) | 5계층 (세션) |
| 지원 프로토콜 | HTTP/HTTPS 전용 | 모든 종류의 트래픽 |
| 데이터 수정 | 헤더 변경 등 수정 가능 | 데이터를 그대로 포워딩 |
| 오버헤드 | 상대적으로 높음 | 낮음 |
| 활용 예시 | 웹브라우징 | 게임, P2P, 이메일 등 |
