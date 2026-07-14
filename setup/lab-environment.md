# 실습 환경 구축 — MacBook M1 + UTM + Rocky Linux 9

> 목적: 이 문서만 보고 실습용 VM을 처음부터 다시 만들 수 있게 한다. (runbook)
> 최초 작성: 2026-07-14

## 호스트 환경
- MacBook M1 (Apple Silicon, aarch64), RAM 16GB
- 가상화: UTM / (예정) Lima — VirtualBox+Vagrant는 Apple Silicon 미지원이라 대체

## 도구 설치
```bash
brew install --cask utm
brew install lima        # W4 서버 팜용, 미리 설치만
```

## ISO 선택
- Rocky Linux 9 **aarch64** `minimal.iso` (약 2.5GB)
- ⚠ x86_64 받으면 에뮬레이션으로 돌아가 10배 느려짐 — 반드시 aarch64
- minimal / boot / dvd 차이:
  - minimal: 설치 패키지 내장, GUI 없음 → 서버 실습 표준 ✅
  - boot: 설치 중 인터넷에서 패키지 다운로드 (네트워크 꼬이면 막힘)
  - dvd: GUI 포함 13GB, 과함

## UTM VM 생성 (lab01)
| 항목 | 값 |
|---|---|
| 모드 | **Virtualize** (Emulate 금지) |
| OS | Linux, 부팅 ISO = 위 minimal |
| RAM / CPU | 2048MB / 2코어 |
| 디스크 | 20GB |
| 이름 | lab01 |

## Rocky 설치 시 체크포인트 (Anaconda)
- Software Selection: **Minimal Install**
- Network & Host Name: 이더넷 **ON** ← 안 켜면 설치 후 인터넷 안 됨
- hostname: `lab01`
- Root 비밀번호 설정 + 사용자 생성 시 ☑ Make this user administrator
- 설치 완료 후: Reboot → **UTM 설정에서 ISO 제거** (안 하면 설치화면 재진입)

## SSH 접속 설정
VM 콘솔에서:
```bash
sudo systemctl enable --now sshd
ip a                     # IP 확인 → <VM_IP> (192.168.64.x 대역, UTM 공유 네트워크)
```
맥에서:
```bash
ssh <계정>@<VM_IP>
# 키 기반 로그인 (비밀번호 없이)
ssh-keygen -t ed25519    # 키 없을 때만
ssh-copy-id <계정>@<VM_IP>
```

## 현재 VM 목록
| VM | IP | 용도 | 생성일 |
|---|---|---|---|
| lab01 | <VM_IP> | W1~3 리눅스/네트워크 실습 (부수고 고치기용) | 2026-07-14 |

## 알려진 문제 / 팁
- IP가 안 보일 때: `sudo nmcli con up enp0s1` (설치 시 네트워크 OFF였던 경우)
- ssh 거부될 때: VM에서 `sudo systemctl status sshd` 먼저 확인