# 08. 디스크 용량 분석 및 파일시스템 마운트 (df, du, mount)

> **학습일:** Day 8 <br>
> **학습 목표:** 리눅스의 블록 장치 및 파일시스템 마운트 구조를 이해하고, `df`와 `du`를 활용해 디스크 용량 부족 원인을 분석하며, 새로운 디스크를 포맷하고 `/etc/fstab`을 통해 영구 마운트하는 실무 기술을 익힙니다.

---

## 1. 디스크 및 파일시스템 기본 개념 (Block Device & Mount)

리눅스는 Windows처럼 `C:\`, `D:\` 같은 드라이브 문자를 사용하지 않으며, **모든 저장 장치(디스크)를 하나의 단일 디렉토리 트리(`/`)에 연결(Mount)하여 사용**합니다.

* **핵심 용어 정리:**
  * **블록 장치 (Block Device):** `/dev/sda`, `/dev/nvme0n1`, `/dev/vda` 등 저장 장치를 의미하는 파일
  * **파티션 (Partition):** 하나의 물리 디스크를 논리적으로 분할한 영역 (`/dev/sda1`, `/dev/sda2` 등)
  * **파일시스템 (FileSystem):** 디스크에 데이터를 조직화하여 저장하는 방식 (Linux 표준: `ext4`, `xfs`)
  * **마운트 (Mount):** 파티션(장치)을 리눅스의 특정 디렉토리(예: `/mnt`, `/data`)와 연결하는 작업

> 💡 **왜 디스크 마운트 개념이 중요할까요?**
> * AWS 등 클라우드 환경에서 EBS(추가 디스크)를 붙여도, 포맷 후 특정 디렉토리에 마운트하기 전까지는 서버에서 사용할 수 없습니다.

---

## 2. 전체 디스크 용량 및 아이노드(Inode) 조회 (`df`)

파일시스템 단위로 전체 디스크 사용량, 남은 용량, 마운트 지점을 확인합니다.

* **주요 옵션:**
  * `df -h` (Human-readable): 사람이 읽기 편한 단위(M, G, T)로 용량 표시 (가장 흔히 사용)
  * `df -i` (Inode): 용량이 아닌 **아이노드(파일 개수) 사용량** 조회
  * `df -T`: 각 파티션의 **파일시스템 타입(`ext4`, `xfs` 등)**을 함께 표시

```bash
# 1. 사람이 읽기 쉬운 용량 단위(GB, TB)로 디스크 사용량 확인
df -h

# 2. 파일 개수(Inode) 남은 수량 확인 (파일 개수 초과 장애 분석 시 필수)
df -i

# 3. 파일시스템 타입(ext4, xfs)을 포함하여 확인
df -hT

# 4. 특정 경로가 속한 파티션의 디스크 용량만 확인
df -h /var/log
```

* **`df -h` 실제 터미널 출력 예시:**

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        29G   12G   17G  42% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
/dev/sda15      105M  6.1M   99M   6% /boot/efi
/dev/sdb1       100G   45G   55G  45% /data
```

> 🛑 **실무 트러블슈팅 (용량이 남았는데 파일 생성이 안 될 때):**
> * `df -h`로 보면 용량이 50% 이상 남았는데 `No space left on device` 에러가 난다면, `df -i`로 **Inode 사용량이 100%인지 확인**해야 합니다.
> * 소용량 로그/세션 파일이 수십만 개 생성되면 용량보다 파일 개수 한도(Inode)가 먼저 넘칠 수 있습니다.

---

## 3. 폴더 및 파일별 용량 상세 분석 (`du` & `ncdu`)

`df`로 디스크 전체가 차오른 것을 확인했다면, **어느 폴더가 용량을 많이 잡아먹고 있는지** 세부 원인을 추적합니다.

### 3.1 `du` 명령어를 활용한 용량 분석
* **주요 옵션:**
  * `-s` (Summary): 하위 항목의 전체 합계만 표시
  * `-h` (Human-readable): 용량을 K, M, G 단위로 표시
  * `--max-depth=N`: 조회할 디렉토리 깊이 제한

```bash
# 1. 현재 디렉토리 내 폴더별 용량 합계 확인 (depth 1단계만 표시)
du -h --max-depth=1 .

# 2. /var 디렉토리 하위 용량을 확인하고 용량이 큰 순서대로 정렬
du -h --max-depth=1 /var 2>/dev/null | sort -hr

# 3. 특정 단일 디렉토리 전체 용량 확인
du -sh /var/log
```

