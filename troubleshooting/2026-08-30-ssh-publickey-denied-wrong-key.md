## SSH 키 등록했으나 publickey 거부

**증상**
[맥] $ ssh deploy@192.168.64.2
deploy@192.168.64.2: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
- [VM] authorized_keys 존재, 권한 600, 소유자 deploy로 정상

**원인**
authorized_keys에 등록된 키가 클라이언트(맥)의 공개키가 아니었음.
[VM]의 관리자 계정 authorized_keys를 복사했는데, 그 안에는 VM에서 생성된
ssh-ed25519 키가 있었고, [맥]이 제시하는 키는 ecdsa-sha2-nistp256이었음.
→ 서버가 인정하는 목록에 클라이언트 키가 없어 거부.

**해결**
[맥] cat ~/.ssh/id_ecdsa.pub 로 내 공개키 확인
[VM] 해당 내용으로 authorized_keys 덮어쓰기 후 chown/chmod 재설정
검증: journalctl 로그에 "Accepted publickey ... ECDSA" 확인

**참조**
- man sshd (AUTHORIZED_KEYS FILE FORMAT)

**교훈**
- 키 인증 실패 시 권한보다 "양쪽 키가 같은지"를 먼저 대조
- 대조 방법: 앞부분(알고리즘명)과 끝부분(주석)을 비교
- 에러 괄호 (publickey,gssapi...) = 서버가 허용하는 인증 방식 목록.
  여기에 password가 없으면 비번 인증이 꺼진 상태