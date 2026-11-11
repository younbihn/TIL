# 인벤토리 그룹과 변수 배치

> 최초 작성: 2026-11-11 (1-4 3-Tier 설계와 서버 증설)

## 계층별 그룹과 그룹을 묶는 그룹
```yaml
all:
  children:
    webservers:   { hosts: { lab01: {ansible_host: 192.168.64.2}, lab02: {ansible_host: 192.168.64.3} } }
    appservers:   { hosts: { app01: {ansible_host: 192.168.64.4} } }
    dbservers:    { hosts: { db01:  {ansible_host: 192.168.64.5} } }
    threetier:
      children:
        webservers:
        appservers:
        dbservers:
    nfs_server:  { hosts: { lab01: } }
    nfs_clients: { hosts: { lab02: } }
```
(실제 파일은 블록 스타일. 위는 구조만 줄여 쓴 것)
- `threetier` 는 호스트를 직접 갖지 않고 하위 그룹을 가진다. 세 계층을 한 이름으로 부를 때 쓴다 (방화벽 플레이북, 공통 구성)
- 하위 그룹 이름 뒤 콜론이 비어 있으면 "새로 정의하는 게 아니라 위에서 정의한 그 그룹을 가리킨다"는 뜻
- 한 호스트가 여러 그룹에 속할 수 있다 (lab01 은 webservers 와 nfs_server)
- 확인: `ansible-inventory -i inventory/hosts.yml --graph`, 호스트 하나의 변수는 `--host app01`

## 변수는 "참조하는 쪽"으로 위치를 정한다
- `app_port` 는 앱의 포트라서 `appservers.yml` 에 둘 것 같지만, 쓰는 곳은 세 군데다: 앱의 systemd 유닛(앱 서버), nginx upstream(웹 서버), 방화벽 규칙(앱 서버)
- `appservers.yml` 에 두면 웹 서버를 처리할 때 이 변수가 안 보인다
- 그래서 **여러 그룹이 함께 쓰는 값은 `group_vars/all.yml` 에 한 번만** 적는다. 같은 값이 두 곳에 적히는 순간 언젠가 어긋난다
```yaml
# inventory/group_vars/all.yml
app_port: 8000
db_port: 3306
db_name: firstcome
db_user: fcapp
```
- 비밀번호 같은 값은 이 파일에 넣지 않는다 (public 저장소). vault 나 저장소 밖 파일로
- 확인: `ansible threetier -i inventory/hosts.yml -m debug -a "var=db_name"` 이 네 대 모두에서 같은 값

## 인벤토리에서 값을 끌어오는 Jinja2
- `groups['webservers']` → `['lab01', 'lab02']`
- `hostvars['lab01']['ansible_host']` → `'192.168.64.2'`
- `groups['webservers'] | map('extract', hostvars, 'ansible_host') | list` → IP 목록. `map` 결과는 아직 계산되지 않은 묶음이라 `list` 로 확정시킨다
- `when: inventory_hostname in groups['appservers']`: 지금 처리 중인 서버가 그 그룹일 때만. **`when` 에는 중괄호를 쓰지 않는다** (이 자리 전체가 이미 Jinja2 식)

## YAML 문자열: `>-` 와 `|`
- `>-`: 여러 줄을 **공백 하나로 이어 한 줄**로 만들고 끝 줄바꿈을 뗀다 → 규칙 한 줄(rich rule), 긴 조건식
- `|`: 줄바꿈을 그대로 유지 → 스크립트, 파일 내용

## `--limit` 은 좁히기만 한다
- 플레이의 `hosts:` 가 지정한 대상 **안에서** 걸러내는 필터. `hosts: webservers` 인 플레이에 `--limit app01` 을 주면 교집합이 비어 `skipping: no hosts matched`
- 새 서버를 대상에 넣으려면 플레이의 `hosts:` 를 `threetier`(또는 `all`)로 넓혀야 한다
- 실행 전에 `--list-hosts` 로 실제 대상을 확인하는 습관 → `troubleshooting/2026-11-11-ansible-limit-no-hosts-matched.md`
