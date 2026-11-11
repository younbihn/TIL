# ansible-lint, yamllint 규칙 다루기

> 최초 작성: 2026-11-11 (11/09 자료 1-3 CI/CD)

## 세 도구의 역할

| 도구 | 보는 것 | 설정 파일 |
|---|---|---|
| yamllint | YAML 서식 (들여쓰기, 줄 길이, truthy 등) | `.yamllint` |
| ansible-lint | Ansible 작성 관례, 위험한 패턴. 설정 없이 돌리면 처음 49건 | `.ansible-lint` |
| ansible-playbook --syntax-check | Ansible 이 플레이북 구조를 해석할 수 있는지 (서버 접속 없음) | 없음 |

- ansible-lint 는 yaml 규칙을 돌릴 때 `.yamllint` 도 함께 읽는다. 하나의 설정이 두 도구에 쓰인다
- 같은 문제라도 도구마다 수위가 다르다. 예) `---` 없음은 yamllint 에선 warning(종료 코드 0), ansible-lint 에선 `yaml[document-start]` 실패

## 처음 49건을 세 갈래로 나누는 기준 (자료 기준, 저장소 사본 실측)

| 규칙 | 건수 | 판단 | 이유 |
|---|---|---|---|
| yaml[truthy] | 22 | 설정으로 허용 | 저장소 전체가 yes/no 로 일관됨 |
| name[casing] | 15 | 끈다 | nginx, nfs-utils 같은 원래 소문자인 이름으로 시작. 대문자로 바꾸면 오히려 부정확 |
| yaml[commas] | 7 | 설정으로 허용 | hardening.yml 의 세로 정렬을 위해 일부러 넣은 공백 |
| no-changed-when | 3 | **코드를 고친다** | 실제 위험 |
| risky-file-permissions | 1 | **코드를 고친다** | 실제 위험 |
| no-relative-paths | 1 | 경고로 남긴다 | templates/ 를 저장소 루트에 둔 구조라 `../` 필요. 롤 구조로 옮길 때 해소 |

- 기준: **의도적 선택과 부딪히는 규칙**은 설정으로 맞춘다 / **대개 맞지만 지금 구조상 예외**인 규칙은 경고로 둔다 / **실제 위험을 가리키는 규칙**은 끄지 않고 고친다
- 경고를 전부 끄는 것도, 전부 고치려다 지치는 것도 아닌 그 사이가 실무

## 끄는 세 가지 방법 (범위가 좁을수록 좋다)

| 방법 | 효과 |
|---|---|
| `skip_list` | 완전히 끔. 출력에도 안 나옴 |
| `warn_list` | 보이지만 실패시키지 않음 (CI 통과, 로그에 경고) |
| `# noqa: 규칙이름` | 그 한 줄만 예외 |

- 규칙을 끌 때마다 **왜 끄는지 주석**을 적는다
- `exclude_paths` 는 검사 대상에서 제외 (`.github/` 는 플레이북이 아니므로 ansible-lint 에서 뺀다. yamllint 는 계속 검사)

## command / shell 의 changed_when 판정 세 가지

| 상황 | 쓰는 것 |
|---|---|
| 조회만 한다 | `changed_when: false` |
| 돌면 반드시 바꾼다 (when 이나 handler 로 필요할 때만 돎) | `changed_when: true` |
| 출력으로 바뀌었는지 판단 가능 | `changed_when: "'changed' in result.stdout"` 같은 조건식 |

- 이 저장소 적용: `chronyc makestep`, `hwclock --systohc` (bootstrap.yml), `exportfs -rav` (nfs-server.yml 핸들러) → 모두 `true`
- 10/28 makestep 교훈의 완성: 그때는 상태를 바꾸는 명령에 `false` 를 붙여 "안 바꿨다"고 거짓 보고한 게 문제였다. `when` 으로 조건부 실행하게 고친 뒤, 린터가 요구하는 "그럼 돌면 바뀐 건가?"의 답이 `true`

## risky-file-permissions
- `file` 모듈로 디렉터리를 만들 때 `mode` 가 없으면 서버의 umask 에 따라 결과가 달라진다
```yaml
- ansible.builtin.file:
    path: "{{ nfs_mount_point }}"
    state: directory
    owner: root
    group: root
    mode: '0755'      # 따옴표 필수
```
- 서버의 설정에 기대지 않고 결과를 코드에 적는다

## no-relative-paths
- Ansible 은 템플릿을 **플레이북 옆의 `templates/`** 에서 먼저 찾는다
- 정석: `playbooks/templates/` 에 두고 `src: nginx.conf.j2` 만 쓰거나, 롤(role) 구조로 묶는다
- 지금은 `../templates/` 라서 경고. 구조를 바꾸면 1-4 의 경로가 전부 달라져서 경고로 남김

## package-latest 와 update_only
- `state: latest` 는 실행할 때마다 결과가 달라질 수 있어 "설치"에는 위험하다는 지적
- 패치는 원래 최신으로 올리는 게 목적이므로 `latest` 가 맞다
- `update_only: yes` 로 "설치된 것만 올리고 새로 설치하지 않는다"를 밝히면 규칙도 통과하고 의도도 정확해진다

## .yamllint 에서 기억할 설정
- `truthy.check-keys: false`: 워크플로의 `on:` 키 때문. YAML 옛 규칙에서 `on` 은 "참"이라 키까지 검사하면 truthy 경고가 난다
- `octal-values.forbid-implicit-octal: true`: 따옴표 없는 `0644` 를 금지. "권한은 따옴표로" 원칙을 기계가 지키게 한다
- `line-length.max: 160`: 기본 80자는 Ansible 의 긴 Jinja2 식과 rich rule 에 너무 짧다
- `ignore:` 에 `.checkup/`, `collections/` 를 넣어 로그와 내려받은 컬렉션은 검사하지 않는다

## 로컬 실행
```bash
# 맥. CI 와 같은 버전으로 고정한 가상환경
python3 -m venv ~/.venvs/lint
~/.venvs/lint/bin/pip install "ansible-lint==26.11.0" "yamllint==1.38.0"
~/.venvs/lint/bin/yamllint . && ~/.venvs/lint/bin/ansible-lint && echo "로컬 통과"
```
- push 전에 로컬에서 통과시킨다. 버전이 다르면 규칙이 달라 결과가 달라질 수 있다
