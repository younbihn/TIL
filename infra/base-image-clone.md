# 베이스 이미지(복제 원본)와 VM 복제

> 최초 작성: 2026-11-11 (1-4 3-Tier 설계와 서버 증설)

## 왜 운영 중인 서버를 바로 복제하지 않나
- 복제는 **역할까지 복제**한다. lab01 은 웹 서버이자 NFS 서버라서 그대로 복제하면 db01 에도 nginx 가 돌고 80, NFS 포트가 열려 있다. 데이터 계층에 웹 포트가 열려 있으면 경계가 시작부터 깨진다
- 그래서 역할이 없는 깨끗한 원본(골든 이미지, 베이스 이미지)을 따로 두고 거기서만 찍어낸다. 클라우드의 AMI, VM 템플릿과 같은 개념
- 원본에는 공통 기본(계정, SSH 키, 하드닝, 시간 동기화)만 두고, 역할은 복제 후에 Ansible 이 입힌다

## 복제하면 같아지는 고유값 세 가지

| 값 | 같으면 생기는 문제 |
|---|---|
| MAC 주소 | DHCP 가 같은 IP 를 준다 (VM 클론 IP 중복 사례) |
| `/etc/machine-id` | 로그 수집, 모니터링에서 서버들이 같은 기계로 보인다 |
| SSH 호스트 키 | "이 서버가 정말 그 서버인가"를 확인하는 장치가 무의미해진다 |

- 원본에서 이 값들을 **비워 두면** 각 복제본이 첫 부팅 때 자기 것을 새로 만든다

## 만드는 절차 (lab01 에서 base 를 만들 때)
```bash
# 🖥️ UTM: lab01 을 끈 뒤 Clone → 이름 base → Network 의 MAC 을 Random 으로
# 🖥️ base: 역할 걷어내기
sudo systemctl disable --now nginx nfs-server
sudo dnf remove -y nginx
sudo truncate -s 0 /etc/exports
sudo firewall-cmd --permanent --remove-service=http --remove-service=nfs \
                  --remove-service=mountd --remove-service=rpc-bind
sudo firewall-cmd --reload
sudo firewall-cmd --list-services           # ssh 계열만 남아야 한다

# 🖥️ base: 고유값 비우고 끄기
sudo hostnamectl set-hostname base
sudo truncate -s 0 /etc/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo poweroff
```
- `truncate -s 0 /etc/machine-id`: **지우지 말고 비운다.** 비어 있으면 systemd 가 첫 부팅으로 보고 새 ID 를 만든다. 파일이 아예 없으면 일부 서비스가 오동작할 수 있다
- `rm -f /etc/ssh/ssh_host_*`: 다음 부팅 때 sshd-keygen 이 새로 만든다. 지금 이 세션은 이미 연결돼 있어서 끊기지 않는다
- `poweroff`: 재부팅이 아니라 끈다. 여기서 재부팅하면 base 가 자기 ID 와 키를 만들어 원본으로서 의미가 사라진다
- **base 는 다시 켜지 않는다.** 켜는 순간 고유값이 생긴다. 실수로 켰다면 같은 절차를 다시 하고 끈다

## 복제본(app01, db01)에서 확인할 것
```bash
sudo hostnamectl set-hostname app01
cat /etc/machine-id            # 다른 VM 과 값이 달라야 한다
ip -4 -brief addr show         # 받은 IP 기록
```
- 맥에서 `ssh app01` 로 한 번 접속해 새 호스트 키를 `known_hosts` 에 받아 둔다 (Ansible 은 yes/no 질문에 답할 수 없다)
- 복제본의 호스트 키가 새로 생성되므로, 예전에 그 IP 를 쓰던 다른 VM 의 키와 충돌하면 `REMOTE HOST IDENTIFICATION HAS CHANGED` 가 뜬다 → `troubleshooting/2026-11-11-ssh-host-key-changed-after-clone.md`

## 호스트명 바꾸기와 프롬프트
```bash
sudo hostnamectl set-hostname base
exec bash          # 새 셸. 프롬프트의 @ 뒤 이름(\h)은 셸이 시작될 때 한 번 읽는 값이다
```
- 복제 직후에는 원본의 이름이 그대로 복제돼서 `base` 가 `[user@lab01 ~]$` 로 보인다. 이름을 바꾼 뒤 새 셸을 열어야 프롬프트에 반영된다

## 메모리 설정 주의
- UTM 의 Memory 값은 MiB 정수로 입력한다 (1.5GB = 1536). 값이 0 이면 부팅 단계에서 `grub … out of memory` 가 난다 → `troubleshooting/2026-11-11-grub-out-of-memory-vm-memory-zero.md`
