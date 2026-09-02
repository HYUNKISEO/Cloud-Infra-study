# 09. 서버 IP와 네트워크 트러블슈팅 (ip, ping, ss, netstat, curl, lsof, traceroute)

> **학습일:** Day 9 <br>
> **학습 목표:** 서버의 IP 및 네트워크 인터페이스를 확인하고, 포트(Port) 점유 상태 및 네트워크 도달 가능성을 진단하며, `curl`을 통해 HTTP API 요청 및 Nginx/App 통신 장애를 해결하는 실무 기술을 익힙니다.

---

## 1. 네트워크 기본 개념 (웹개발 & 클라우드 관점)

웹 서비스가 가동되려면 서버는 **IP 주소**로 위치를 나타내고, 실행 중인 애플리케이션(Node.js, Spring Boot, Nginx 등)은 **포트(Port)**로 구분됩니다.

* **핵심 용어 정리:**
  * **Public IP (공인 IP):** 인터넷망을 통해 외부 유저나 클라이언트가 접속할 수 있는 IP (AWS Elastic IP 등)
  * **Private IP (사설 IP):** 외부 인터넷에는 노출되지 않고, 내부 네트워크(AWS VPC 등)끼리 통신하는 IP (예: `10.0.x.x`, `192.168.x.x`)
  * **Loopback (`127.0.0.1` / `localhost`):** 서버 자기 자신을 가리키는 내부 전용 IP
  * **All Interfaces (`0.0.0.0`):** 서버가 가진 모든 네트워크 인터페이스(외부 IP + 내부 IP)를 통해 들어오는 요청을 받겠다는 의미
  * **Port (포트):** 하나의 IP 안에서 어떤 프로세스(앱)로 데이터를 보낼지 구별하는 번호 (0~65535)
    * `80`: HTTP (기본 웹) / `443`: HTTPS (보안 웹)
    * `22`: SSH (서버 원격 접속)
    * `3306`: MySQL / `5432`: PostgreSQL
    * `8080`, `3000`: 백엔드 애플리케이션 개발 표준 포트

> 🛑 **[웹개발/클라우드 단골 장애] 바인딩 주소 실수 (`127.0.0.1` vs `0.0.0.0`)**
> * Node.js, Spring, FastAPI 등 앱을 실행할 때 Listen 주소를 `127.0.0.1`로 설정하면 **서버 내부에서만 접속 가능**하고 외부 인터넷 유저는 접속할 수 없습니다.
> * 외부 유저나 Load Balancer의 요청을 받아야 하는 웹 애플리케이션은 반드시 **`0.0.0.0:8080`** 형태로 바인딩하여 실행해야 합니다.

---

## 2. 서버 IP 및 네트워크 인터페이스 확인 (`ip`)

과거 리눅스에서 쓰이던 `ifconfig`는 최신 리눅스 표준에서 보류(Deprecated)되었으며, 현재는 **`ip` 명령어**를 표준으로 사용합니다.

```bash
# 1. 서버에 할당된 모든 네트워크 인터페이스 및 IP 주소 확인 (가장 흔히 사용)
ip a
# 또는 상세 표기: ip addr show

# 2. 외부 인터넷으로 나가는 기본 게이트웨이(Routing Table) 확인
ip route
```

* **`ip a` 실제 터미널 출력 예시 및 해석:**

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP
    inet 172.31.16.45/20 brd 172.31.31.255 scope global dynamic eth0
```
* `lo`: 서버 내부 루프백 인터페이스 (`127.0.0.1`)
* `eth0` (또는 `ens33`, `enp0s3`): 메인 랜카드/네트워크 인터페이스 (`172.31.16.45` -> 이 서버의 사설 IP)

---

## 3. 네트워크 도달성 및 경로 추적 (`ping`, `traceroute`, `nc`)

상대방 서버나 내부 서비스에 네트워크 신호가 정상적으로 도달하는지, 어디서 막히는지 확인합니다.

### 3.1 `ping` (ICMP 프로토콜 레벨 테스트) & `traceroute` (경로 추적)
상대방 IP/도메인까지 네트워크 패킷이 왕복할 수 있는지 확인하고, 도중 끊기는 구간을 찾아냅니다.

```bash
# 1. google.com으로 4번만 패킷을 보내 연결 상태 및 응답 시간(ms) 확인
ping -c 4 google.com

