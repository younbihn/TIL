# SSH

## 서비스 켜기
sudo systemctl enable --now sshd
- enable: 부팅 시 자동 시작 등록 / --now: 지금 즉시 시작

## 키 기반 로그인
ssh-keygen -t ed25519        # 키 쌍 생성 (맥)
ssh-copy-id user@host        # 공개키를 서버 ~/.ssh/authorized_keys에 등록
→ 이후 비밀번호 없이 로그인. 서버 접근의 표준 방식.