## CI 도입 전 로컬 lint 에서 ansible-lint 5건 실패, 수정 중 YAML 문법 오류

**상태**: 해결. 들여쓰기 오류(YAML 문법 오류)는 c2754a6 에서 고쳐졌고 CI(`플레이북 검증`)가 통과했다. 같은 수정 과정에서 생긴 `when` 자리표시 문자열 문제는 별도 노트(2026-11-11-when-placeholder-left-in-playbook.md). 2번 증상의 원인은 해당 파일을 직접 보지 않고 에러 메시지로 추정한 것.

**증상 1**
[맥] $ ~/.venvs/lint/bin/yamllint . && ~/.venvs/lint/bin/ansible-lint && echo "로컬 통과"
- yamllint: `./inventory/hosts.yml 1:1 warning missing document start "---"` (종료 코드 0)
- ansible-lint: `Failed: 5 failure(s), 1 warning(s) in 11 files processed` 로 종료되어 「로컬 통과」가 출력되지 않음
  - `yaml[document-start]` inventory/hosts.yml:1
  - `no-changed-when` playbooks/bootstrap.yml:56, :61 / playbooks/nfs-server.yml:70
  - `risky-file-permissions` playbooks/nfs-client.yml:18
  - `no-relative-paths` playbooks/deploy-web.yml:13 (warning, 실패 건수에는 미포함)

**원인 1**
- command/shell 은 실행할 때마다 항상 changed 로 보고되므로 `changed_when` 명시를 요구
- file 모듈로 디렉터리를 만들 때 `mode` 가 없으면 서버 umask 에 따라 결과가 달라짐
- 인벤토리에 문서 시작 `---` 가 없었음. yamllint 는 warning 으로 통과시키지만 ansible-lint 는 같은 문제를 실패로 처리 (두 도구의 수위 차이)
- `PATH altered` 경고는 venv 를 절대경로로 호출해서 뜨는 것. `oneline`/`tree` callback deprecation 은 `ansible.cfg` 의 callback 설정에서 옴. 둘 다 실패 원인은 아님

**해결 1 (안)**
- hosts.yml 첫 줄에 `---`
- bootstrap.yml 의 `chronyc makestep`, `hwclock --systohc` 와 nfs-server.yml 의 `exportfs -rav` 에 `changed_when: true` (when 이나 handler 로 필요할 때만 돌기 때문)
- nfs-client.yml 의 file 작업에 `mode: '0755'`
- deploy-web.yml 의 `src` 는 `"{{ playbook_dir }}/../templates/..."` 로 바꾸거나 템플릿을 playbooks/templates/ 로 이동 (경고라 선택)

**증상 2**
[맥] 수정 후 yamllint 재실행
./playbooks/bootstrap.yml
  57:5  error  syntax error: expected <block end>, but found '?' (syntax)
  58:5, 59:5, 62:5, 63:5, 64:5  error  wrong indentation: expected 6 but found 4 (indentation)

**원인 2 (추정)**
추가한 키(`changed_when`, `when` 등)가 태스크의 다른 키가 있는 열(6칸)이 아니라 `- name:` 의 대시와 같은 열(4칸)에 들어감. 값에 `?`, `:` 가 따옴표 없이 들어갔을 가능성도 있음.

**해결 2 (안)**
`- name:` 의 `name`, 모듈명, `when`, `changed_when`, `tags` 의 첫 글자를 같은 열에 맞춘다. 대시 뒤 공백 한 칸 때문에 `name` 은 대시보다 2칸 안쪽이다.
```yaml
    - name: 시각 즉시 보정
      ansible.builtin.command: chronyc makestep
      changed_when: true
      when: ...
```
수정 후 `yamllint playbooks/bootstrap.yml` 로 문법 오류부터 확인하고 전체 lint 재실행.

**참조**
- ansible-lint rules: no-changed-when, risky-file-permissions, no-relative-paths, yaml
- ansible/lint-rules.md, cicd/github-actions.md

**교훈**
- 로컬에서 `echo "로컬 통과"` 가 찍히기 전에는 push 하지 않는다. 안 그러면 CI 에서 같은 이유로 실패한다
- yamllint 가 통과(경고만)해도 ansible-lint 는 실패할 수 있다. 두 도구를 모두 돌린다
- 규칙을 끄기 전에 "의도적 선택인가, 실제 위험인가"를 먼저 가른다 (ansible/lint-rules.md)