> 💡 **`df`와 `du`의 용량 불일치 이슈 (`Deleted` 파일 점유):**
> * `rm`으로 대용량 로그 파일을 지웠는데 `du`에서는 용량이 줄었지만 `df`에서는 여전히 100%로 나올 때가 있습니다.
> * 이는 **실행 중인 프로세스가 삭제된 파일의 FD(File Descriptor)를 아직 잡고 있기 때문**입니다. `sudo lsof | grep deleted`로 해당 프로세스를 찾아 재시작하면 디스크 용량이 즉시 확보됩니다.

### 3.2 [추천 도구] 대화형 용량 분석기 `ncdu`
실무에서 복잡한 디렉토리 트리를 탐색하며 대용량 파일을 찾을 때 `du`보다 훨씬 빠르고 편리한 텍스트 기반 GUI 도구입니다.

```bash
# 설치 (Ubuntu / Debian)
sudo apt update && sudo apt install ncdu -y

# /var 디렉토리 용량 대화형 분석
ncdu /var
```

---

## 4. 디스크 장치 및 파티션 구조 확인 (`lsblk`, `fdisk`)

서버에 물리적으로/논리적으로 연결된 블록 장치 목록과 파티션 구조를 확인합니다.

```bash
# 1. 블록 장치 목록 및 마운트 트리 구조를 한눈에 조회
lsblk

# 2. 상세 파일시스템 타입(ext4, xfs) 및 UUID 함께 조회
lsblk -f

# 3. 디스크 파티션 상세 정보 조회 (관리자 권한 필요)
sudo fdisk -l
```

---

## 5. 파일시스템 생성(포맷) 및 마운트 (`mkfs`, `mount`, `umount`)

새로 추가된 디스크(`/dev/sdb1`)를 포맷하고 특정 디렉토리에 마운트하여 사용할 수 있게 만듭니다.

```bash
# 1. 디스크 파티션을 Linux 표준 ext4 또는 xfs 파일시스템으로 포맷 (데이터 전체 삭제 주의)
sudo mkfs.ext4 /dev/sdb1
# 또는 XFS 파일시스템 사용 시: sudo mkfs.xfs /dev/sdb1

# 2. 마운트 포인트 디렉토리 생성
sudo mkdir -p /mnt/data

# 3. 디스크 파티션을 디렉토리에 수동 마운트
sudo mount /dev/sdb1 /mnt/data

# 4. 마운트 상태 확인
df -h | grep /mnt/data

# 5. 마운트 해제 (언마운트) - 해당 디렉토리를 사용 중인 프로세스가 없어야 함
sudo umount /mnt/data
```

---

## 6. 부팅 시 자동 마운트 설정 (`/etc/fstab`)

수동 `mount`는 서버를 재부팅하면 해제됩니다. **서버가 다시 켜져도 자동으로 마운트**되도록 `/etc/fstab` 파일에 등록합니다.

### Step 1. 디스크 UUID 확인
장치명(`/dev/sdb1`)은 서버 부팅 순서에 따라 변경될 수 있으므로 고유식별자인 **UUID**를 사용하는 것이 정석입니다.

```bash
# 장치의 UUID 확인
sudo blkid /dev/sdb1
# 출력 예시: /dev/sdb1: UUID="a1b2c3d4-e5f6-7890-abcd-1234567890ab" TYPE="ext4"
```

### Step 2. `/etc/fstab` 파일 편집

```bash
sudo vi /etc/fstab
```

맨 아래에 아래 형식으로 한 줄을 추가합니다.

```text
# [UUID 또는 장치명] [마운트포인트] [파일시스템타입] [옵션] [dump] [pass]
UUID=a1b2c3d4-e5f6-7890-abcd-1234567890ab /mnt/data ext4 defaults,nofail 0 2
```

* **fstab 설정 값 의미:**
  * `defaults`: 기본 마운트 옵션 적용 (rw, suid, dev, exec, auto, nouser, async)
  * `nofail`: **[클라우드 필수]** 해당 디스크가 분리되거나 로드되지 않아도 부팅 중단(Emergency Mode) 없이 정상 진입하게 하는 안전 옵션
  * `dump (0)`: 백업 가능 여부 (0: 사용 안 함)
  * `pass (2)`: 부팅 시 파일시스템 점검 순서 (0: 미점검, 1: 루트`/`, 2: 기타 파티션)

