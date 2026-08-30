## 파이프로 원격 sudo 실행 시 터미널 오류

**증상**
[맥] $ cat key.pub | ssh user@host "sudo tee /path/file"
sudo: a terminal is required to read the password; either use ssh's -t option...

**원인**
파이프로 표준입력이 채워지면 ssh가 대화형 터미널(TTY)을 할당하지 않음.
sudo는 비밀번호 입력에 터미널이 필요해 충돌.

**해결**
[맥] ssh -t 로 터미널 강제 할당, 또는 명령을 나눠서 실행

**참조**
- man ssh (-t 옵션), man sudo

**교훈**
"사람이 비밀번호를 쳐줄 수 없는 상황"이 자동화의 근본 문제.
실무 해법: NOPASSWD로 특정 명령만 허용, 또는 Ansible become_password
→ W04에서 Ansible을 쓰는 이유와 직결