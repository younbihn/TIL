## 복제한 VM 에 접속하자 REMOTE HOST IDENTIFICATION HAS CHANGED

**상태**: 해결. 이후 `ssh app01` 접속 정상. 단, 지문 대조를 실제로 했는지는 기록이 없음 (미확인).

**증상**
[맥] $ ssh app01
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
The fingerprint for the ED25519 key sent by the remote host is SHA256:88qthF…
Offending ED25519 key in /Users/…/.ssh/known_hosts:16
Host key for 192.168.64.4 has changed and you have requested strict checking.
Host key verification failed.

**원인**
맥의 `known_hosts` 16번째 줄에 192.168.64.4 의 **예전 호스트 키**가 저장돼 있었고, 지금 그 주소에서 응답하는 서버의 키가 달라서 접속을 거부했다. 두 가지가 겹칠 수 있다.
- base 를 만들 때 `/etc/ssh/ssh_host_*` 를 지웠으므로, 복제본(app01)은 첫 부팅 때 **새 호스트 키를 생성**한다 (의도한 동작)
- 그 IP 를 예전에 다른 VM 이 썼고 DHCP 가 이번에 app01 에 같은 주소를 줬다
- 둘 중 어느 쪽이었는지는 확인하지 못했다 (미확인)

**해결**
```bash
# 🖥️ app01 콘솔: 지금 서버의 진짜 지문 확인
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub     # 경고의 SHA256 값과 같은지 대조
# 💻 맥: 낡은 항목 삭제 후 재접속
ssh-keygen -R 192.168.64.4
ssh-keygen -R app01          # 별칭으로도 저장돼 있으면
ssh app01                    # 지문 확인 후 yes
```
- `ssh-keygen -lf <공개키>`: 호스트 키의 지문 출력. `ssh-keygen -R <호스트>`: `known_hosts` 에서 항목 삭제 (지운 줄은 `.old` 로 백업)

**참조**
- man ssh-keygen (-l, -R), man ssh (StrictHostKeyChecking)
- infra/base-image-clone.md

**교훈**
- 이 경고는 중간자 공격을 잡는 장치다. 이유를 모른 채 뜨면 지우지 말고 원인부터 확인한다. 이번엔 복제로 키가 바뀐다는 것을 알고 있었으므로 지문을 대조한 뒤 지웠다
- `known_hosts` 를 통째로 지우거나 `StrictHostKeyChecking no` 로 끄면 모든 서버의 확인 장치가 사라진다
- 복제본마다 처음 접속할 때 같은 경고가 날 수 있다
- (별개) 접속 시 보이던 `connection is not using a post-quantum key exchange algorithm` 경고는 서버 OpenSSH 가 PQ 키 교환을 지원하지 않는다는 안내로, 접속 실패와 무관하다
