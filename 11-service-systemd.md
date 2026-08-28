# 11일차: 시스템 백그라운드 서비스 및 로그 관리 (systemctl, journalctl)

> **학습 목표:** 리눅스의 최상위 프로세스인 `systemd` 구조를 이해하고, `systemctl`을 활용해 백그라운드 서비스를 제어하며, `journalctl`을 이용해 용량 폭주 없이 안전하게 실시간 장애 로그를 추적/관리하는 능력을 습득합니다.

---

## 1. systemd와 systemctl의 개념

**systemd**는 리눅스 시스템 부팅 시 가장 먼저 실행되는 **PID 1번 최상위 부모 프로세스(Init System)**입니다. 시스템의 모든 서비스, 데몬, 네트워크, 디바이스 등을 종합 관리합니다.

`systemctl`은 이러한 `systemd`에게 명령을 내려 **백그라운드 서비스(Service Unit)의 상태를 조회하고 제어하는 컨트롤러 명령어**입니다.

```text
+-----------------------------------------------------------------------+
|                             systemd (PID 1)                           |
+-----------------------------------------------------------------------+
           |                                         |
           v                                         v
 +-------------------+                     +-------------------+
 |  Nginx (Service)  |                     |  MySQL (Service)  |
 +-------------------+                     +-------------------+
           ^                                         ^
           |                                         |
           +-----------------+-----------------------+
                             |
                   [ systemctl / journalctl ]
```

---

## 2. systemctl 필수 제어 및 상태 조회 명령어

### 2.1 서비스 상태 확인 및 즉시 제어

```bash
# 1. 서비스 현재 상태 확인 (가장 자주 사용)
sudo systemctl status nginx

# 2. 서비스 즉시 시작 / 중지
sudo systemctl start nginx
sudo systemctl stop nginx

# 3. 서비스 재시작 (설정 변경 적용 또는 장애 조치 시)
sudo systemctl restart nginx

# 4. 서비스 중단 없이 설정 파일만 새로고침 (Reload)
sudo systemctl reload nginx
```

> 💡 **`restart` vs `reload` 차이:**
> * `restart`: 프로세스를 완전히 종료했다가 다시 켭니다. (짧은 순간 연결 끊김 발생)
> * `reload`: 프로세스를 종료하지 않고 설정 파일만 다시 읽어옵니다. (서비스 중단 없음, Nginx 설정 변경 시 권장)

---

### 2.2 부팅 시 자동 실행(Enable) 관리

```bash
# 1. 서버 재부팅 시 서비스 자동 실행 등록
sudo systemctl enable nginx

# 2. 부팅 시 자동 실행 해제
sudo systemctl disable nginx

# 3. 서비스 즉시 시작 + 자동 실행 등록을 동시에 처리
sudo systemctl enable --now nginx

# 4. 자동 실행 등록 여부만 확인
systemctl is-enabled nginx
```

---

### 2.3 [실무 팁] 장애 서비스 및 전체 목록 조회

서비스명을 잘 모르거나 에러 난 서비스를 한눈에 파악할 때 유용합니다.

```bash
# 1. 현재 에러(Failed) 상태로 죽어 있는 서비스만 필터링
systemctl failed

# 2. 현재 실행 중인 모든 서비스 목록 확인
systemctl list-units --type=service --state=running
```

---

## 3. journalctl을 활용한 로그 분석 및 용량 폭주 방지

`systemd`는 제어하는 모든 서비스의 출력을 중앙 집중식 바이너리 로그(Journal)로 수집합니다. 이를 조회하는 전용 도구가 **`journalctl`**입니다.

### 3.1 실무 필수 로그 조회 명령어

```bash
# 1. 특정 서비스 로그 실시간 추적 (Tail -f 기능)
sudo journalctl -u nginx -f

# 2. 특정 서비스의 최근 N줄 로그만 보기
sudo journalctl -u nginx -n 50

# 3. 상세 에러 로그 및 시스템 원인 추적 (-x: 설명 추가, -e: 끝으로 이동)
sudo journalctl -xe

# 4. 특정 시간대 이후의 로그 필터링
sudo journalctl -u nginx --since "10 min ago"
sudo journalctl -u nginx --since "2026-08-28 09:00:00"

# 5. 에러(Error) 등급 이상의 중요한 로그만 필터링
sudo journalctl -u nginx -p err
```

