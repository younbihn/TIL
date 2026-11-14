# ansible-playbook 이 Gathering Facts 에서 실패: "sudo: 암호가 필요합니다" (2026-11-14, lab02)

## 증상
`ansible-playbook -i inventory/hosts.yml playbooks/selinux-web-content.yml --limit lab02 -K` 를 같은 명령으로 세 번 연달아 실행했을 때 둘째 실행(15:19:06)만 첫 태스크에서 실패했다.
```
TASK [Gathering Facts]
[ERROR]: Task failed: Action failed: The following modules failed to execute: ansible.legacy.setup.
| Module result deserialization failed: No start of json char found
fatal: [lab02]: FAILED! => {... "module_stdout": "sudo: 암호가 필요합니다\r\n", "module_stderr": "Shared connection to 192.168.64.3 closed.\r\n", "rc": 1 ...}
```
첫 실행(15:18:57)과 셋째 실행(15:19:10)은 성공했다. 실패한 실행은 0.38초 만에 끝났고 아무 변경도 하지 않았다(`ok=0 changed=0 failed=1`).

## 원인 (추정, 확인 못 함)
`-K` 로 받은 become 비밀번호가 서버의 `sudo` 에 전달되지 않아 sudo 가 비밀번호를 다시 요구했다. 오류 문구의 `module_stdout` 이 sudo 의 메시지이고, `No start of json char found` 는 모듈이 JSON 대신 sudo 오류를 출력해서 생긴 2차 증상이다. 즉 플레이북이나 SELinux 문제가 아니다. 비밀번호 입력 오타나 빈 입력일 가능성이 가장 높으나 입력 내용을 볼 수 없어 확인하지 못했다. 서버 쪽 원인(sudo 설정 변경 등)은 셋째 실행이 같은 명령으로 성공한 점에서 가능성이 낮다.

## 해결
같은 명령을 다시 실행해 비밀번호를 정확히 입력했다. 반복되면 `BECOME password:` 줄에서 입력이 맞았는지 확인하고, 자주 입력하기 번거로우면 `ssh-add` 와 같은 방식이 아니라 become 비밀번호를 파일에 두지 않는 방법(예: `ansible-vault`, 또는 sudoers 의 특정 명령 NOPASSWD)을 검토한다(이번에 시험하지 않음).

## 참조
- 구분하는 법: `msg` 가 `Module result deserialization failed` 이고 `module_stdout` 에 `sudo:` 로 시작하는 문구가 있으면 모듈이 아니라 권한 상승 단계의 실패다. 대상에 직접 `ssh -t lab02 'sudo -v'` 로 sudo 가 되는지 본다.
- 11/12 의 `ssh 서버 'sudo ...'` tty 문제(`troubleshooting/2026-08-30-sudo-terminal-required.md`)와 같은 계열: 사람이 비밀번호를 쳐 줄 수 없는 경로에서 sudo 가 막힌다.
