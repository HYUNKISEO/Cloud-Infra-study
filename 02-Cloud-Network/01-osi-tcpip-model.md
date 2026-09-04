# 🌐 [Phase 2 - Day 01] 네트워크의 뼈대: OSI 7계층 & TCP/IP 4계층 모델

* **학습일**: Phase 2 - Day 01
* **목표**: OSI 7계층과 TCP/IP 4계층 모델의 구조 및 데이터 캡슐화 원리를 이해하고, 실무에서 통신 장애 발생 시 하위 계층부터 차례대로 점검하는 **Bottom-Up 트러블슈팅 체계**를 습득한다.

---

## 1. 네트워크 표준 모델의 개요

### 1.1 통신 모델의 존재 이유
네트워크 통신은 수많은 제조사의 하드웨어와 다양한 소프트웨어가 데이터를 주고받는 복잡한 과정입니다. 이를 단계별로 표준화(모듈화)하여, **특정 단계에 문제가 생겨도 해당 계층만 독립적으로 수정 및 트러블슈팅**할 수 있도록 만든 기준입니다.

### 1.2 OSI 7계층 vs TCP/IP 4계층 비교

| OSI 7계층 | TCP/IP 4계층 | PDU (데이터 단위) | 주요 역할 및 대표 프로토콜 / 장비 |
| :--- | :--- | :--- | :--- |
| **L7 Application** (응용) | **Application**<br>(응용 계층) | **Data** | 사용자 인터페이스, **HTTP, HTTPS, SSH, DNS, FTP** |
| **L6 Presentation** (표현) | | **Data** | 데이터 암호화/복호화, 압축 (**TLS/SSL**, JPEG) |
| **L5 Session** (세션) | | **Data** | 통신 세션 유지 및 관리 (**RPC**, NetBIOS) |
| **L4 Transport** (전송) | **Transport**<br>(전송 계층) | **Segment** | 신뢰성 있는 전송, 포트 제어 (**TCP, UDP**, L4 스위치) |
| **L3 Network** (네트워크) | **Internet**<br>(인터넷 계층) | **Packet** | 최적 경로 라우팅, IP 패킷 전달 (**IP, ICMP**, 라우터, L3 스위치) |
| **L2 Data Link** (데이터링크) | **Network Access**<br>(네트워크 접근 계층) | **Frame** | 동일 네트워크 내 물리전송 (**Ethernet, MAC 주소**, L2 스위치) |
| **L1 Physical** (물리) | | **Bit** | 0과 1의 전기/광신호 변환 (케이블, 허브, 리피터) |

> 💡 **PDU (Protocol Data Unit)**: 각 계층에서 처리되는 데이터의 단위입니다. 동일한 데이터라도 계층을 거치며 붙는 헤더에 따라 부르는 이름이 달라집니다.

---

## 2. 데이터 송수신 원리: 캡슐화 & 역캡슐화

### 2.1 캡슐화 (Encapsulation - 송신 측)
상위 계층(L7)에서 작성된 데이터가 하위 계층으로 내려가면서 각 계층에 필요한 **제어 정보(Header)**가 붙는 과정입니다.

```text
[사용자 데이터] (L7 Application)
       │
       ▼  + L4 Header (송신/수신 Port)
[ L4 Segment ]
       │
       ▼  + L3 Header (송신/수신 IP)
[ L3 Packet ]
       │
       ▼  + L2 Header (송신/수신 MAC) + Trailer (FCS 에러검출)
[ L2 Frame ]
       │
       ▼  전기/광 신호 변환 (01010110...)
[ L1 Physical ]  ──(물리 케이블 전송)──▶
```

### 2.2 역캡슐화 (Decapsulation - 수신 측)
물리적 신호(Bit)를 받아 상위 계층으로 올라가면서 **각 계층의 Header를 검증 후 제거**하여 최종 데이터만 애플리케이션에 전달합니다.

* **L2 계층**: MAC 주소가 내 것인지 확인 ➔ L2 Header 제거
* **L3 계층**: IP 주소가 내 것인지 확인 ➔ L3 Header 제거
* **L4 계층**: 포트 번호를 확인하여 실행 중인 해당 프로세스로 데이터 전달 ➔ L4 Header 제거