---

### 3.2 ⚠️ [필수 실무 보안] 로그 영구 저장 & 용량 제한(Rotation) 설정

기본적으로 `journalctl` 로그는 RAM(메모리)에 보관되므로 **서버를 재부팅하면 장애 원인이 담긴 이전 로그가 전부 삭제**됩니다. 하지만 영구 저장만 설정하고 방치하면 디스크가 가득 차서 서버가 마비됩니다.

#### 1단계: 로그 영구 저장 디렉토리 생성
```bash
# 로그 영구 저장 폴더 생성 후 데몬 재시작
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

#### 2단계: 로그 최대 용량 제한 설정 (`journald.conf`)
로그가 디스크를 과도하게 점유하지 않도록 **최대 용량**을 제한합니다.

```bash
sudo vi /etc/systemd/journald.conf
```

```ini
[Journal]
# 로그 파일이 사용할 최대 디스크 용량 제한 (예: 1GB)
SystemMaxUse=1G

# 로그 파일 1개당 최대 크기 (예: 100MB)
SystemMaxFileSize=100M

# 로그 보관 최대 기간 (예: 1개월)
MaxRetentionSec=1month
```

```bash
# 설정 변경 후 적용
sudo systemctl restart systemd-journald
```

#### 3단계: 수동 로그 정리 명령어
```bash
# 용량이 넘쳤을 때 최근 500MB만 남기고 옛날 로그 삭제
sudo journalctl --vacuum-size=500M

# 2주 이전의 오래된 로그 수동 삭제
sudo journalctl --vacuum-time=2weeks
```

---

## 4. 나만의 백그라운드 서비스 만들기 (`systemd unit`)

내가 만든 애플리케이션(Node.js, Python, Java 등)을 `systemctl`로 관리하려면 **서비스 등록 파일(`.service`)**을 하나 작성하면 됩니다.

### 4.1 서비스 파일 작성 위치 및 예시

```bash
sudo vi /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Custom Web Application
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/app
ExecStart=/usr/bin/node /home/ubuntu/app/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 4.2 서비스 등록 및 적용

```bash
# 1. systemd 설정 새로고침 (서비스 파일 추가/수정 후 필수!)
sudo systemctl daemon-reload

# 2. 등록한 내 서비스 시작 및 자동 실행 처리
sudo systemctl start myapp
sudo systemctl enable myapp

# 3. 실행 상태 확인
sudo systemctl status myapp
```

---

## 5. systemctl & journalctl 핵심 요약표

| 구분 | 명령어 | 설명 |
| :--- | :--- | :--- |
| **상태 확인** | `systemctl status <service>` | 서비스 실행 여부, PID, 최근 로그 요약 출력 |
| **죽은 서비스** | `systemctl failed` | 현재 장애 발생으로 멈춘 서비스 목록 출력 |
| **즉시 제어** | `systemctl start\|stop\|restart` | 서비스 즉시 구동, 중지, 재시작 |
| **무중단 적용** | `systemctl reload <service>` | 서비스 중단 없이 설정 파일 변경사항 반영 |
| **자동 실행** | `systemctl enable\|disable` | 서버 재부팅 시 서비스 자동 실행 설정/해제 |
| **실시간 로그** | `journalctl -u <service> -f` | 해당 서비스의 출력 로그를 실시간 타임라인으로 추적 |
| **상세 에러** | `journalctl -xe` | 서비스 구동 실패 시 시스템 원인 추적 디버깅 |
| **용량 정리** | `journalctl --vacuum-size=500M` | 지정한 용량만 남기고 오래된 로그 정리 |
| **설정 반영** | `systemctl daemon-reload` | `/etc/systemd/system/` 아래 파일 수정 후 데몬 재로드 |
