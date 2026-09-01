# 13일차: 서버 로그 분석을 통한 장애 트러블슈팅 (/var/log)

> **학습 목표:** 리눅스 환경에서 시스템 및 애플리케이션 로그가 저장되는 `/var/log` 디렉토리의 구조를 이해하고, 장애 발생 시 원인을 빠르게 규명하기 위한 로그 추적 명령어 및 `logrotate` 매커니즘을 습득합니다.

---

## 1. /var/log 디렉토리의 역할과 핵심 파일 구조

리눅스의 모든 핵심 시스템 이벤트, 보안 기록, 서비스 동작 결과는 `/var/log` 이하에 파일 형태로 누적됩니다. 

```text
/var/log/
├── syslog (또는 messages)  <-- OS 전체 시스템 종합 이벤트 로그
├── auth.log (또는 secure) <-- 사용자 로그인, SSH 접속, sudo 권한 사용 기록
├── dmesg                  <-- 커널(Kernel) 부팅 및 하드웨어/메모리 관련 로그
├── nginx/ (또는 apache2/)  <-- 웹 서버 접근(access) 및 에러(error) 로그
├── mysql/ (또는 postgresql/)<-- 데이터베이스 쿼리 에러 및 데몬 로그
└── logrotate.conf         <-- 로그 용량 폭주 방지를 위한 순환 설정 파일
```

---

## 2. OS 계열별 주요 로그 파일 비교

OS 종류(Ubuntu 계열 vs CentOS/RHEL 계열)에 따라 주요 로그 파일의 이름이 상이하므로 실무에서 이를 구분하는 것이 중요합니다.

| 용도 | Ubuntu / Debian 계열 | CentOS / RHEL / Rocky 계열 | 주요 기록 내용 |
| :--- | :--- | :--- | :--- |
| **시스템 종합 로그** | `/var/log/syslog` | `/var/log/messages` | OS 전반의 일반 동작, 커널 경고, 서비스 상태 |
| **보안 및 인증 로그** | `/var/log/auth.log` | `/var/log/secure` | SSH 접속 시도(성공/실패), sudo 명령 실행 이력 |
| **커널 메세지 로그** | `/var/log/dmesg` | `/var/log/dmesg` | 하드웨어 인식, 메모리 부족(OOM Killer) 발생 기록 |
| **부팅 로그** | `/var/log/boot.log` | `/var/log/boot.log` | 시스템 부팅 과정에서의 데몬 시작 성공/실패 여부 |

---

## 3. 실무 로그 탐색 및 분석 핵심 명령어 파이프라인

수 백 MB ~ 수 GB에 달하는 로그 파일 속에서 필요한 에러만 빠르게 찾아내는 실무 필수 Command Line 패턴입니다.

### ① 실시간 로그 추적 (`tail -f`)
```bash
# Nginx 에러 로그 실시간 감시
tail -f /var/log/nginx/error.log

# 최근 100줄을 먼저 출력한 뒤 실시간 추적 (-n + -f)
tail -n 100 -f /var/log/syslog
```

### ② 특정 키워드(ERROR, FATAL) 검색 (`grep`)
```bash
# auth.log에서 SSH 로그인 실패(Failed password) 이력만 추출
grep "Failed password" /var/log/auth.log

# 대소문자 구분 없이(i) 'error' 단어가 포함된 행 수(c) 계산
grep -ic "error" /var/log/syslog

# 키워드 검색 시 앞뒤 3줄 맥락 함께 출력 (-B: Before, -A: After)
grep -B 3 -A 3 "FATAL" /var/log/syslog
```

### ③ 대용량 로그 검색 (`less +F`)
* `cat`으로 수 GB 짜리 로그를 열면 메모리가 마비됩니다. `less` 명령어를 사용하면 메모리 과부하 없이 빠르게 검색할 수 있습니다.
```bash
less /var/log/syslog
# less 접속 후 Shift + G (맨 아래 이동), /ERROR (키워드 검색), Shift + F (실시간 모드 전환)
```