# 2. [인프라/네트워크 전담팀 영역] 내 서버부터 목적지까지 거치는 라우터 구간(Hop) 추적
traceroute 8.8.8.8
# 또는 traceroute가 없을 때: tracepath 8.8.8.8
```

> 🛑 **클라우드(AWS) 주의사항 및 traceroute 활용:**
> * AWS EC2 보안그룹(Security Group)이나 방화벽 설정에서 **ICMP 프로토콜을 차단**해 두면 `ping` 타임아웃이 납니다.
> * `traceroute`는 내 컴퓨터 ➔ 통신사 ➔ 게이트웨이 ➔ AWS 인프라 구간 중 **어느 단계 라우터에서 패킷이 병목되거나 병목/지연이 생기는지** 홉(Hop) 단위로 점검할 때 사용합니다.

### 3.2 `nc` (Netcat) - 특정 IP + 포트 열림 확인 (★ 클라우드 필수)
`ping`은 특정 포트(예: 8080포트)가 열렸는지는 확인하지 못합니다. **방화벽이나 보안그룹에서 해당 포트를 허용했는지** 확인하려면 `nc`를 사용합니다.

```bash
# 상대방 IP(192.168.1.50)의 8080 포트가 열려있는지 확인 (타임아웃 3초 설정)
nc -zv -w 3 192.168.1.50 8080
```
* **출력 예시별 진단 방법:**
  * `Succeeded!`: 네트워크 및 보안그룹/방화벽 허용 완료.
  * `Connection refused`: 서버에는 도달했으나 해당 포트에서 실행 중인 앱(프로세스)이 떠있지 않음.
  * `Operation timed out`: 방화벽/AWS 보안그룹에서 해당 포트 접근을 차단하고 있음.

---

## 4. 포트 점유 및 소켓 상태 조회 (`ss`, `lsof`, `netstat`)

서버 내부에서 **어떤 포트가 열려 있는지**, **포트를 차지하고 있는 프로세스가 무엇인지** 확인합니다.

```bash
# 1. 현재 LISTEN(대기) 중인 모든 TCP/UDP 포트와 관련 프로세스(PID/이름) 조회
sudo ss -tulpn

# 2. [초고속 직관적 명령어] 특정 포트(예: 3000번)를 점유 중인 프로세스 즉시 확인
sudo lsof -i :3000
```

* **`ss -tulpn` 옵션 의미:**
  * `-t` (TCP): TCP 프로토콜 소켓 조회
  * `-u` (UDP): UDP 프로토콜 소켓 조회
  * `-l` (Listening): 접속 대기 중인 포트만 조회
  * `-p` (Process): 해당 포트를 점유 중인 프로세스 PID 및 이름 표시 (sudo 권한 필요)
  * `-n` (Numeric): 도메인/서비스 이름 대신 숫자 포트 번호로 표기 (`http` -> `80`)

> 💡 **구형 레거시 명령어 매핑 노트:**
> * 과거 자료나 블로그에서는 `netstat -tulpn` 또는 `netstat -anp`를 사용했습니다. 최신 리눅스 OS에서는 `ss`가 훨씬 빠르고 표준이므로 **`ss -tulpn`**을 사용하는 것이 권장됩니다.

### 4.1 웹개발 단골 에러 해결 (`Address already in use`)
새로운 앱(예: Node.js)을 3000번 포트로 실행하려는데 이미 기존 프로세스가 3000번을 쓰고 있으면 발생하는 에러입니다.

```bash
# 1. 3000번 포트를 점유 중인 프로세스 PID 확인 (lsof 방식 사용 시)
sudo lsof -i :3000

