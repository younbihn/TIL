# firewalld

## zone — 신뢰 수준 단위의 규칙 묶음
네트워크 인터페이스를 zone에 배정하면 그 인터페이스의 트래픽에 해당 zone의 규칙이
적용된다. NIC이 여러 개일 때 외부망과 내부 관리망에 다른 정책을 주기 위한 구조다.

| zone | 성격 |
|---|---|
| drop | 모든 입력을 응답 없이 폐기. 가장 엄격 |
| block | 거부하되 거부 응답을 전송 |
| public | 기본값. 명시적으로 허용한 것만 통과 |
| internal / home / work | 내부망. 기본 허용 서비스가 더 많음 |
| trusted | 전체 허용 (사실상 방화벽 해제) |
| dmz | 외부에 노출되는 서버용. 제한적 허용 |
| external | 외부망. masquerade(NAT)가 기본 활성 |
| nm-shared | NetworkManager 연결 공유용. 자동 생성 |

## 조회
```bash
firewall-cmd --get-zones             # 사용 가능한 zone
firewall-cmd --get-default-zone      # 기본 zone
firewall-cmd --get-active-zones      # 인터페이스가 붙은 zone만
firewall-cmd --list-all              # 기본 zone 전체 설정
firewall-cmd --get-services          # 사전 정의 서비스 목록
firewall-cmd --info-service=ssh      # 해당 서비스의 포트 확인
```

### --list-all 출력 읽기
| 항목 | 의미 |
|---|---|
| target | 규칙에 걸리지 않은 패킷의 처리. default=거부 / ACCEPT / DROP |
| interfaces | 이 zone에 배정된 인터페이스 |
| sources | IP 대역으로 배정된 출처 (인터페이스 대신 출처로 zone 선택 가능) |
| services | 허용된 서비스 이름 |
| ports | 서비스 이름 없이 직접 열어둔 포트 |
| masquerade | NAT. 거쳐 나가는 패킷의 출처를 변경 |
| forward-ports | 포트 포워딩 |
| rich rules | 조건이 붙은 규칙 |

## service vs port
가능하면 service를 쓴다. 의도가 분명하고 다중 포트 서비스도 한 번에 처리된다.
port는 표준 서비스가 아닌 애플리케이션에 사용한다.
```
--add-service=http      (= 80/tcp)
--add-port=8080/tcp
```
서비스 정의는 XML로 저장된다. 기본 정의는 `/usr/lib/firewalld/services/`,
직접 만든 것은 `/etc/firewalld/services/` 에 둔다.
→ 비표준 포트를 쓰는 사내 애플리케이션을 등록할 때 사용

주요 포트: ssh 22 / http 80 / https 443 / mysql 3306

## permanent와 runtime의 분리
```
--add-...                            런타임만. 재부팅 시 소멸
--permanent --add-...                저장소만. 즉시 적용되지 않음
--permanent --add-... 후 --reload    저장 + 적용
```
- `--reload` 는 저장된 설정을 런타임에 다시 불러온다 → "저장된 상태로 되돌리기"로도 쓸 수 있다
- 분리된 이유: 실수로 자기 접속을 끊었을 때 **재부팅이 복구 수단**이 된다
- 위험한 규칙은 일부러 `--permanent` 없이 런타임에만 테스트한다
- 흔한 함정: `--permanent` 만 쓰고 `--reload` 를 잊어 "설정했는데 안 되는" 상태
- Ansible firewalld 모듈의 `permanent: yes` + `immediate: yes` 가 둘 다 처리

## rich rule — 조건을 붙인 규칙
service/port는 "포트를 열어라"가 전부다. "누구에게만"은 rich rule로 쓴다.
```
rule family="ipv4" source address="192.168.64.0/24" port port="8080" protocol="tcp" accept
```
| 조각 | 의미 |
|---|---|
| family | ipv4 / ipv6 |
| source address | 출처 제한 (CIDR) |
| port / protocol | 대상 포트와 프로토콜 |
| 동작 | accept / reject / drop / log |

로그를 남기는 예:
```
rule family="ipv4" service name="ssh" log prefix="SSH-ACCESS" level="info" limit value="3/m" accept
```
`limit` 없이 로그를 켜면 공격을 받을 때 디스크가 찬다.

조회: `firewall-cmd --list-rich-rules`

→ 3-Tier 계층 간 접근 제어에 사용한다 (웹→앱, 앱→DB 소스 제한)
