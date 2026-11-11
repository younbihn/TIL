# 3-Tier 계층 간 접근 제어 (firewalld)

> 최초 작성: 2026-11-11 (1-4 3-Tier 설계와 서버 증설)

## 정책 표 (코드를 쓰기 전에 표부터)

| 출발 | 도착 | 포트 | 판정 | 이유 |
|---|---|---|---|---|
| 어디서나 | 웹 | 80 | 허용 | 사용자가 들어오는 입구 |
| 어디서나 | 모든 서버 | 22 | 허용 | Ansible, 운영 접속 |
| 웹 두 대 | 앱 | 8000 | 허용 | nginx 가 /api/ 를 넘기는 길 |
| 앱 | DB | 3306 | 허용 | 앱이 회원 정보를 읽고 쓰는 길 |
| 그 외 | 앱 | 8000 | 거부 | 앱은 웹을 거쳐서만 |
| 그 외 | DB | 3306 | 거부 | DB 는 절대 직접 노출하지 않는다 |
| **웹** | **DB** | 3306 | **거부** | 계층을 건너뛰는 호출 차단 |

- 마지막 줄이 설계 의도. 웹에서 DB 로 가는 통신은 애초에 일어나지 않지만 일부러 막는다. 급할 때 웹 서버에서 DB 를 바로 조회하고 싶어지는 날이 오는데, 통신이 되면 그렇게 하게 되고 그 순간 계층 구조가 무너진다. 필요한 것만 열고 나머지는 닫는 **최소 권한**을 네트워크에 적용한 것

## firewalld 가 새 연결을 판정하는 순서 (public zone)
1. 이미 맺어진 연결의 후속 패킷 → 통과 (응답이 막히지 않는 이유)
2. **rich rule** 에 맞는 것이 있으면 그 규칙대로
3. zone 에 허용된 service, port → 통과
4. 아무것도 안 맞으면 zone 기본 동작. public 은 **reject**

- "8000번을 웹 두 대에만"은 `--add-port=8000/tcp` 로 못 만든다 (모두에게 연다). **출발지 조건을 걸 수 있는 rich rule** 이 필요

## rich rule 한 줄
```
rule family="ipv4" source address="192.168.64.2/32" port port="8000" protocol="tcp" accept
```
- `source address="…/32"`: 출발지. `/32` 는 정확히 주소 하나, `/24` 면 256개 대역
- `port port="8000" protocol="tcp"`: 도착 포트. 앞의 `port` 는 요소 이름, `port=` 는 속성
- `accept`: 허용 (`reject`, `drop` 도 있음)
- YAML 에서는 `rich_rule: >-` 로 여러 줄을 공백으로 이어 한 줄로 만든다. 블록 안의 큰따옴표는 rich rule 문법의 것이라 바깥에 YAML 따옴표를 또 두르지 않는다
- 규칙에 넣을 IP 는 인벤토리에서 끌어온다:
  `groups['webservers'] | map('extract', hostvars, 'ansible_host') | list` → `['192.168.64.2', '192.168.64.3']`
  서버를 인벤토리에 추가하는 것만으로 규칙이 따라온다 (같은 값이 한 곳에만 있다는 원칙)

## 누가 언제 처리하나 (실행 위치 구분)
- 맥의 Ansible 은 `source address="192.168.64.2/32"` 라는 **글자**를 만들어 VM 에 넘길 뿐 (배포 시점)
- 패킷을 막고 통과시키는 것은 VM 의 커널(nftables)이 **패킷이 올 때마다** 한다 (요청 시점)

## 연결 시도로 검증하기
```bash
# 🖥️ 시도하는 VM 에서. bash 의 가짜 경로 /dev/tcp 로 TCP 연결만 해보고 닫는다
timeout 3 bash -c '</dev/tcp/192.168.64.5/3306'; echo "종료 코드 $?"
```
- `timeout 3`: 패킷이 조용히 버려지면(drop) 끝없이 기다리므로 3초 제한
- `$?`: 0 성공, 1 실패, 124 timeout

### 실패 메시지의 종류가 곧 진단 정보

| 메시지 | 의미 | 어디까지 갔나 |
|---|---|---|
| `No route to host` | 방화벽이 **거부(reject)** 하고 알려줬다 | 상대 방화벽에서 멈춤 |
| `Connection refused` | 방화벽은 통과, **그 포트에 듣는 프로그램이 없다** | 상대 커널까지 도달 |
| timeout (124) | 방화벽이 **조용히 버렸다(drop)** 거나 경로가 없다 | 알 수 없음 |
| 종료 코드 0 | 연결 성공 | 프로그램까지 도달 |

## 실제 측정 (2026-11-11 22:04경, db01 에 MySQL 설치 전)
```
# 🖥️ lab01 (웹) → db01:3306
bash: connect: 호스트로 갈 루트가 없음         (= No route to host)
종료코드 1

# 🖥️ app01 (앱) → db01:3306
bash: connect: 연결이 거부됨                   (= Connection refused)
종료코드 1

# 🖥️ db01 의 커널 로그 (log-denied=all 를 켠 상태)
filter_IN_public_REJECT: IN=enp0s1 SRC=192.168.64.2 DST=192.168.64.5 PROTO=TCP SPT=50226 DPT=3306 SYN
```
- lab01: 웹 → DB 차단이 동작. db01 로그에 출발지 `192.168.64.2`(lab01), 도착 포트 3306 의 SYN 이 REJECT 로 찍혔다
- app01: 방화벽을 통과해 db01 의 커널까지 도달. 거절은 **아직 MySQL 을 설치하지 않아서**. 정책이 맞게 들어갔다는 증거
- **통하는 것보다 막힌 것을 확인하는 쪽이 설계를 증명한다.** "앱에서 DB 가 된다"는 경계가 없어도 성립하지만, "웹에서 DB 가 안 된다"는 경계가 있어야만 성립한다
- MySQL 설치 후 기대값: app01 → 종료코드 0, lab01 → 계속 No route to host (설치 후 재확인은 아직 하지 않았다)

## 거부 로그 켜고 끄기
```bash
# 🖥️ db01
sudo firewall-cmd --set-log-denied=all
sudo journalctl -k -f | grep -i reject     # 다른 창에서 시도를 반복하면 한 줄씩 찍힌다
sudo firewall-cmd --set-log-denied=off     # 확인이 끝나면 끈다. 켜 두면 로그가 넘친다
```

## 확인의 한계
- `ansible all -m ping` 전부 pong: SSH + Python 실행만 확인. 서버끼리 통신은 모른다 (ping 모듈은 네트워크 ping 이 아님)
- `--list-rich-rules` 에 규칙이 보임: 저장, 적용은 확인되지만 의도대로 막고 통과시키는지는 별개 → 허용 경로와 차단 경로를 둘 다 실제로 시도해야 한다