# 2. 해당 프로세스 PID(예: 1234) 확인 후 안전 종료(kill -15) 또는 강제 종료(kill -9)
sudo kill -15 1234
```

---

## 5. HTTP API 호출 및 웹 트러블슈팅 (`curl`)

웹개발자 및 DevOps 엔지니어가 CLI 환경에서 **API를 호출하고, HTTP 상태 코드 및 헤더를 검증하는 핵심 도구**입니다.

### 5.1 기본 GET 요청 및 응답 헤더 확인

```bash
# 1. 간단한 GET 요청으로 HTML/JSON 응답 본문 출력
curl [https://api.example.com/health](https://api.example.com/health)

# 2. [개발/테스트 필수] SSL 인증서 검증 무시하고 강제 접속 (-k 또는 --insecure)
curl -k [https://dev-api.local/health](https://dev-api.local/health)

# 3. HTTP 리다이렉트(301/302) 발생 시 최종 목적지까지 추적하여 이동 (-L)
curl -L [http://example.com](http://example.com)

# 4. 응답 헤더(HTTP 상태코드 200/404/500, Server, Content-Type 등)만 확인 (-I)
curl -I [https://api.example.com](https://api.example.com)

# 5. [트러블슈팅 전용] 요청/응답 전체 과정 및 SSL/TLS 핸드셰이크 상세 출력 (-v)
curl -v [https://api.example.com](https://api.example.com)
```

### 5.2 POST/PUT 데이터 전송 (API 테스트)

```bash
# JSON 데이터를 포함한 POST 요청 테스트
curl -X POST [https://api.example.com/users](https://api.example.com/users) \
  -H "Content-Type: application/json" \
  -d '{"name": "devuser", "role": "admin"}'
```

> 💡 **Nginx 리버스 프록시 트러블슈팅 실무 활용:**
> * 웹 브라우저에서 `502 Bad Gateway`가 뜰 때, 서버 내부에서 `curl http://localhost:8080`을 날려봅니다.
> * `localhost:8080`에서 응답이 잘 온다면 **백엔드 앱은 정상이고 Nginx 설정 문제**라는 것을 1초 만에 밝혀낼 수 있습니다.

---

## 6. DNS 도메인 해석 진단 (`dig`, `nslookup`, `/etc/hosts`)

도메인(예: `myapi.com`)을 쳤을 때 올바른 IP 주소로 연결되는지 확인하고, 도메인 연결 전 강제로 매핑을 테스트합니다.

```bash
# 1. 특정 도메인의 DNS A 레코드(IP) 상세 조회
dig myapi.com

# 2. 간략하게 IP 매핑 정보만 빠르게 확인
nslookup myapi.com
```

### 6.1 로컬 DNS 강제 매핑 (`/etc/hosts`)
DNS 서버 설정이 완료되기 전이거나, 개발 환경에서 특정 도메인을 내 테스트 서버 IP로 강제 연결하여 검증할 때 사용합니다.

```bash
# /etc/hosts 파일에 테스트용 IP 및 도메인 추가
sudo vi /etc/hosts

# 파일 하단에 매핑 정보 작성 예시:
# [서버 IP]        [연결할 도메인]
# 192.168.1.100   myapi.local
```

---

## 7. Day 9 핵심 명령어 요약표 & 실무 트러블슈팅 가이드

| 명령어 / 조합 | 주요 목적 | 실무 핵심 활용 시나리오 |
| :--- | :--- | :--- |
| **`ip a`** | 서버 IP 주소 확인 | 내 서버의 사설(Private) IP 파악 시 사용 |
| **`ping [IP/도메인]`** | 기본 네트워크 도달성 확인 | 인터넷 연결 상태 확인 (단, AWS ICMP 차단 주의) |
| **`traceroute [IP]`** | 네트워크 라우팅 경로 추적 | 어느 구간(Hop) 라우터에서 패킷 지연/손실이 발생하는지 진단 |
| **`nc -zv [IP] [PORT]`** | 특정 포트 접근 가능 여부 | **AWS 보안그룹/방화벽이 닫혔는지** 진단하는 1순위 도구 |
| **`sudo lsof -i :[PORT]`**| 특정 포트 점유 프로세스 검색 | 포트 충돌 시 해당 포트를 쓰는 PID를 초고속 직관적 추적 |
| **`sudo ss -tulpn`** | 열려 있는 모든 포트 & PID 조회 | **Address already in use** 포트 충돌 전체 현황 파악 |
| **`curl -k [URL]`** | SSL 인증서 검증 무시 호출 | 테스트/개발 서버의 자체 발급 SSL 인증서 에러 우회 |
| **`curl -v [URL]`** | HTTP 요청/응답 과정 상세 분석 | API 상태 코드(500, 502, 403) 및 SSL 인증서 문제 분석 |
| **`curl -I [URL]`** | HTTP 헤더만 빠르게 확인 | 웹 서버 작동 여부 및 캐시/CORS 헤더 점검 |
| **`dig [도메인]`** | DNS IP 매핑 조회 | 도메인 연결 설정 후 IP 반영 여부 확인 |
| **`/etc/hosts`** | 로컬 DNS 강제 매핑 파일 | DNS 등록 전 개발용 도메인을 원하는 IP로 로컬 테스트 |
