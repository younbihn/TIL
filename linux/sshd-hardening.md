# sshd 보안 설정

| 설정 | 값 | 이유 |
|---|---|---|
| PermitRootLogin | no | root 직접 접속 차단. 계정명 추측이 불필요해지는 것을 막고 작업 추적성을 확보 |
| PasswordAuthentication | no | 비밀번호 무작위 대입 공격 원천 차단 |
| PermitEmptyPasswords | no | 빈 비밀번호 로그인 차단 (기본값 no이나 명시) |
| MaxAuthTries | 3 | 한 연결당 인증 시도 횟수 제한 |
| X11Forwarding | no | GUI 전달. 서버에 불필요하며 공격 표면만 증가 |
| ClientAliveInterval | 300 | 클라이언트 생존 확인 주기(초) |
| ClientAliveCountMax | 2 | 무응답 2회 시 세션 종료. 방치 세션 정리 |

원칙: 일반 계정으로 접속해 sudo로 권한을 올린다. root 직접 접속은 막는다.

## 적용 전 필수 확인
```bash
[맥] ssh -o PasswordAuthentication=no minkyluv@<IP> 'echo ok'
```
키 인증만으로 접속되는지 확인한다. 실패한 상태로 적용하면 잠긴다.
→ 플레이북에서는 stat + assert 로 코드가 검사하게 만들 수 있다.

## 적용 후 검증
```bash
[맥] ssh minkyluv@<IP>                          # 새 터미널로. 기존 세션은 유지
[맥] ssh -o PubkeyAuthentication=no minkyluv@<IP>  # Permission denied (publickey) 나와야 정상
[맥] ssh root@<IP>                              # 거부되어야 정상
```
에러 괄호의 인증 방식 목록에서 `password` 가 사라졌는지 확인한다.

## 검사와 복구
```bash
[VM] sudo sshd -t     # 문법 검사 (재시작 전 필수)
[VM] sudo sshd -T     # 실제 적용 중인 전체 설정 출력
```
최악의 경우 UTM 콘솔로 진입해 설정을 복구한다.
설정 변경 후에는 기존 SSH 세션을 끊지 말고 새 터미널로 접속을 테스트한다.
