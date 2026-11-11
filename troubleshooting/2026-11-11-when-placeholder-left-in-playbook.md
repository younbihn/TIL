## 플레이북에 자리표시 문자열이 그대로 들어가 when 구문 오류

**상태**: 원인 확인, 수정안 제시. 수정 적용 후 재실행(app01, db01, 그리고 이미 실행한 lab01, lab02) 결과는 미확인. 잘못된 내용이 이미 `main` 에 병합된 상태라 수정도 PR 로 올려야 한다.

**증상**
[맥] $ ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --limit app01,db01 -K
TASK [시스템 시각을 하드웨어 시계에 반영 (보정했을 때만)]
[ERROR]: Task failed: A 'when' expression failed: Syntax error in expression: unexpected '<'
Origin: playbooks/bootstrap.yml:63:13
  63     when: <기존 조건>
fatal: [app01]: FAILED!  fatal: [db01]: FAILED!

**원인**
lint 수정 과정에서 안내받은 예시 코드의 **자리표시**(`when: <기존 조건>`)를 실제 코드처럼 파일에 그대로 적었다. 같은 안내의 첫 태스크 예시 `when: time_offset_exceeded | default(false)` 도 실제 변수가 아닌 예시 값이었다.
- 두 번째 태스크(63줄)는 `<` 때문에 **에러로 드러났다**
- 첫 번째 태스크(58줄)는 `time_offset_exceeded` 가 정의되지 않은 변수라 `default(false)` 로 항상 거짓이 되어 **에러 없이 `chronyc makestep` 이 절대 실행되지 않았다** (조용한 오동작)
- 두 태스크의 `tags: [time]` 도 사라졌다 (`--tags time` 으로 실행하면 빠짐)
- 원래 조건: `when: clock_drift_sec | int > clock_drift_threshold` (`clock_drift_threshold: 5` 는 플레이 vars)

**왜 CI 는 초록이었나**
`ansible-playbook --syntax-check` 는 `when` 식의 **내용을 계산하지 않는다**. yamllint, ansible-lint 에게는 그냥 문자열이다. 세 검사를 모두 통과해도 `{{ }}`, `when` 안의 변수와 값은 검증되지 않는다 (cicd/github-actions.md 의 「초록 체크가 보증하는 범위」).

**해결 (안)**
두 태스크를 원래 조건과 태그로 복원한다.
```yaml
    - name: 시각 즉시 보정 (오차 임계치 초과 시에만)
      ansible.builtin.command: chronyc makestep
      changed_when: true
      when: clock_drift_sec | int > clock_drift_threshold
      tags: [time]

    - name: 시스템 시각을 하드웨어 시계에 반영 (보정했을 때만)
      ansible.builtin.command: hwclock --systohc
      changed_when: true
      when: clock_drift_sec | int > clock_drift_threshold
      tags: [time]
```
- 수정 후 `yamllint`, `ansible-lint` 로 서식 확인, `--check --diff` 와 실제 실행으로 동작 확인
- 이미 잘못된 조건으로 돌린 lab01, lab02 는 `makestep` 이 건너뛰어졌을 뿐 상태를 망가뜨리지는 않았다. 수정 후 다시 실행해 시각 보정이 필요한 경우 반영되게 한다

**참조**
- ansible/lint-rules.md (changed_when 판정), cicd/github-actions.md

**교훈**
- 안내나 문서의 `<…>`, `…`, `기존 조건` 같은 자리표시는 코드가 아니다. 붙여 넣기 전에 **원래 값을 `git diff` / `git show <커밋>:파일` 로 확인**한다
- 수정 후에는 diff 로 의도하지 않은 줄(사라진 `tags`)이 없는지 본다
- 가장 위험한 것은 에러가 나는 오류가 아니라 **조용히 안 도는** 오류다. 조건이 있는 태스크는 한 번은 `--check -v` 나 로그로 실제로 실행되는지 확인한다
- CI 초록은 의미 검증이 아니다
