# 이름이 이상한 .env 사본이 gitignore 를 피해 커밋됨, 그리고 Grafana 비밀번호 교체

## 환경
- 💻 맥, `~/infra-fleet-automation`, 브랜치 feat/monitoring, zsh
- 2026-11-14 16:05~16:08

## 증상
`git add monitoring/` 후 status 에 `.env` 로 시작하는 이상한 파일이 `A` 로 나타났고, 확인 없이 같은 블록에서 commit 과 push 까지 진행됐다.
```
A  "monitoring/.env\030\342\211\210"
```
이름은 `.env` 뒤에 Ctrl+X(`\030`)와 `≈`(Option+x)가 붙은 것. nano 에서 저장할 때 파일 이름 입력란에 키 입력이 글자로 들어간 것으로 추정(미확인).

## 원인
- `.gitignore` 의 `monitoring/.env` 는 이름이 정확히 같은 파일만 막는다. 접미 글자가 붙은 사본은 걸러지지 않는다.
- 그 파일에는 `GRAFANA_ADMIN_PASSWORD=` 값이 비어 있지 않게 들어 있었다(`grep -c` 로 줄 수만 확인, 값은 출력하지 않음). 비밀번호가 origin 에 두 번 올라갔다(377afc1, 강제 푸시한 117936e).
- 방지 장치 실수: status 확인과 commit, push 를 한 블록에 묶어 이상한 항목이 보여도 멈출 틈이 없었다.

## 해결
1. 이상한 파일을 `git rm --cached` 와 `rm` 으로 제거, `.gitignore` 에 `monitoring/.env*` 와 `!monitoring/.env.example` 추가
2. 아직 PR 이 없는 본인 브랜치라서 `git commit --amend --no-edit` 후 `git push --force-with-lease`(117936e -> 1c82bea). `ls-tree` 로 파일이 사라진 것을 확인
3. 이전 커밋은 SHA 로 한동안 열람될 수 있으므로 비밀번호를 교체

## 함정 모음
- zsh 에서 `[!.]` 는 `!.` 이 히스토리 확장으로 읽혀 `event not found` 가 난다. zsh 에서는 `[^.]` 를 쓰거나 `!` 를 작은따옴표 안에 둔다. 실행되지 않은 줄 뒤에 `--amend` 와 푸시가 이어져 같은 내용이 다시 올라갔다.
- Grafana 의 `GF_SECURITY_ADMIN_PASSWORD` 는 최초 DB 생성 시에만 적용된다. `.env` 를 바꾸고 컨테이너를 재생성해도 기존 계정 비밀번호는 바뀌지 않는다. 기동 중인 컨테이너에서 `docker compose exec grafana grafana cli admin reset-admin-password '값'` 로 바꾸고 .env 도 맞춘다. 출력 `Admin password changed successfully` 로 변경 확인.
- 확인 값이 기대와 다르면(`wc -l .env` 가 1 이 아니라 3) 멈춰야 하는데 블록을 그대로 실행해 `.env` 가 덮어써졌다. 다른 두 줄의 내용은 확인하지 못했다(compose 는 이 변수 하나만 쓴다).

## 재발 방지
- `git add` 직후 `git status --short` 를 보고, 의도한 파일만 있는지 눈으로 확인한 뒤 commit 한다. 확인과 commit 을 한 블록에 묶지 않는다.
- 비밀 파일 규칙은 정확한 이름이 아니라 접두 패턴(`.env*`)으로 쓰고 예시 파일만 예외로 둔다.
- nano 저장은 Ctrl+O, Enter, Ctrl+X 를 따로 누른다.
