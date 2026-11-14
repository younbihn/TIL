# 정리 명령 두 가지가 조용히 실패했다: ssh 로 sudo 가 막힘, `&&` 로 이어 둔 린트가 건너뛰어짐 (2026-11-14)

## 증상
L4 비교 실험을 끝내고 한 번에 붙여 넣은 정리 명령과 커밋 전 검사 명령에서 두 가지가 의도대로 되지 않았다.

1. 정리 명령 `ansible-playbook … --tags config` 는 성공(changed=2, 설정 재배포와 reload)했지만 바로 이어진 `ssh lb01 'sudo firewall-cmd --permanent --remove-port=81/tcp && sudo firewall-cmd --reload'` 는 `sudo: a terminal is required to read the password; either use ssh's -t option or configure an askpass helper` 로 실패했다. 그 결과 lb01 의 81번 방화벽 규칙이 남았다(재시도 여부는 확인하지 않음).
2. `yamllint -s . && ansible-lint` 가 `zsh: command not found: yamllint` 로 끝났다. `&&` 때문에 뒤의 `ansible-lint` 는 실행조차 되지 않았고, 이어서 `git status`, `git add`, `git commit` 은 그대로 진행됐다.

## 원인
- ssh 로 원격 명령 한 줄을 실행하면 터미널(tty)이 할당되지 않아 sudo 가 비밀번호를 물을 수 없다. 11/12 의 같은 증상(`troubleshooting/2026-08-30-sudo-terminal-required.md`)의 재발이다. 정리 명령을 붙여 넣을 때 앞의 `ansible-playbook` 만 비밀번호(`-K`)를 받고 뒤의 `ssh … sudo` 는 받을 길이 없었다.
- 맥에 `yamllint` 가 설치되어 있지 않았다. CI 는 `pip install "ansible-lint==26.11.0" "yamllint==1.38.0"` 로 버전을 고정해 설치하지만 로컬에는 같은 도구가 없다(`ansible-lint` 가 로컬에 있는지는 확인하지 않음). 명령을 `&&` 로 이으면 앞이 실패했을 때 뒤가 실행되지 않으므로 검사가 "통과"가 아니라 "미실행"이 되었다.
- 참고: 같은 출력의 `** WARNING: connection is not using a post-quantum key exchange algorithm` 는 ssh 클라이언트가 서버의 키 교환 방식에 대해 보여 주는 안내이고 위 실패와 관련이 없다.

## 해결
1. 방화벽 규칙은 tty 를 요구하는 `-t` 로 실행한다. 제거 후 열린 포트 목록으로 확인한다.
```
ssh -t lb01 'sudo firewall-cmd --permanent --remove-port=81/tcp && sudo firewall-cmd --reload'
ssh -t lb01 'sudo firewall-cmd --list-ports'
```
2. 린트 도구가 있는지부터 확인하고, 없으면 CI 와 같은 버전으로 설치한다(설치 가능 여부와 실행 결과는 확인하지 않음).
```
which yamllint ansible-lint
pip install "yamllint==1.38.0" "ansible-lint==26.11.0"
yamllint -s . ; ansible-lint
```
검사를 `;` 로 이으면 앞이 실패해도 뒤가 돌아 결과를 둘 다 본다. 커밋 직전 검사는 `&&` 가 맞지만, 도구가 없다는 사실은 "통과"로 읽으면 안 된다.

## 참조
- 이 변경은 PR 로 병합됐고 `main` 은 필수 상태 검사 `플레이북 검증`(CI)을 요구하므로 CI 에서는 같은 검사가 돌았을 것이다. Actions 결과 화면은 직접 확인하지 않았다(미확인).
- 확인 명령: `which 명령`, `echo $?`(직전 명령의 종료 코드), `firewall-cmd --list-ports`
- 한꺼번에 붙여 넣는 명령 묶음에서는 한 줄의 실패가 다음 줄을 막지 않으므로, 각 줄의 결과(특히 `sudo` 가 들어간 줄)를 따로 확인한다.
