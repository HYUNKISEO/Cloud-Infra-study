# 🐧 15. 실무 장애 대응 시나리오 및 Phase 1 종합 점검

> 실무 인프라 운영에서 자주 발생하는 3가지 핵심 장애 상황을 통해 원인 추적 및 조치 프로세스(Troubleshooting Workflow)를 다지고 Phase 1을 최종 마감합니다.

---

## 🛠️ 표준 장애 대응 4단계 워크플로우

* **Step 1. 증상 확인 (Symptom):** 장애 발생 알람 및 접수된 이상 증상 파악
* **Step 2. 원인 진단 (Diagnosis):** 디스크, 프로세스, 네트워크, 시스템 로그 다각도 분석
* **Step 3. 문제 조치 (Resolution):** 서비스 재시작, 용량 확보, 프로세스 제어, 설정 수정
* **Step 4. 최종 검증 (Verification):** 포트/프로세스 상태 재확인 및 서비스 통신 테스트

---

## 🚨 실무 장애 대응 시나리오 3선

### 1. 디스크 용량 100% 초과로 인한 서비스 중단

> **상황:** DB 쓰기 작업 실패 및 애플리케이션 로그 생성 중단 현상 발생.

* **Step 1. 디스크 상태 및 대용량 파일 추적**
  ```bash
  # 1. 전체 디스크 사용량 점검
  df -h

  # 2. 용량 점유 상위 디렉토리/파일 추적 (/var/log 내 상위 5개)
  du -sh /var/log/* | sort -rh | head -n 5
  ```

* **Step 2. 안전한 용량 비우기 (파일 구조 유지)**
  ```bash
  # 파일 삭제(rm) 대신 내용만 0바이트로 초기화하여 파일 핸들 유지
  cat /dev/null > /var/log/nginx/access.log
  ```

* **Step 3. 유령 파일(Ghost File) 처리 및 서비스 정상화**
  ```bash
  # rm으로 지웠으나 프로세스가 잡고 있어 용량이 줄어들지 않는 파일 확인
  lsof | grep -i deleted

  # 해당 서비스 재로드하여 파일 핸들 해제 및 용량 완전 확보
  systemctl reload nginx
  ```

> 💡 **실무 노하우 & 재발 방지 (Troubleshooting & Prevention)**
> * **흔한 실수 방지:** 서비스 동작 중 `rm`으로 로그를 지우면 프로세스가 파일 핸들을 쥐고 있어 공간이 해제되지 않습니다. 용량이 줄지 않는다고 서버를 리부팅하는 실수를 피하려면 항상 `cat /dev/null > [파일명]`으로 초기화하세요.
> * **최종 검증 방법:** 조치 후 `df -h /var/log` 명령어로 Available 용량이 정상 확보되었는지 반드시 확인합니다.
> * **근본 재발 방지:** `logrotate` 스케줄러를 설정해 로그를 일자별로 자동 압축/삭제하고, 모니터링 툴(Prometheus 등)에 디스크 80% 이상 경고 알림을 등록합니다.

---

### 2. 웹 서버(Nginx/Apache) 먹통 및 접속 불량

> **상황:** 외부에서 웹사이트 접속 불가 (`curl: (7) Failed to connect`).

* **Step 1. 서비스 상태 및 에러 로그 확인**
  ```bash
  # 1. 데몬 실행 상태 확인
  systemctl status nginx

  # 2. systemd 상세 에러 로그 최근 50줄 확인
  journalctl -u nginx -n 50 --no-pager
  ```

* **Step 2. 포트 충돌 및 네트워크 점유 점검**
  ```bash
  # 80/443 포트를 다른 프로세스가 사용 중인지 확인
  ss -tulpn | grep -E ':80|:443'
  ```

* **Step 3. 설정 문법 검사 및 서비스 재시작**
  ```bash
  # 설정 파일 오타 검사
  nginx -t

  # 서비스 재시작
  systemctl restart nginx
  ```

> 💡 **실무 노하우 & 재발 방지 (Troubleshooting & Prevention)**
> * **흔한 실수 방지:** `Address already in use` 에러 발생 시 무작정 `kill -9`를 치기 전, 타팀 서비스나 중요 프로세스가 포트를 선점한 것은 아닌지 `ss -tulpn`으로 PID와 프로세스 이름을 먼저 확인해야 합니다.
> * **최종 검증 방법:** 조치 후 `curl -I http://localhost`로 `HTTP/1.1 200 OK` 응답 헤더가 정상 반환되는지 테스트합니다.
> * **근본 재발 방지:** systemd 서비스 파일(`/etc/systemd/system/nginx.service`)에 `Restart=always` 및 `RestartSec=5s` 옵션을 부여해 불의의 프로세스 다운 시 자동 복구되도록 설정합니다.

---

### 3. CPU 사용량 100% 과점 및 의문의 프로세스

> **상황:** 서버 반응 속도 저하 및 CPU 점유율 100% 지속.

* **Step 1. CPU 상위 점유 프로세스 PID 및 실행 경로 식별**
  ```bash
  # 실시간 프로세스 모니터링 (P 키로 CPU순 정렬)
  top

  # CPU 사용량 상위 5개 프로세스 정렬 출력
  ps aux --sort=-%cpu | head -n 6

  # 의심 프로세스(PID)의 실제 실행 바이너리 경로 확인
  ls -l /proc/<PID>/exe
  ```

* **Step 2. 프로세스 종료 및 스케줄러 점검**
  ```bash
  # 정상 종료 시도 후 불응 시 강제 종료
  kill -15 <PID>
  kill -9 <PID>

  # 악성 재실행 방지를 위한 크론탭 스케줄러 점검
  crontab -l
  cat /etc/crontab
  ```

> 💡 **실무 노하우 & 재발 방지 (Troubleshooting & Prevention)**
> * **흔한 실수 방지:** 의문의 프로세스는 이름을 정상 서비스처럼 속이는 경우가 많습니다. 이름만 보고 판단하지 말고 반드시 `ls -l /proc/<PID>/exe`로 실행 원천 경로(예: `/tmp` 여부)를 추적하세요.
> * **최종 검증 방법:** 조치 후 `top -b -n 1`로 CPU `Cpu(s) id`(Idle) 비율이 정상 회복되었는지 모니터링합니다.
> * **근본 재발 방지:** 임시 디렉토리인 `/tmp`에 `noexec` 마운트 옵션을 부여해 외부 스크립트 실행을 차단하고, `cgroups`로 단일 프로세스의 최대 CPU 점유율을 제한합니다.

---

## 🎓 Phase 1 종합 점검 체크리스트

| 구 분 | 핵심 명령어 / 개념 | 상태 |
| :---: | :--- | :---: |
| **파일/권한** | `ls -la`, `chmod 755/600`, `chown`, `umask` | ✅ 완료 |
| **텍스트/로그** | `grep -rnI`, `tail -f`, `journalctl -u` | ✅ 완료 |
| **프로세스/디스크** | `ps aux`, `top`, `kill -9`, `df -h`, `du -sh` | ✅ 완료 |
| **네트워크/SSH** | `ip a`, `ss -tulpn`, `curl -I`, `ssh-keygen`, `authorized_keys` | ✅ 완료 |
| **서비스/자동화** | `systemctl start/enable`, `crontab -e`, Bash `.sh` 기초 | ✅ 완료 |
