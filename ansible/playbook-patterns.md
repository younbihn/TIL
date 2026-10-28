# Ansible 플레이북 패턴

## 설정 파일을 다루는 세 가지 모듈

| 모듈 | 용도 | 선택 기준 |
|---|---|---|
| lineinfile | 특정 줄만 교체 | 기존 파일의 일부만 수정 (sshd_config 등) |
| copy | 파일 전체 배포 | 내용이 고정된 파일 |
| template | 변수 치환 후 배포 | 서버마다 값이 다른 설정 |

lineinfile을 쓰는 이유: 전체를 덮어쓰면 OS 업데이트로 추가된 기본 설정이 유실된다.

### 정규표현식
```
^#?PermitRootLogin
  ^   줄의 시작 (없으면 다른 줄 중간에 걸릴 수 있음)
  #?  # 이 0개 또는 1개 → 주석 처리된 기본값까지 잡기 위함
```
수량자: `?` 0~1회 / `*` 0회 이상 / `+` 1회 이상

## loop
- 같은 작업을 목록만큼 반복. `{{ item }}` 으로 각 항목 참조
- 항목이 딕셔너리면 `item.key` 로 접근
- Shell의 for와 달리 **항목마다 멱등성이 개별 판정**됨
- 목록을 vars로 빼두면 설정 항목을 한 곳에서 관리 가능

## notify / handlers
- notify는 해당 태스크가 changed일 때만 핸들러를 예약. ok면 호출되지 않음
- 여러 태스크가 같은 핸들러를 호출해도 **플레이 끝에 한 번만** 실행 (중복 제거)
- tasks 전체가 끝난 뒤 실행 → 중간 재시작으로 연결이 끊기는 사고 방지
- 멱등성 검증 시 changed=0이면 핸들러도 실행되지 않아야 정상

## validate — 배포 전 검사
파일을 실제 위치에 놓기 전에 검사 명령을 실행한다. `%s`가 임시 파일 경로로 치환되고,
통과하지 못하면 배포가 중단되며 원본은 유지된다.

| 대상 | validate |
|---|---|
| sudoers | `visudo -cf %s` |
| sshd_config | `sshd -t -f %s` |
| nginx.conf | `nginx -t -c %s` |

→ 설정 파일을 다루는 데몬은 대개 검사 모드를 제공한다 (-t, -c 등)

## 안전장치 패턴: stat + assert
위험한 작업 전에 전제 조건을 코드로 검사한다.
```yaml
- ansible.builtin.stat:
    path: "/home/{{ ansible_user }}/.ssh/authorized_keys"
  register: authkey

- ansible.builtin.assert:
    that:
      - authkey.stat.exists
      - authkey.stat.size | int > 0
    fail_msg: "키가 없다. 비밀번호 인증을 끄면 접속 불가."
```
→ 사람의 기억에 의존하지 않고 코드가 검증하게 만든다

## failed_when: false
없는 대상을 조작하려 할 때 발생하는 에러를 실패로 집계하지 않는다.
예) 설치되지 않은 서비스를 방화벽에서 차단하려는 경우

## 조건부 실행 (set_fact + when)
- `set_fact`: 플레이 실행 중에 계산한 값을 변수로 만든다
- `ansible_facts.date_time.epoch` : 대상 서버의 유닉스 시간 (Gathering Facts가 수집)
- `lookup('pipe', '명령')` : 컨트롤 노드에서 명령을 실행해 결과를 가져옴
- `| abs` 절댓값 / `| int` 정수 변환 (Jinja2는 문자열로 넘기므로 비교 전 변환 필요)
- `>-` YAML에서 여러 줄을 한 줄로 이어붙이는 표기

## changed_when 의 올바른 사용
- 상태를 바꾸지 않는 **조회성 명령에만** 붙인다
- 실제로 상태를 바꾸는 명령에 붙이면 "멱등한 것처럼 보고하지만 부작용이 있는" 상태가 된다

실제 사례: `chronyc makestep` 에 `changed_when: false` 를 붙여, 정상 서버의
시간 동기화 상태를 매 실행마다 리셋하면서 changed=0 으로 보고했다.
→ 해결: `when` 으로 조건부 실행하고 changed_when 제거.
→ 멱등성은 보고 숫자가 0인 것이 아니라 **바꿀 게 없을 때 건드리지 않는 것**이다.

## facts 접근 방식 (중요)
Gathering Facts가 수집한 정보는 `ansible_facts` 딕셔너리에 담긴다.
과거에는 편의상 `ansible_` 접두사를 붙여 최상위 변수로도 자동 복사해줬으나,
이름 충돌과 추적 곤란 때문에 제거 예정이다 (ansible-core 2.24).

| 예전 (사라짐) | 지금 써야 할 것 |
|---|---|
| ansible_hostname | ansible_facts.hostname |
| ansible_date_time.epoch | ansible_facts.date_time.epoch |
| ansible_default_ipv4.address | ansible_facts.default_ipv4.address |
| ansible_distribution | ansible_facts.distribution |
| ansible_memtotal_mb | ansible_facts.memtotal_mb |

규칙: `ansible_` 접두사를 떼고 `ansible_facts.` 아래로 넣는다.

수집되는 facts 전체 확인:
```bash
ansible lab01 -i inventory/hosts.yml -m setup | head -60
```
setup 모듈이 Gathering Facts를 수행하는 모듈이다. OS 버전, IP, 메모리, 디스크,
CPU 정보가 변수로 들어오므로 플레이북에서 조건 분기에 쓸 수 있다.

## 실행 제어 옵션

| 목적 | 옵션 |
|---|---|
| 문법만 검사 | `--syntax-check` |
| 변경 예정만 확인 | `--check`, `--check --diff` |
| 특정 호스트만 | `--limit lab02` |
| 특정 태스크만 | `--tags time` |
| 특정 태스크 제외 | `--skip-tags time` |
| 중간부터 재개 | `--start-at-task "이름"` |
| 하나씩 확인하며 | `--step` |
| 목록 확인 | `--list-hosts`, `--list-tasks`, `--list-tags` |

태그는 목록으로 적어 하나의 태스크에 여러 개를 붙일 수 있다: `tags: [account, ssh]`