---

## 3. 실무 엔지니어의 계층별 트러블슈팅 (Bottom-Up)

서버 통신 장애 발생 시, 가장 하위 계층(L1)의 물리적 상태부터 상위 계층(L7)으로 올라가며 문제를 격리하는 것이 인프라 엔지니어의 표준 트러블슈팅 방식입니다.

```text
[L7 점검] curl / 웹 응답 코드 확인 (200, 502 등)
   ▲
[L4 점검] nc / 포트 개방 및 방화벽 상태 확인
   ▲
[L3 점검] ping / IP 통신 및 라우팅 경로 확인
   ▲
[L1/L2 점검] ip link / NIC 카드의 물리적 UP 상태 확인
```

---

## 4. 실무 필수 CLI 검증 명령어 모음

### 4.1 L1 / L2 계층 점검: `ip link`
네트워크 인터페이스 카드(NIC)의 물리적 연결 및 활성화 상태를 확인합니다.

```bash
$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
```
* **점검 포인트**: 상태값에 `UP` 및 `LOWER_UP`이 명시되어 있어야 물리적으로 정상 연결된 상태입니다.

### 4.2 L3 계층 점검: `ping` & `ip route`
IP 네트워크 통신이 가능한지 ICMP 패킷을 전송하여 확인합니다.

```bash
# 외부 IP(예: Google DNS)로 ICMP 패킷 4개 전송
$ ping -c 4 8.8.8.8

# 기본 게이트웨이(Routing Table) 경로 확인
$ ip route
default via 192.168.1.1 dev eth0 proto dhcp src 192.168.1.100
```
* **점검 포인트**: `Destination Host Unreachable`은 라우팅 문제, `Request Timeout`은 방화벽 패킷 차단일 확률이 높습니다.

### 4.3 L4 계층 점검: `nc` (Netcat) & `ss`
목적지 서버의 특정 포트(L4)가 열려 있는지, 내 서버 내부에서 해당 포트가 정상 바인딩 상태인지 검증합니다.

```bash
# 대상 IP의 80번 포트가 오픈되어 있는지 확인 (Timeout 3초 설정)
$ nc -zv -w 3 192.168.1.50 80
Connection to 192.168.1.50 80 port [tcp/http] succeeded!

# 내 서버에서 현재 Listening 중인 포트 및 프로세스 확인
$ ss -tulpn
Netid  State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port  Process
tcp    LISTEN  0       128      0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=892))
tcp    LISTEN  0       511      0.0.0.0:80           0.0.0.0:*          users:(("nginx",pid=1204))
```

### 4.4 L7 계층 점검: `curl`
HTTP/HTTPS 요청을 보내 애플리케이션의 최종 응답 상태 코드 및 헤더 정보를 확인합니다.

```bash
# HTTP 헤더 및 상태 코드만 상세 출력 (-I: Head만 조회, -v: 상세보기)
$ curl -I [https://google.com](https://google.com)
HTTP/2 200
date: Fri, 04 Sep 2026 05:30:00 GMT
content-type: text/html; charset=ISO-8859-1
```

---

## 5. 실무 장애 대응 시나리오

### 💡 장애 상황: "클라이언트가 웹 서비스(Port 80)에 접속할 수 없다고 할 때"

1. **L1/L2 점검**: 서버의 NIC가 다운되었는지 확인 (`ip link`).
2. **L3 점검**: 외부 망과 통신이 가능한지 확인 (`ping 8.8.8.8`).
3. **L4 점검 (내부)**: Web Server(Nginx 등) 프로세스가 포트를 잡고 수신 대기 중인지 확인 (`ss -tulpn | grep 80`).
4. **L4 점검 (외부)**: 방화벽(AWS SG 등)에서 80 포트가 막혔는지 외부 접속 테스트 (`nc -zv <서버IP> 80`).
5. **L7 점검**: 로컬에서 웹 서비스가 정상 응답을 반환하는지 확인 (`curl -I http://localhost`).
