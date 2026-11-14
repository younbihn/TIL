# ansible-lint 가 `community.general.sefcontext` 를 모른다고 할 때 (컬렉션 누락)

## 증상
- 맥에서 `yamllint . && ansible-lint` 를 돌리면 먼저 `zsh: command not found: yamllint` (이번이 4번째 반복)
- 린트를 venv(`~/.venvs/lint`)에 설치한 뒤에는 ansible-lint 가 selinux 플레이북의 `community.general.sefcontext` 를 알 수 없는 모듈로 보고했다

## 원인
- `yamllint` 는 맥에 설치된 적이 없었다. 이전에는 `&&` 로 이은 린트가 "통과"가 아니라 "미실행"이었다(06cded2 기록)
- ansible-lint 는 플레이북이 쓰는 모듈의 컬렉션이 설치돼 있어야 모듈을 해석한다. `collections/requirements.yml` 에 `community.general` 이 없어서 CI 와 로컬 모두 같은 문제가 생길 수 있었다

## 해결
1. 린트 도구를 venv 에 설치하고 CI 와 같은 버전으로 고정: `ansible-lint==26.11.0`, `yamllint==1.38.0`
2. `collections/requirements.yml` 에 `community.general` 추가 (커밋 df335f7), `ansible-galaxy collection install -r collections/requirements.yml` 로 venv 에도 설치
3. 결과: `Passed: 0 failure(s), 9 warning(s)` (조건: 2026-11-14 16:04 사용자 맥, 한 번 실행). 경고는 `.ansible-lint` 의 `warn_list`(no-relative-paths, fqcn[canonical])에 의도적으로 둔 것

## 참조
- 로컬 린트 결과가 CI 와 다를 수 있으니 버전을 CI 와 맞춘다
- 컬렉션 의존성은 `collections/requirements.yml` 한 곳에 모은다. 플레이북에 새 컬렉션 모듈을 쓰면 같은 PR 에서 이 파일도 고친다
- Rating 3/5 는 최종 판정이 아니라 규칙 위반 수의 지표라고 이해했다(설명 근거는 미확인)
