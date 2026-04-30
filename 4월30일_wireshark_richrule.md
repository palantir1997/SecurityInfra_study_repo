# 📝 오늘 배운 내용 요약

## 1. 네트워크 보안 인프라 구축 (GNS3)

* **라우터 기본 설정**: IP 주소 할당, 인터페이스 활성화 (no shutdown)
* **DACL (Access Control List)**: R3 → PC1만 허용, 나머지 차단
  ```
  access-list 101 permit icmp host 3.3.3.2 host 192.168.10.1
  access-list 101 deny icmp any any
  ```
* **ZFW (Zone-Based Firewall)**: Inside/Outside/DMZ 존 기반 정책
  - Inside → Outside/DMZ: Any 통신 허용
  - Outside/DMZ → Inside: ICMP only 허용
  - 기본 정책: ALL DENY

## 2. 프로토콜 분석 (Wireshark)

* **3-way Handshake**: TCP 연결 과정 (SYN, SYN-ACK, ACK)
* **Telnet 평문 전송**: "Password required..." 등이 평문으로 네트워크에 노출
* **HTTP**: TCP 포트 80, 마찬가지로 평문 전송 위험
* **디스플레이 필터링**: http, tcp, udp, icmp, ospf 등 프로토콜 기준 필터링

## 3. Linux 서버 방화벽 (firewalld vs iptables)

* **firewalld**: Zone 기반, 동적 설정, 일반 서버 운영 권장
  ```
  sudo firewall-cmd --list-all                          # 현재 존 정책 확인
  sudo firewall-cmd --list-all-zones                    # 모든 존 확인
  sudo firewall-cmd --permanent --add-rich-rule='...'   # 정책 추가
  sudo firewall-cmd --reload                            # 설정 적용
  ```
* **iptables**: Chain 기반, 정적 설정, 레거시 스크립트 운영
  ```
  sudo iptables -A INPUT -s 172.16.11.233 -j DROP      # 특정 IP 차단
  sudo iptables -L -n -v                                # 규칙 상세 확인
  ```

---

# 💭 오늘의 회고

## 1. 배운 점
- 네트워크 3계층(DACL) + 4계층(ZFW)의 차원이 다른 방화벽 정책 이해
- Wireshark로 실제 패킷 분석: 평문 전송의 위험성 직접 확인
- Linux 방화벽의 두 가지 철학: Zone 기반(firewalld) vs Chain 기반(iptables)

## 2. 어려운 점/개선할 점
- ZFW의 zone-pair 설정이 복잡함 (4가지 정책 조합)
- firewalld rich-rule 문법이 길고 복잡
- Wireshark에서 원하는 패킷 필터링 하는 데 시간 소요

## 3. 액션 플랜
- ✅ DACL/ZFW 설정 자동화 스크립트 만들기
- ✅ firewalld 주요 명령어 치트시트 작성
- ✅ Wireshark 필터링 규칙 정리

## 4. 함께 나누고 싶은 점
- SSH(22번 포트) 권장: Telnet 대신 암호화된 원격 접속 사용
- HTTPS(443번 포트) 권장: HTTP 대신 암호화된 웹 통신 사용
- 현업에서는 firewalld 사용이 일반적 (CentOS, RHEL)

---
