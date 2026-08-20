# 06. 패키지 매니저를 통한 프로그램 설치와 삭제 (apt / yum)

> **학습일:** Day 6 <br>
> **학습 목표:** 리눅스 계열별 패키지 매니저(`apt`, `yum`/`dnf`)의 기본 동작 원리와 저장소(Repository) 개념을 이해하고, 프로그램의 검색·설치·업데이트·삭제 및 의존성 관리 기술을 익힙니다.

---

## 1. 패키지 매니저 개념과 저장소(Repository) 이해

리눅스에서 소프트웨어는 소스 코드를 직접 빌드하기보다, 필요한 라이브러리와 실행 파일이 압축된 **'패키지'** 형태로 관리합니다.

* **패키지 매니저의 역할:**
  * **의존성(Dependency) 자동 해결:** 특정 프로그램 실행에 필요한 다른 라이브러리들을 자동으로 추적하여 함께 설치합니다.
  * **중앙 저장소(Repository) 연동:** 검증된 원격 서버(저장소)에서 안전하게 패키지를 다운로드합니다.
* **리눅스 계열별 패키지 시스템 구분:**
  * **Debian / Ubuntu 계열:** `.deb` 패키지 포맷 ➡️ **`apt`** (과거 `apt-get`) 사용
  * **RHEL / CentOS / Rocky 계열:** `.rpm` 패키지 포맷 ➡️ **`yum`** / **`dnf`** 사용

> 💡 **왜 패키지 설치/삭제 시 `sudo`가 필수일까요?**
> * 프로그램 설치 시 시스템 공용 디렉터리인 `/usr/bin`, `/etc`, `/var` 등에 파일이 생성·수정됩니다.
> * 이러한 디렉터리는 `root` 소유이므로 일반 계정에서 패키지를 관리하려면 반드시 `sudo`를 통해 관리자 권한을 차용해야 합니다.

---

## 2. Debian/Ubuntu 계열 패키지 관리 (`apt`)

Ubuntu 등 우분투 계열에서 사용하는 가장 대표적인 패키지 관리 도구입니다.

* **주요 명령어 및 흐름:**
  * `apt update`: 원격 저장소의 최신 패키지 **목록(인덱스)**을 업데이트 (실제 프로그램 설치/업그레이드는 안 됨)
  * `apt upgrade`: 현재 시스템에 설치된 패키지들을 최신 버전으로 **일괄 업그레이드**
  * `apt search [키워드]`: 저장소에서 패키지 검색
  * `apt install [패키지명]`: 패키지 설치 (`-y` 옵션으로 확인 절차 자동 승인)
  * `apt remove` vs `apt purge`: 패키지 삭제 (설정 파일 남김 vs 설정 파일까지 완전 삭제)
  * `apt autoremove`: 더 이상 필요 없는 의존성 패키지 일괄 정리

```bash
# 1. 패키지 설치 전 반드시 저장소 목록 최신화
sudo apt update

# 2. 패키지 검색 (예: nginx 웹 서버 검색)
apt search nginx

# 3. 패키지 설치 (대화형 확인 프롬프트 자동 승인: -y)
sudo apt install -y nginx curl git

# 4. 패키지 삭제 (설정 파일은 유지)
sudo apt remove nginx

# 5. 패키지 완전 삭제 (설정 파일까지 깔끔하게 제거)
sudo apt purge nginx

# 6. 사용하지 않는 불필요한 의존성 패키지 정리
sudo apt autoremove
```

> 🛑 **실무 주의사항 (`apt update` vs `apt upgrade`):** 
> * `apt update`는 단지 **"무슨 패키지가 최신으로 나왔나 목록만 확인"**하는 명령입니다.
> * 실제 프로그램을 최신으로 바꾸려면 `apt update` 후 `apt upgrade`를 실행해야 합니다.

---

## 3. RHEL/CentOS/Rocky 계열 패키지 관리 (`yum` / `dnf`)

RedHat 계열에서 사용하는 패키지 관리 도구로, 최근 버전(CentOS 8+, Rocky Linux)에서는 `yum`의 개선판인 `dnf`가 기본 내장되어 있으며 사용법은 거의 동일합니다.

* **주요 명령어 및 흐름:**
  * `yum check-update`: 업데이트 가능한 패키지 목록 확인
  * `yum search [키워드]`: 저장소에서 패키지 검색
  * `yum info [패키지명]`: 패키지 상세 정보 확인
  * `yum install [패키지명]`: 패키지 설치
  * `yum update [패키지명]`: 특정 패키지 또는 전체 시스템 업그레이드
  * `yum remove [패키지명]`: 패키지 삭제

```bash
# 1. 업데이트 가능한 패키지 목록 확인
yum check-update

# 2. 패키지 검색 및 정보 확인
yum search htop
yum info htop

# 3. 패키지 설치 (-y 옵션으로 자동 승인)
sudo yum install -y htop wget

# 4. 특정 패키지 업데이트
sudo yum update htop

# 5. 패키지 삭제
sudo yum remove htop
```

