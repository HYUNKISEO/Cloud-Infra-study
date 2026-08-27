# 시큐어한 원격 접속의 기본: SSH 설정 및 키 인증 (authorized_keys)

> **학습 목표:** SSH(Secure Shell)의 동작 원리를 이해하고, 비밀번호 인증 대신 안전한 **비대칭 키(공개키/개인키) 인증 방식**을 구축하며, SSH 보안 설정(`sshd_config`) 및 접속 장애 트러블슈팅 능력을 습득합니다.

---

## 1. SSH 및 비대칭 키 인증 원리

**SSH(Secure Shell)**는 암호화되지 않은 기존의 Telnet, FTP 등을 대체하여 리눅스 서버에 안전하게 원격 접속하기 위한 암호화 네트워크 프로토콜(기본 포트: 22)입니다.

### 1.1 비밀번호 방식 vs SSH 키 인증 방식

| 구분 | 비밀번호(Password) 인증 | SSH 키(Key) 인증 |
| :--- | :--- | :--- |
| **인증 방식** | 계정 비밀번호 입력 | 비대칭 키(개인키/공개키) 수학적 검증 |
| **보안성** | 무차별 대입 공격(Brute Force)에 취약 | 키 파일 없이는 침입 불가능 (극상의 보안) |
| **편의성** | 접속할 때마다 비밀번호 입력 필요 | 최초 설정 후 비밀번호 없이 자동 접속 |
| **실무 권장** | ❌ 사용 금지 (개발/테스트에만 제한적 사용) | **✅ 현업 필수 표준 규격** |

<Image src="image_agent_tag_9184575780395344337" alt="SSH 공개키 및 개인키 인증 구조 시퀀스 다이어그램" caption="SSH 키 기반 인증 메커니즘" />

---

### 1.2 비대칭 키 쌍(Key Pair)의 구조

* **개인키 (Private Key, 예: `id_ed25519`):** 
  * **내 컴퓨터(클라이언트)**에만 보관하는 비밀 키입니다. **절대로 외부에 유출되어서는 안 됩니다.**
* **공개키 (Public Key, 예: `id_ed25519.pub`):** 
  * **접속할 원격 리눅스 서버**의 `~/.ssh/authorized_keys` 파일에 등록하는 키입니다. 외부에 공개되어도 안전합니다.

---

## 2. SSH 키 쌍 생성 및 서버 등록 실습

### 2.1 내 컴퓨터에서 키 쌍 생성 (`ssh-keygen`)

```bash
# 1. ED25519 알고리즘으로 키 쌍 생성 (최신 표준, RSA보다 빠르고 안전함)
ssh-keygen -t ed25519 -C "my-server-key"

# (참고) 구형 시스템 호환성이 필요한 경우 RSA 4096비트 사용:
# ssh-keygen -t rsa -b 4096 -C "my-server-key"
```
* 명령어를 실행하면 키를 저장할 경로(`~/.ssh/id_ed25519`)와 Passphrase(비밀번호) 설정 여부를 물어봅니다. Enter를 눌러 기본값으로 진행합니다.

생성 결과 파일:
* `~/.ssh/id_ed25519` : **개인키** (클라이언트 보관)
* `~/.ssh/id_ed25519.pub` : **공개키** (서버 전송용)

---

### 2.2 원격 서버에 공개키 등록하기

#### 방법 A: `ssh-copy-id` 명령어 사용 (가장 편리함)

```bash
# 공개키를 원격 서버(192.168.1.100)의 ubuntu 계정에 자동으로 등록
ssh-copy-id ubuntu@192.168.1.100
```

#### 방법 B: 수동으로 등록하기 (명령어가 없거나 제한된 환경)

```bash
# 1. 내 컴퓨터의 공개키 내용 확인 및 복사
cat ~/.ssh/id_ed25519.pub

# 2. 원격 서버 접속 후 authorized_keys 파일 끝에 붙여넣기
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

---

## 3. SSH 접속 장애의 90%: 파일/디렉토리 권한(Permission) 설정

리눅스 SSH 데몬(`sshd`)은 보안상 **키 파일이나 디렉토리의 권한이 너무 느슨하면 접속을 자동으로 거부**합니다.

> 🛑 **[필수 암기] SSH 필수 권한 규칙**

```bash
# 1. SSH 디렉토리 권한 설정 (700: 소유자만 읽기/쓰기/실행)
chmod 700 ~/.ssh

