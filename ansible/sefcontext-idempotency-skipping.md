# SELinux 경로 규칙 플레이북과 멱등성: changed=0 의 이유를 skipping 으로 확인 (lab02)

## 측정 조건
- 일시: 2026-11-14 15:18~15:19 KST, 맥에서 `ansible-playbook -i inventory/hosts.yml playbooks/selinux-web-content.yml --limit lab02 -K`, 대상 lab02 한 대, 콜백 timer 와 profile_tasks, 각 한 번(중간에 become 비밀번호 문제로 한 번 실패한 실행은 제외, 별도 troubleshooting 참조)
- 플레이북 구성(학습 자료 1-9 11절, 사용자의 파일 내용은 직접 보지 못함): 관리 도구 설치(dnf) → 디렉터리 생성(`file`) → 경로 규칙 등록(`community.general.sefcontext`) → 어긋난 라벨 조회(`restorecon -Rnv`, `changed_when: false`) → 어긋난 라벨 복원(`restorecon -Rv`, `when: relabel_check.stdout | length > 0`)

## 결과
| 태스크 | 1회차 | 재실행 |
|---|---|---|
| 관리 도구 설치 | changed (3.47초) | ok (0.85초) |
| 디렉터리 생성 | changed | ok |
| 경로 규칙 등록 | changed | ok |
| 어긋난 라벨 조회 | ok | ok |
| 어긋난 라벨 복원 | changed | **skipping** |
| RECAP | ok=6 changed=4 | ok=5 changed=0 skipped=1 |
| 소요 | 7초 | 3초 |

서버 확인: `ls -Zd /srv/web` → `unconfined_u:object_r:httpd_sys_content_t:s0`, `semanage fcontext -l -C` → `/srv/web(/.*)?  all files  system_u:object_r:httpd_sys_content_t:s0`.

## 읽은 것
- **규칙 등록과 라벨 적용은 별개 단계다.** `sefcontext` 는 정책에 규칙만 기록하고 파일 라벨은 안 바꾼다(`semanage fcontext -a` 와 같음). 실제 파일에는 `restorecon` 이 규칙을 보고 적용한다. 그래서 플레이북이 두 단계를 따로 가진다
- **재실행에서 changed=0 인 이유를 건너뜀으로 설명할 수 있다.** 조회 태스크 출력이 비어 `when` 이 거짓이 되어 복원 태스크가 skipping 이었다. "숫자가 0" 이 아니라 "바꿀 것이 없어서 건드리지 않았다"를 보여 준다. 조회 태스크가 `changed_when: false` 인 것은 조회만 하기 때문에 정당하다(0-6 의 교훈: `changed_when: false` 는 조회성 명령에만)
- 라벨의 user 부분은 `unconfined_u`, 규칙은 `system_u` 로 다르게 보인다. `restorecon` 이 `-F` 없이는 type 만 고치기 때문으로 보이나(알려진 동작) `-F` 로 시험하지 않았다(미확인)

## 한계와 미확인
- lab02 한 대에서만 확인. lab01 에 적용했는지, 플레이북의 `hosts:` 값은 이 기록에 없다(미확인)
- 로컬 `yamllint` 가 없어 `ansible-lint` 가 돌지 않았다. 커밋은 `--syntax-check` 통과 후 만들었고 린트는 CI 에서 처음 실행된다
- 이 플레이북은 파일 문맥까지만 다루며 nginx 가 `/srv/web` 을 서빙하게 하는 설정은 없다