> 💡 **`yum`과 `dnf` 관계:** `dnf`는 `yum`의 메모리 사용량과 성능을 개선한 차세대 패키지 매니저입니다. RHEL/Rocky 최신 버전에서는 `yum` 명령어를 입력해도 내부적으로 `dnf`로 자동 연결됩니다.

---

## 4. 패키지 저장소(Repository) 파일 위치 및 설정

패키지 매니저가 어느 서버에서 프로그램을 다운로드할지 결정하는 설정 파일 위치입니다.

* **Ubuntu / Debian:** `/etc/apt/sources.list` 및 `/etc/apt/sources.list.d/`
* **RHEL / CentOS / Rocky:** `/etc/yum.repos.d/*.repo`

```bash
# 1. Ubuntu 저장소 설정 파일 확인
cat /etc/apt/sources.list

# 2. CentOS/Rocky 저장소 설정 디렉터리 목록 확인
ls -l /etc/yum.repos.d/
```

> 💡 **실무 팁 (EPEL 저장소):** CentOS/Rocky 기본 저장소에 없는 유용한 패키지(예: `htop`, `nginx` 등)를 설치할 때는 확장 패키지 저장소인 **EPEL(Extra Packages for Enterprise Linux)**을 먼저 설치해 주어야 합니다.
> * 실행 예시: `sudo yum install -y epel-release`

---

## 5. 실무 트러블슈팅 & 락(Lock) 에러 해결법

패키지 설치 중 가장 흔하게 만나는 에러는 **'다른 프로세스가 패키지 매니저를 이미 사용 중인 경우(Lock 에러)'**입니다.

* **발생 상황:** 백그라운드에서 자동 업데이트가 실행 중이거나 이전 설치가 비정상 종료되었을 때 발생
* **에러 메시지 예시:** `Could not get lock /var/lib/dpkg/lock-frontend`

```bash
# 1. 현재 apt 또는 yum 프로세스가 동작 중인지 확인
ps aux | grep -i apt

# 2. 프로세스가 점유 중이라면 안전하게 종료되길 기다리거나 프로세스 종료
sudo kill -9 [PID]

# 3. 비정상 종료로 락 파일이 남아있을 때 잠금 파일 제거 (최후의 수단)
sudo rm /var/lib/apt/lists/lock
sudo rm /var/lib/dpkg/lock-frontend
```

---

## 6. Day 6 핵심 명령어 & 옵션 통합 요약

| 구분 | Ubuntu / Debian (`apt`) | RHEL / CentOS (`yum` / `dnf`) | 역할 및 특징 |
| :--- | :--- | :--- | :--- |
| **목록 갱신** | `sudo apt update` | `yum check-update` | 원격 저장소 패키지 정보 업데이트 |
| **패키지 검색** | `apt search [키워드]` | `yum search [키워드]` | 저장소 내 패키지 이름/설명 검색 |
| **패키지 설치** | `sudo apt install -y [이름]` | `sudo yum install -y [이름]` | 패키지 및 의존성 자동 설치 |
| **패키지 삭제** | `sudo apt remove [이름]` | `sudo yum remove [이름]` | 패키지 제거 (기본 설정 유지) |
| **완전 삭제** | `sudo apt purge [이름]` | - | 패키지 및 관련 설정 파일까지 완전 제거 |
| **시스템 전체 업데이트**| `sudo apt upgrade` | `sudo yum update` | 설치된 전체 패키지 최신화 |
| **의존성 정리** | `sudo apt autoremove` | `sudo yum autoremove` | 불필요해진 잔여 의존성 파일 삭제 |

---

## 7. Day 6 실무 핵심 파일 & 패키지 관리 CLI 팁

| 구분 | 주요 대상 | 역할 및 사용법 | 실무 활용 예시 |
| :--- | :---: | :--- | :--- |
| **저장소 파일** | **`/etc/apt/sources.list`** | Ubuntu 패키지 다운로드 미러 서버 주소 | 카카오/미러성균관대 등 한국 서버로 변경 시 속도 향상 |
| **저장소 디렉터리**| **`/etc/yum.repos.d/`** | RHEL/CentOS 저장소 설정 파일(.repo) 모음 | 외부 공식 저장소(Docker, MySQL 등) 추가 시 사용 |
| **확장 저장소** | **`epel-release`** | RedHat 계열 공식 추가 패키지 모음집 | `sudo yum install -y epel-release` 사전 설치 |
| **설치 확인** | **`dpkg -l` / `rpm -qa`** | 현재 시스템에 로컬로 설치된 전체 패키지 출력 | `dpkg -l \| grep nginx`로 설치 여부 체킹 |
