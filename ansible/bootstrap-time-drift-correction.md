# bootstrap.yml 5대 실행과 시각 오차 보정 (2026-11-13)

## 조건
- 맥에서 `ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml -K`, 대상 5대(app01, db01, lab01, lab02, lb01), 22:05:48 시작
- 콜백 `ansible.posix.timer`, `ansible.posix.profile_tasks` 로 태스크별 시간을 측정 (`ANSIBLE_CALLBACKS_ENABLED` 환경변수)
- 병렬 수(forks)는 기본값(값 미확인)

## 결과
| 서버 | ok | changed | skipped | 시각 오차(컨트롤 노드 기준) |
|---|---|---|---|---|
| app01 | 8 | 0 | 2 | 2초 |
| db01 | 10 | 2 | 0 | 75365초 |
| lab01 | 10 | 2 | 0 | 75349초 |
| lab02 | 10 | 2 | 0 | 75389초 |
| lb01 | 10 | 2 | 0 | 75334초 |

- 총 소요 6.3초(`Playbook run took 6 seconds`, 타이머 0:00:06.328). 가장 오래 걸린 태스크는 Gathering Facts 1.71초
- 오차가 큰 4대에서 "시각 즉시 보정"과 "시스템 시각을 하드웨어 시계에 반영" 두 태스크가 changed 였고, 오차 2초인 app01 은 두 태스크 모두 skipping 이었다
- 오차 75,3xx초는 약 20.9시간이다(75349 ÷ 3600)

## 해석
- `when: clock_drift_sec | int > clock_drift_threshold` 조건이 의도대로 동작했다. 오차가 임계치를 넘은 서버에서만 보정하고, 넘지 않은 서버는 건너뛰었다. 11일에 자리표시 문자열이 남아 있던 이 `when` 을 고친 뒤 처음으로 실제 서버에서 확인된 결과다(그날 lab01, lab02 에 재실행했는지는 미확인이었음).
- lb01 이 5대에 포함됐다는 것은 `hosts: all` 변경으로 bootstrap 이 lb01 에도 적용됐다는 확인이다. lb01 의 나머지 태스크(deploy 계정, SSH 키, sudoers, chronyd, 방화벽 ssh)는 모두 ok 라서 이미 적용된 상태였다.
- 이번 changed=2 는 설정이 바뀐 것이 아니라 시각 오차가 있어서 생긴 조건부 변경이다. 멱등성 확인은 한 번 더 실행해서 changed=0 이 나와야 한다(아직 안 함).

## 미확인
- 오차 원인: 네 서버가 약 20.9시간 어긋났고 app01 만 2초였다. VM 이 일시정지나 중단된 시간이 있었을 가능성을 의심하지만 확인하지 않았다. 오차의 방향(빠름 또는 느림)도 출력에 없다.
- `clock_drift_threshold` 의 값 (오차 2초는 건너뛰고 75k초는 보정한 것만 확인)
- 2회차 실행 결과(changed=0 기대), 보정 후 chronyd 동기 상태(`chronyc tracking`)
- 참고: 11월 12일 이중화 측정 때는 프로브 시작 시각과 lb01 의 HAProxy 로그 시각이 맞아떨어졌으므로(DOWN 이 시작 약 30초 뒤) 그때는 시계가 맞는 상태였다고 본다.
