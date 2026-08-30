# SSH

## 서비스 켜기
sudo systemctl enable --now sshd
- enable: 부팅 시 자동 시작 등록 / --now: 지금 즉시 시작

## 키 기반 로그인
ssh-keygen -t ed25519        # 키 쌍 생성 (맥)
ssh-copy-id user@host        # 공개키를 서버 ~/.ssh/authorized_keys에 등록
→ 이후 비밀번호 없이 로그인. 서버 접근의 표준 방식.

## 공개키 인증의 구조
- 키 쌍은 **접속하는 쪽(클라이언트)**에서 생성
  - 개인키(id_ecdsa): 클라이언트에 보관, 유출 금지
  - 공개키(id_ecdsa.pub): 서버의 ~/.ssh/authorized_keys에 등록
- authorized_keys = "이 계정으로 들어올 수 있는 공개키 목록" (한 줄에 키 하나)
- 서버에서 생성한 키는 "서버가 밖으로 나갈 때" 쓰는 것 → 클라이언트 접속과 무관
- 키 종류: ed25519(권장, 짧고 빠름) / ecdsa / rsa

## 권한 요구사항 (sshd가 강제)
~/.ssh          → 700
authorized_keys → 600
이유: 느슨하면 제3자가 키를 몰래 추가할 수 있음 → sshd가 키 인증 자체를 거부

## 새 계정에 키 배포 (비번 로그인이 막힌 서버)
ssh-copy-id는 "그 계정으로 로그인 가능"이 전제라 사용 불가.
→ 관리자 계정으로 서버에 접속해 직접 배포:
  [VM] sudo mkdir -p /home/deploy/.ssh
  [VM] 공개키 내용을 authorized_keys에 기록
  [VM] sudo chown -R deploy:deploy /home/deploy/.ssh
  [VM] sudo chmod 700 /home/deploy/.ssh
  [VM] sudo chmod 600 /home/deploy/.ssh/authorized_keys
실무에선 cloud-init 또는 Ansible authorized_key 모듈로 자동화

## 인증 성공 로그 읽기
[VM] sudo systemctl status sshd 또는 journalctl -u sshd
  sshd-session[12471]: Accepted publickey for deploy from 192.168.64.1 port 49875 ssh2: ECDSA SHA256:...
  - Accepted publickey : 비번이 아닌 공개키로 인증됨
  - for deploy         : 접속 대상 계정
  - from 192.168.64.1  : 접속 출처 IP (맥)
  - ECDSA SHA256:...   : 사용된 키의 종류와 지문(fingerprint)
→ 인증 문제 추적 시 이 줄이 Accepted인지 Failed인지가 1차 단서

## 디버깅 도구
[맥] ssh -vvv user@host                      # 제시한 키와 거부 사유 추적
[맥] ssh -o PubkeyAuthentication=no user@host # 비번 로그인 차단 검증
[VM] sudo journalctl -u sshd                  # 서버 측 인증 로그
[VM] sudo sshd -t                             # 설정 문법 검사 (재시작 전 필수)

## 알아둘 문법
sudo tee 파일     # sudo로 파일에 쓸 때 >> 대신 사용
  이유: >> 는 sudo가 아닌 내 셸이 파일을 열어 Permission denied 발생
ssh -t            # 원격에서 sudo 등 터미널이 필요한 명령 실행 시
cmd1 || cmd2      # cmd1 실패 시에만 cmd2 실행
cmd1 && cmd2      # cmd1 성공 시에만 cmd2 실행
pbcopy < 파일     # (맥 전용) 파일 내용을 클립보드로 복사