---

## 4. 로그 디스크 폭주 방지: logrotate 매커니즘 및 용량 정리

로그 파일이 계속 쌓여 디스크 용량이 100%가 되면 서버가 다운됩니다. 이를 방지하기 위해 리눅스는 `logrotate` 서비스를 이용해 로그를 주기적으로 압축하고 오래된 로그를 삭제합니다.

### 4.1 로그 순환 파일 이름 규칙
```text
syslog        <-- 현재 실시간으로 기록 중인 파일
syslog.1      <-- 지난 주(또는 어제) 기록된 1차 백업 파일
syslog.2.gz   <-- gzip으로 압축되어 용량이 줄어든 과거 로그 파일
```

### 4.2 압축된 로그 파일 검색 및 전체 열람 (`zgrep`, `zless`)
`.gz`로 압축된 과거 로그는 해제하지 않고 전용 명령어로 즉시 확인이 가능합니다.
```bash
# 1. 압축된 syslog.2.gz 안에서 "CRON" 키워드만 검색
zgrep "CRON" /var/log/syslog.2.gz

# 2. 압축된 로그 파일 전체를 less처럼 편하게 열어보기 (q로 종료)
zless /var/log/syslog.2.gz
```

### 4.3 [실무 팁] journald 로그 용량 폭주 시 긴급 정리 (`journalctl --vacuum`)
디스크가 100% 찼을 때 systemd 저널 로그를 긴급 삭제하여 공간을 확보합니다.
```bash
# journal 로그 용량을 최신 500MB만 남기고 삭제
sudo journalctl --vacuum-size=500M

# 7일 이상 된 journal 로그 일괄 삭제
sudo journalctl --vacuum-time=7d
```

---

## 5. 🛑 실무 3대 장애 발생 시 로그 분석 가이드 (Troubleshooting)

### 장애 1: 시스템 메모리가 부족하여 프로세스가 갑자기 강제 종료됨 (OOM Killer)
* **원인:** 메모리 고갈 시 커널이 가장 메모리를 많이 먹는 프로세스를 강제로 죽임 (`Out of Memory`).
* **로그 확인 명령어:**
  ```bash
  dmesg | grep -i "oom"
  # 또는
  grep -i "killed process" /var/log/syslog
  ```

### 장애 2: 외부 해커의 SSH 무차별 대입 공격(Brute Force) 의심
* **원인:** 특정 IP에서 지속적으로 잘못된 비밀번호로 접속을 시도함.
* **로그 확인 명령어:**
  ```bash
  # 로그인 실패 IP별 시도 횟수 상위 집계
  grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head -n 10
  ```

### 장애 3: 웹 서버 접속 불가 (500 Internal Server Error)
* **원인:** 백엔드 애플리케이션 연동 오류 또는 파일 권한 문제.
* **로그 확인 명령어:**
  ```bash
  tail -n 50 /var/log/nginx/error.log
  ```

---

## 6. /var/log 분석 핵심 요약표

| 구분 | Ubuntu 파일 경로 | CentOS 파일 경로 | 핵심 트러블슈팅 활용법 |
| :--- | :--- | :--- | :--- |
| **시스템 에러** | `/var/log/syslog` | `/var/log/messages` | OS 장애, 데몬 비정상 종료 원인 규명 |
| **인증/보안** | `/var/log/auth.log` | `/var/log/secure` | 무단 접근 시도, sudo 권한 남용 추적 |
| **메모리/커널** | `dmesg` 명령어 | `dmesg` 명령어 | OOM Killer 동작 여부 및 하드웨어 에러 확인 |
| **압축 파일 검색/열람**| `zgrep`, `zless` | `zgrep`, `zless` | `logrotate`로 압축된 과거 로그 검색 및 열람 |
| **용량 긴급 정리**| `journalctl --vacuum-size` | `journalctl --vacuum-size` | 저널 로그 정리로 디스크 용량 확보 |
