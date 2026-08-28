# 11일차: 시스템 백그라운드 서비스 및 로그 관리 (systemctl, journalctl)

> **학습 목표:** 리눅스의 핵심 메인 프로세스인 `systemd`의 구조를 이해하고, `systemctl`을 활용해 백그라운드 서비스를 제어하며, `journalctl`로 실시간 장애 로그를 추적 및 분석하는 능력을 습득합니다.

---

## 1. systemd와 systemctl의 개념

**systemd**는 리눅스 시스템이 부팅될 때 가장 먼저 실행되는 **PID 1번의 최상위 부모 프로세스(Init System)**입니다. 시스템의 모든 서비스, 데몬, 네트워크, 디바이스 등을 종합적으로 관리합니다.

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

## 2. systemctl 필수 제어 명령어

서비스 관리 시 가장 중요한 구별은 **"지금 실행 중인가?(start/stop)"**와 **"재부팅해도 자동으로 켜지는가?(enable/disable)"**의 차이를 이해하는 것입니다.

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

## 3. journalctl을 활용한 시스템/서비스 로그 분석

`systemd`는 제어하는 모든 서비스의 출력을 중앙 집중식 바이너리 로그(Journal)로 수집합니다. 이를 조회하는 전용 도구가 **`journalctl`**입니다.

### 3.1 실무에서 매일 쓰는 필수 옵션

```bash
# 1. 특정 서비스의 로그만 실시간 추적 (Tail F 기능 - 실무 활용 1위)
sudo journalctl -u nginx -f

# 2. 특정 서비스의 최근 N줄 로그만 보기
sudo journalctl -u nginx -n 50

# 3. 최근 발생한 상세 에러 로그 및 부팅 관련 에러 확인 (-x: 설명 추가, -e: 끝으로 이동)
sudo journalctl -xe

# 4. 특정 시간대 이후의 로그 필터링
sudo journalctl -u nginx --since "10 min ago"
sudo journalctl -u nginx --since "2026-08-28 09:00:00"

# 5. 에러(Error) 등급 이상의 중요한 로그만 필터링 (-p: priority)
sudo journalctl -u nginx -p err
```

---

## 4. [실무 꿀팁] 나만의 백그라운드 서비스 만들기 (`systemd unit`)

내가 만든 파이썬/Node.js/자바 애플리케이션을 `systemctl`로 관리하려면 **서비스 등록 파일(`.service`)**을 하나 작성하면 됩니다.

### 4.1 서비스 파일 작성 위치 및 작성 예시

```bash
# 서비스 파일 생성
sudo vi /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Custom Node.js Web Application
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
| **즉시 제어** | `systemctl start\|stop\|restart` | 서비스 즉시 구동, 중지, 재시작 |
| **무중단 적용** | `systemctl reload <service>` | 서비스 중단 없이 설정 파일 변경사항 반영 |
| **자동 실행** | `systemctl enable\|disable` | 서버 재부팅 시 서비스 자동 실행 설정/해제 |
| **실시간 로그** | `journalctl -u <service> -f` | 해당 서비스의 출력 로그를 실시간 타임라인으로 추적 |
| **상세 에러** | `journalctl -xe` | 서비스 구동 실패 시 시스템 원인 추적 디버깅 |
| **설정 반영** | `systemctl daemon-reload` | `/etc/systemd/system/` 아래 파일 수정 후 데몬 재로드 |