# 2. authorized_keys 파일 권한 설정 (600: 소유자만 읽기/쓰기)
chmod 600 ~/.ssh/authorized_keys

# 3. 내 컴퓨터의 개인키 파일 권한 설정 (600: 소유자만 읽기/쓰기)
chmod 600 ~/.ssh/id_ed25519
```

---

## 4. 서버 SSH 보안 강화 설정 (`/etc/ssh/sshd_config`)

공개키 등록이 완료되면, **비밀번호 접속을 완전히 차단**하고 **Root 직접 접속을 금지**하여 서버를 요새화합니다.

```bash
# SSH 서버 설정 파일 수정
sudo vi /etc/ssh/sshd_config
```

### 필수 보안 변경 항목 Check List

```text
# 1. 비밀번호를 이용한 로그인 완전히 차단 (키 인증만 허용)
PasswordAuthentication no

# 2. root 계정으로 직접 SSH 접속 차단 (일반 계정 접속 후 sudo 이용)
PermitRootLogin no

# 3. 빈 비밀번호 로그인 차단
PermitEmptyPasswords no

# 4. (선택/실무 권장) 기본 22번 포트를 다른 포트로 변경 (예: 2222)
Port 2222
```

설정 변경 후 반드시 **SSH 서비스를 재시작**해야 적용됩니다:

```bash
# SSH 데몬 재시작
sudo systemctl restart sshd

# (주의!) 기존 SSH 접속 세션을 끊지 말고, 새로운 터미널 창을 열어 정상 접속되는지 테스트하세요!
```

---

## 5. SSH 접속 장애 트러블슈팅

### 5.1 `Permission denied (publickey)` 에러가 날 때
* **원인 1:** 서버의 `~/.ssh/authorized_keys`에 내 공개키가 올바르게 들어가지 않음.
* **원인 2:** 서버의 `~/.ssh` 디렉토리(700)나 `authorized_keys` 파일(600)의 **권한이 잘못됨**.
* **원인 3:** 내 개인키 파일 경로가 지정되지 않음 ➔ `ssh -i ~/.ssh/id_ed25519 user@server_ip`로 키 명시.

### 5.2 `WARNING: UNPROTECTED PRIVATE KEY FILE!` 에러가 날 때
* **원인:** 내 컴퓨터에 있는 개인키 파일의 권한이 너무 느슨함(`755` 또는 `777`).
* **해결:** `chmod 600 ~/.ssh/id_ed25519` 실행.

### 5.3 상세 로그 추적 디버깅 (`-v` 옵션)
SSH 접속이 왜 안 되는지 단계별로 확인하고 싶을 때 **`-v`** (Verbose) 옵션을 붙입니다.

```bash
# 접속 과정의 핸드셰이크 및 인증 실패 원인 상세 출력
ssh -v ubuntu@192.168.1.100

# 더 자세한 디버깅 로그 필요 시 -vv 또는 -vvv
```

---

## 6. SSH 핵심 요약표

| 구분 | 파일 / 명령어 | 설명 |
| :--- | :--- | :--- |
| **키 생성** | `ssh-keygen -t ed25519` | ED25519 알고리즘으로 공개키/개인키 쌍 생성 |
| **키 전송** | `ssh-copy-id user@IP` | 원격 서버에 내 공개키를 자동으로 등록 |
| **공개키 저장소** | `~/.ssh/authorized_keys` | 서버에 등록된 허용된 클라이언트 공개키 목록 |
| **권한 설정** | `chmod 700 ~/.ssh` <br> `chmod 600 authorized_keys` | SSH 접속 거부 방지를 위한 필수 권한 설정 |
| **보안 설정** | `/etc/ssh/sshd_config` | `PasswordAuthentication no`, `PermitRootLogin no` |
| **디버깅** | `ssh -v user@IP` | SSH 접속 실패 시 원인 추적 로그 출력 |
