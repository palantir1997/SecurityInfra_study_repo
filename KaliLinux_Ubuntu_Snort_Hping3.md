# 🛡️ 보안 인프라 구축 실습 정리

> 네트워크 보안 수업 실습 기록 — Nmap · Snort 3 · IDS/IPS 개념 정리

---

## 📌 실습 환경

| 항목 | 값 |
|------|-----|
| 내 IP | `172.16.11.213` |
| 타겟 IP | `172.16.11.237` |
| 게이트웨이 | `172.16.11.254` |
| 네트워크 대역 | `172.16.11.0/24` |

---

## 📚 목차

1. [모의 해킹 프로세스 (Kill Chain)](#1-모의-해킹-프로세스)
2. [Nmap — 네트워크 스캔](#2-nmap--네트워크-스캔)
3. [tcpdump — 패킷 캡처](#3-tcpdump--패킷-캡처)
4. [hping3 — DoS 공격 테스트](#4-hping3--dos-공격-테스트)
5. [Snort 3 설치 전체 흐름](#5-snort-3-설치-전체-흐름)
6. [Snort NIC 설정](#6-snort-nic-설정-프로미스큐어스-모드)
7. [Snort 설정 파일 수정](#7-snort-설정-파일-수정)
8. [룰 파일 생성 및 검증](#8-룰-파일-생성-및-검증)
9. [IDS/IPS 핵심 개념 정리](#9-idsips-핵심-개념-정리)
10. [Snort vs Suricata 비교](#10-snort-vs-suricata-비교)
11. [설치 진행 상태](#11-설치-진행-상태)

---

## 1. 모의 해킹 프로세스

사이버 킬 체인(Cyber Kill Chain) — 공격자의 시각으로 보안을 이해하는 프레임워크

| 단계 | 명칭 | 설명 |
|------|------|------|
| 1 | 정찰 (Reconnaissance) | ping, 포트 스캔, 서비스 스캔으로 정보 수집 |
| 2 | 무기화 (Weaponization) | 취약점 분석, 공격 코드 준비 |
| 3 | 전달 (Delivery) | 악성 코드 전송 |
| 4 | 공격 (Exploitation) | 취약점 실제 공격 |
| 5 | 설치 (Installation) | 백도어 설치 |
| 6 | C&C | 원격 제어 |
| 7 | 목표 달성 (Actions on Objectives) | 데이터 탈취, 시스템 파괴 |

---

## 2. Nmap — 네트워크 스캔

### 호스트 탐색 (살아있는 IP 확인)

```bash
sudo nmap -sn 172.16.11.0/24
```

결과에서 확인할 항목:
- `Nmap scan report for [IP]` → 살아있는 장치 목록
- `Host is up` → 통신 가능한 상태
- `MAC Address` → 가상머신(VMware 등) 여부 판단 가능

### 자주 쓰는 옵션 모음

| 명령어 | 설명 |
|--------|------|
| `nmap -sn <target>` | 포트 스캔 없이 호스트만 탐색 (Ping Scan) |
| `nmap -p <범위> <target>` | 특정 포트 범위 스캔 |
| `nmap -p- <target>` | 모든 TCP 포트 스캔 (0~65535) |
| `nmap -sV <target>` | 열린 포트의 서비스 버전 감지 |
| `nmap -O <target>` | 운영체제 감지 |
| `nmap -A <target>` | 공격적 스캔 (OS + 버전 + 스크립트 + traceroute) |
| `nmap -sS <target>` | SYN 스캔 (3-way handshake 미완성, 은밀함) |
| `nmap -sT <target>` | TCP 연결 스캔 (포트 열림/닫힘 확인) |
| `nmap -sU -p <범위> <target>` | UDP 포트 스캔 |
| `nmap -T<0-5> <target>` | 속도 조절 (0=가장 느림, 5=가장 빠름) |
| `nmap -iL <파일> <target>` | 파일에서 대상 호스트 읽어서 스캔 |
| `nmap --script <스크립트명> <target>` | NSE 스크립트로 정보 수집 |
| `nmap -p 80,443,22 <target>` | 특정 포트만 스캔 |

---

## 3. tcpdump — 패킷 캡처

패킷을 실시간으로 캡처하는 도구

```bash
# 기본 실행 (모든 패킷 캡처)
sudo tcpdump

# 특정 인터페이스 지정
sudo tcpdump -i eth0
```

---

## 4. hping3 — DoS 공격 테스트

### SYN Flooding 공격

```bash
# SYN 패킷 1000개 전송
sudo hping3 172.16.11.235 -S -c 1000
```

### DoS 공격 유형

| 공격 유형 | 설명 |
|-----------|------|
| SYN Flooding | 다량의 SYN 패킷으로 서버 자원 고갈 |
| LAND Attack | 출발지/목적지 IP를 동일하게 설정 → 시스템 루프 유발 |
| Smurf Attack | 출발지 IP 변조 후 브로드캐스트로 다량 응답 유도 |

---

## 5. Snort 3 설치 전체 흐름

### 5-1. 의존성 패키지 설치

```bash
sudo apt install build-essential libpcap-dev libpcre3-dev libnet1-dev \
zlib1g-dev luajit hwloc libdumbnet-dev bison flex liblzma-dev openssl \
libssl-dev pkg-config libhwloc-dev cmake cpputest libsqlite3-dev uuid-dev \
libcmocka-dev libnetfilter-queue-dev libmnl-dev autotools-dev \
libluajit-5.1-dev libunwind-dev libfl-dev -y

sudo apt install libpcre2-dev -y
sudo apt install -y git
```

### 5-2. libdaq 설치 (Snort 데이터 수집 부품)

```bash
sudo git clone https://github.com/snort3/libdaq.git
cd libdaq
sudo ./bootstrap
sudo ./configure
sudo make
sudo make install
cd ~
```

### 5-3. gperftools 설치 (성능 향상 도구)

```bash
sudo wget https://github.com/gperftools/gperftools/releases/download/gperftools-2.13/gperftools-2.13.tar.gz
sudo tar xzf gperftools-2.13.tar.gz
cd gperftools-2.13
sudo ./configure
sudo make install
sudo ldconfig
```

### 5-4. Snort 3 빌드 및 설치

```bash
# snort3 소스 폴더에서 실행
sudo ./configure_cmake.sh --prefix=/usr/local --enable-tcmalloc
sudo make install
sudo ldconfig

# 설치 확인
sudo snort -V
```

### 5-5. apt 오류 날 때 (잠금 파일 충돌)

```bash
sudo rm /var/lib/apt/lists/lock
sudo rm /var/cache/apt/archives/lock
sudo rm /var/lib/dpkg/lock*
sudo dpkg --configure -a
sudo apt update
```

---

## 6. Snort NIC 설정 (프로미스큐어스 모드)

### 프로미스큐어스 모드란?

> 일반 NIC는 자신의 MAC 주소가 목적지인 패킷만 처리한다.  
> 프로미스큐어스 모드를 켜면 **네트워크 상의 모든 패킷**을 수신해서 분석할 수 있다.  
> Snort 같은 IDS가 전체 망을 감시하려면 반드시 필요하다.

### 수동으로 켜기

```bash
sudo ip link set dev enp0s3 promisc on
sudo ethtool -K enp0s3 gro off lro off

# 확인
ip addr
sudo ethtool -k enp0s3 | grep receive-offload
```

### 부팅 시 자동 실행 서비스 등록

```bash
sudo vi /usr/lib/systemd/system/snort3-nic.service
```

```ini
[Unit]
Description=Set Snort 3 NIC in promiscuous mode and Disable GRO, LRO on boot
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip link set dev enp0s3 promisc on
ExecStart=/usr/sbin/ethtool -K enp0s3 gro off lro off
TimeoutStartSec=0
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start snort3-nic
sudo systemctl enable snort3-nic
```

---

## 7. Snort 설정 파일 수정

### snort.lua 열기

```bash
sudo vi /usr/local/etc/snort/snort.lua
```

### 188번 라인 — 네트워크 대역 설정

```lua
HOME_NET = '172.16.11.0/16'
EXTERNAL_NET = '!$HOME_NET'
```

### 197번 라인 — 룰 파일 연결

```lua
ips =
{
    variables = default_variables,
    rules = [[
        include /usr/local/etc/snort/rules/local.rules
    ]]
}
```

---

## 8. 룰 파일 생성 및 검증

```bash
cd /usr/local/etc/snort
sudo mkdir rules
cd rules
sudo touch local.rules
ls local.rules

# 설정 검증 — 에러 없으면 "Successfully validated" 출력
sudo snort -c /usr/local/etc/snort/snort.lua
```

### 검증 화면에서 확인할 3가지 포인트

| 확인 항목 | 위치 | 의미 |
|-----------|------|------|
| `Successfully validated` | 맨 아랫줄 | 설정 파일 문법 오류 없음 |
| `total rules loaded: 219` | 중간 `rule counts` 섹션 | 룰이 정상적으로 로딩됨 |
| `pcap DAQ configured to passive` | 하단 | 수동 감시 모드 (차단 아닌 탐지만) |

---

## 9. IDS/IPS 핵심 개념 정리

### Snort 3가지 동작 모드

| 모드 | 설명 |
|------|------|
| Sniffer 모드 | 패킷을 읽어서 화면에 출력 (tcpdump와 유사) |
| Packet Logger 모드 | 탐지 패킷을 디스크에 로그 파일로 저장 |
| NIDS/NIPS 모드 | 실시간 패킷 분석 후 룰에 따라 탐지/차단 |

### Snort 룰 구조 ★시험 단골★

```
alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"HTTP Access"; content:"GET"; sid:1000001;)
```

**룰 헤더 (Rule Header)**

| 항목 | 예시 | 설명 |
|------|------|------|
| Action | `alert` | alert / log / pass / drop |
| Protocol | `tcp` | tcp / udp / icmp / ip |
| 출발지 | `$EXTERNAL_NET any` | IP : 포트 |
| 방향 | `->` | 단방향 / `<>` 양방향 |
| 목적지 | `$HOME_NET 80` | IP : 포트 |

**룰 옵션 (Rule Options)**

| 옵션 | 설명 |
|------|------|
| `msg` | 로그에 남길 메시지 |
| `content` | 패킷에서 찾을 문자열 |
| `sid` | 룰 고유 번호 (100만번대 이상 사용) |

### IDS vs IPS

| 구분 | IDS | IPS |
|------|-----|-----|
| 역할 | 탐지 후 경보 | 탐지 후 실시간 차단 |
| Snort 모드 | 기본 모드 | `-Q` 옵션 (Inline Mode) |
| 인터페이스 | 1개 | 2개 이상 필요 |

---

## 10. Snort vs Suricata 비교

| 항목 | Snort | Suricata |
|------|-------|----------|
| 스레딩 | 싱글 스레드 | 멀티 스레드 (대용량 처리 유리) |
| 룰 호환 | Snort 룰 | Snort 룰 호환 |
| 로그 형식 | 텍스트 | EVE JSON (`eve.json`) |
| 설정 검증 | `snort -c [파일] -T` | `suricata -T -c [파일] -v` |
| 로그 모니터링 | — | `tail -f eve.json \| jq 'select(.event_type=="alert")'` |

---

## 11. 설치 진행 상태

| 단계 | 작업 내용 | 상태 |
|------|-----------|------|
| 1단계 | Snort 3 설치 | ✅ 완료 |
| 2단계 | NIC 서비스 파일 생성 | ✅ 완료 |
| 3단계 | snort.lua 설정 수정 | ✅ 완료 |
| 4단계 | 설정 검증 | ✅ 성공 |

---

## 기타 유용한 명령어

```bash
# 성능 모니터링 도구 설치
sudo apt install htop

# Docker 컨테이너 중지
sudo docker stop pmm-server
```

---
