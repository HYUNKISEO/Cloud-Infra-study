# 11일차: 시스템 백그라운드 서비스 및 로그 관리 (systemctl, journalctl)

> **학습 목표:** 리눅스의 최상위 프로세스인 `systemd` 구조를 이해하고, `systemctl`을 활용해 백그라운드 서비스를 제어하며, `journalctl`을 이용해 용량 폭주 없이 안전하게 실시간 장애 로그를 추적/관리하는 능력을 습득합니다.

---

## 1. systemd와 systemctl의 개념

* **systemd:** 리눅스 부팅 시 가장 먼저 실행되는 **PID 1번 최상위 부모 프로세스**이자 서버 전체를 총괄하는 **'지휘관(사장님)'**입니다.
* **systemctl:** 지휘관(`systemd`)에게 "Nginx 켜라", "상태 보고해라"라고 명령을 전달하는 **'무전기(리모컨)'**입니다.

```text
+-----------------------------------------------------------------------+
|                             systemd (PID 1 / 지휘관)                 |
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
                   [ systemctl / journalctl ] (무전기)
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
> * `restart`: 프로세스를 완전히 종료했다가 다시 켭니다. (짧은 순간 연결 끊김)
> * `reload`: 프로세스 종료 없이 설정 파일만 다시 읽어옵니다. (서비스 중단 없음, Nginx 설정 변경 시 권장)

---

### 2.2 부팅 시 자동 실행(Enable) 관리

```bash
# 1. 서버 재부팅 시 서비스 자동 실행 등록
sudo systemctl enable nginx

# 2. 부팅 시 자동 실행 해제
sudo systemctl disable nginx

# 3. 서비스 즉시 시작 + 자동 실행 등록을 동시에 처리 (치트키)
sudo systemctl enable --now nginx

# 4. 자동 실행 등록 여부만 확인
systemctl is-enabled nginx
```

---

### 2.3 [실무 팁] 장애 서비스 및 전체 목록 조회

```bash
# 1. 현재 에러(Failed) 상태로 죽어 있는 서비스만 필터링
systemctl failed

# 2. 현재 실행 중인 모든 서비스 목록 확인
systemctl list-units --type=service --state=running
```

---

## 3. journalctl을 활용한 로그 분석 및 용량 폭주 방지

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

# 5. 에러(Error) 등급 이상의 중요한 로그만 필터링
sudo journalctl -u nginx -p err
```

---

### 3.2 ⚠️ [필수 실무] 로그 영구 저장 & 용량 제한(Rotation) 설정

기본적으로 `journalctl` 로그는 RAM에만 보관되어 **재부팅 시 삭제**됩니다. 하지만 영구 저장만 설정하고 방치하면 **디스크가 꽉 차 서버가 다운**되므로 반드시 용량 제한을 세트로 설정합니다.

#### 1단계: 로그 영구 저장 디렉토리 생성
```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

#### 2단계: 로그 최대 용량 제한 설정 (`/etc/systemd/journald.conf`)
```bash
sudo vi /etc/systemd/journald.conf
```
```ini
[Journal]
SystemMaxUse=1G        # 로그 파일이 사용할 최대 디스크 용량 (1GB)
SystemMaxFileSize=100M # 로그 파일 1개당 최대 크기 (100MB)
MaxRetentionSec=1month # 로그 보관 최대 기간 (1개월)
```
```bash
# 설정 변경 후 적용
sudo systemctl restart systemd-journald
```

#### 3단계: 수동 로그 정리 명령어
```bash
# 용량 과다 시 최근 500MB만 남기고 옛날 로그 삭제
sudo journalctl --vacuum-size=500M
```

---

## 4. 나만의 백그라운드 서비스 만들기 (`systemd unit`)

### 4.1 서비스 파일 작성 예시 (`/etc/systemd/system/myapp.service`)

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

> 🛑 **실무에서 가장 많이 실수하는 3대 함정:**
> 1. **명령어 절대 경로 필수:** `ExecStart`에 `node`만 쓰면 동작 안 함 (`/usr/bin/node`처럼 전체 경로 입력)
> 2. **수정 후 `daemon-reload` 필수:** `.service` 파일을 수정했다면 반드시 `sudo systemctl daemon-reload`를 실행해야 반영됨
> 3. **보안 상 `root` 사용 자제:** `User=ubuntu`처럼 서비스 구동 권한을 일반 사용자 계정으로 지정 권장

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
