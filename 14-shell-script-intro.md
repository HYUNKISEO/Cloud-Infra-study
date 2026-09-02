# 14일차: 반복 업무를 줄여주는 쉘 스크립트(Shell Script) 작성

> **학습 목표:** 쉘 스크립트의 기초 구조(Shebang, 변수, 매개변수)와 제어문(조건문 `if`, 반복문 `for`)을 익히고, 서버 백업 및 로그 정리를 자동화하는 실무형 `.sh` 스크립트를 직접 작성합니다.

---

## 1. 쉘 스크립트 기초 및 작성 규칙

쉘 스크립트는 리눅스 터미널에서 실행하는 명령어들을 파일로 모아둔 실행 파일입니다.

### 1.1 기본 작성 규칙 3요소
1. **쉬뱅(Shebang, `#!/bin/bash`):** 파일 첫 줄에 작성하여 이 파일이 bash 쉘로 해석됨을 명시합니다.
2. **실행 권한 부여:** 작성 후 반드시 `chmod +x 스크립트명.sh` 권한을 부여해야 실행할 수 있습니다.
3. **절대 경로 사용:** 크론탭이나 백그라운드 실행 시 환경변수 오작동을 방지하기 위해 명령어 및 파일 경로는 절대 경로 사용을 권장합니다.

```bash
#!/bin/bash

# 간단한 출력 예시
echo "현재 날짜 및 시간: $(date)"
echo "현재 접속 계정: $(whoami)"
```

---

## 2. 실무 필수 특수 변수 (Special Variables)

스크립트 실행 시 외부에서 인자(Argument)를 전달받거나, 이전 명령어의 성공 여부를 판단할 때 사용하는 핵심 변수입니다.

| 변수 | 의미 | 실무 활용 예시 |
| :--- | :--- | :--- |
| **`$0`** | 실행된 스크립트의 파일명 | 스크립트 사용법(Usage) 안내 출력 시 |
| **`$1`, `$2`** | 첫 번째, 두 번째 전달 인자 | `./backup.sh /var/www html` (위치 매개변수) |
| **`$#`** | 전달된 인자의 총 개수 | 필수 인자 입력 여부 검증 (`if [ $# -lt 1 ]`) |
| **`$?`** | 바로 직전 명령어의 **종료 상태 코드** | `0` = 성공, `0 이외` = 에러 발생 (트러블슈팅 핵심) |

```bash
#!/bin/bash

# 인자 전달 테스트 스크립트
echo "스크립트명: $0"
echo "첫 번째 인자: $1"
echo "전달된 인자 개수: $#"
```

---

## 3. 조건문(if) 및 파일/디렉토리 상태 검사

서버 자동화 스크립트에서는 특정 디렉토리가 존재하는지, 파일이 비어있지 않은지 검사하는 구문이 필수적입니다.

### 3.1 조건문 기본 문법 (대괄호 양옆 공백 필수!)
```bash
if [ 조건식 ]; then
    # 조건이 참(True)일 때 실행
elif [ 조건식2 ]; then
    # 조건2가 참일 때 실행
else
    # 거짓(False)일 때 실행
fi
```

### 3.2 실무 자주 쓰는 파일 상태 검사 옵션
* **`[ -d 경로 ]`**: 해당 경로에 **디렉토리**가 존재하는지 확인
* **`[ -f 경로 ]`**: 해당 경로에 **일반 파일**이 존재하는지 확인
* **`[ -z "$문자열" ]`**: 문자열의 길이가 **0(빈 값)**인지 확인

```bash
#!/bin/bash

TARGET_DIR="/var/log/backup"

# 디렉토리가 없으면 자동 생성
if [ ! -d "$TARGET_DIR" ]; then
    echo "백업 디렉토리가 없어 생성합니다: $TARGET_DIR"
    mkdir -p "$TARGET_DIR"
fi
```

---

## 4. 반복문(for)을 활용한 다중 대상 처리

여러 디렉토리나 파일을 순회하며 동일한 작업을 반복 수행할 때 사용합니다.

```bash
#!/bin/bash

# 주요 로그 파일 압축 목록 순회
LOG_FILES=("/var/log/syslog" "/var/log/auth.log" "/var/log/nginx/error.log")

for FILE in "${LOG_FILES[@]}"; do
    if [ -f "$FILE" ]; then
        echo "[정상] 검사 대상 파일 존재: $FILE"
    else
        echo "[경고] 파일이 존재하지 않음: $FILE"
    fi
done
```

---

## 5. 🛑 [실무 종합] 자동 백업 및 오래된 로그 정리 스크립트

실무에서 크론탭(`crontab`)에 등록하여 매일 새벽마다 작동시키는 백업 자동화 완성형 스크립트입니다.

```bash
#!/bin/bash
# 파일명: /opt/scripts/auto_backup.sh
# 설명: 웹 로그 압축 백업 및 7일 지난 백업 파일 자동 삭제

# 1. 변수 정의 (절대 경로 및 날짜 포맷)
SOURCE_DIR="/var/log/nginx"
BACKUP_DIR="/backup/nginx"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/nginx_log_$DATE.tar.gz"

# 2. 백업 디렉토리 점검
if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR"
fi

# 3. 로그 파일 압축 수행
echo "[$(date)] 백업을 시작합니다..." >> /var/log/backup.log
tar -czf "$BACKUP_FILE" "$SOURCE_DIR" > /dev/null 2>&1

# 4. 압축 성공 여부 검증 ($? 활용)
if [ $? -eq 0 ]; then
    echo "[SUCCESS] 백업 성공: $BACKUP_FILE" >> /var/log/backup.log
else
    echo "[ERROR] 백업 실패! 원인을 확인하세요." >> /var/log/backup.log
    exit 1
fi

# 5. 7일 이상 지난 오래된 백업 파일 자동 삭제 (디스크 용량 확보)
find "$BACKUP_DIR" -type f -name "*.tar.gz" -mtime +7 -delete
echo "[$(date)] 7일 초과 백업 파일 정리 완료." >> /var/log/backup.log
```

---

## 6. 쉘 스크립트 핵심 요약표

| 구분 | 주요 구문 / 옵션 | 설명 및 실무 팁 |
| :--- | :--- | :--- |
| **Shebang** | `#!/bin/bash` | 스크립트 최상단 필수 명시 |
| **실행 권한** | `chmod +x script.sh` | 권한 미부여 시 `Permission denied` 에러 발생 |
| **상태 코드** | `$?` | 바로 전 명령어 성공 시 `0`, 실패 시 `1~255` 반환 |
| **디렉토리 검사** | `if [ -d "$DIR" ]` | 대괄호`[`와 조건식 사이에 **반드시 공백** 필요 |
| **종료 처리** | `exit 0` / `exit 1` | 스크립트 정상/비정상 종료 상태 전달 |