🛑 **[실무 최우선 절대 경고] `/etc/fstab` 작성 후 검증 필수!**
* `/etc/fstab` 파일에 오타가 있으면 **서버가 부팅되지 않고 멈추는(Emergency Mode) 치명적인 장애**가 발생합니다.
* 파일 수정 후 서버를 재부팅하기 전에 반드시 다음 명령어로 오타가 없는지 마운트 테스트를 수행해야 합니다.

```bash
# /etc/fstab에 설정된 모든 마운트를 즉시 재테스트 (오타가 있으면 에러 출력됨)
sudo mount -a
```

---

## 7. [실무 팁] 클라우드 디스크 용량 증설 후 파일시스템 확장

AWS EBS 등 클라우드에서 볼륨 용량을 10GB → 20GB로 늘렸을 때, 리눅스 OS가 늘어난 용량을 즉시 인식하도록 파일시스템을 확장하는 방법입니다.

```bash
# 1. 파티션 크기 확장 (예: /dev/xvda 장치의 파티션 1번)
sudo growpart /dev/xvda 1

# 2. 파일시스템 타입에 따른 용량 확장 적용
# Case A: ext4 파일시스템인 경우
sudo resize2fs /dev/xvda1

# Case B: xfs 파일시스템인 경우 (마운트 경로를 인자로 전달)
sudo xfs_growfs /
```

---

## 8. Day 8 핵심 명령어 & 옵션 통합 요약

| 명령어 | 주요 역할 | 필수/핵심 옵션 | 실무 활용 및 주의사항 |
| :--- | :--- | :---: | :--- |
| **`df`** | 전체 파일시스템 용량 조회 | `-h`, `-i`, `-T` | `df -h`(용량 확인), `df -i`(파일 개수 한도 초과 점검) |
| **`du`** | 폴더/파일 세부 용량 측정 | `-sh`, `--max-depth=N` | `du -h --max-depth=1 /var` 형태로 용량 주범 추적 |
| **`ncdu`** | 대화형 디스크 용량 분석 | - | 터미널 GUI 기반으로 대용량 폴더/파일 빠르게 삭제 관리 |
| **`lsblk`** | 블록 장치 및 마운트 트리 조회 | `-f` | UUID 및 파일시스템 타입(`ext4`/`xfs`) 동시 확인 |
| **`mkfs`** | 파티션 파일시스템 포맷 | `.ext4`, `.xfs` | 포맷 시 기존 데이터 파괴되므로 대상 파티션 재확인 |
| **`mount`** | 장치를 디렉토리에 연결 | `-a` | `mount -a`로 `/etc/fstab` 오타 사전 검증 필수 |
| **`umount`** | 마운트 해제 | - | 디렉토리 사용 중(`target is busy`)일 경우 프로세스 종료 후 해제 |
| **`blkid`** | 장치 고유 UUID 조회 | - | `/etc/fstab` 등록용 UUID 추출 시 사용 |

---

## 9. Day 8 실무 핵심 디스크 관리 CLI 팁

| 구분 | 주요 대상 | 역할 및 사용법 | 실무 활용 예시 |
| :--- | :---: | :--- | :--- |
| **대용량 파일 정렬** | **`du` + `sort`** | 가장 용량이 큰 폴더 Top 10 추출 | `du -ah /var 2>/dev/null \| sort -rh \| head -n 10` |
| **지워진 파일 점유** | **`lsof`** | `rm` 삭제 후에도 용량 반환 안 될 때 | `sudo lsof \| grep deleted` 프로세스 조회 후 재시작 |
| **fstab 안전망** | **`nofail`** | 외장/추가 디스크 부팅 실패 방지 옵션 | `/etc/fstab` 옵션란에 `defaults,nofail` 추가 시 디스크 없어도 부팅 진행 |
| **마운트 점유 해제** | **`fuser` / `lsof`** | `umount` 시 target is busy 에러 해결 | `sudo fuser -mv /mnt/data`로 점유 중인 PID 확인 |
| **Inode 과점유 삭제** | **`find` + `delete`** | 소용량 파일 수십만 개 일괄 삭제 | `find /var/spool/clientmqueue -type f -delete` |
