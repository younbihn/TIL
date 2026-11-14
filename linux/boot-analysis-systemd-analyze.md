# systemd-analyze 로 읽은 VM 부팅 시간과 UEFI 확인 (lab01)

## 측정 조건
- 일시: 2026-11-14 15시대(정확한 시각은 기록 못 함), lab01(192.168.64.2, Rocky 9 aarch64, UTM VM), 커널 5.14.0-687.66.1.el9_8.aarch64, 한 번 실행
- 명령: `ls /sys/firmware/efi | head -3`, `cat /proc/cmdline`, `systemctl get-default`, `systemd-analyze`, `systemd-analyze blame | head -5`, `findmnt /boot /boot/efi`
- 참고: 첫 줄에 `🖥️ VM lab01`(자료의 위치 표시)을 같이 붙여 넣어 `명령어를 찾을 수 없음` 이 한 번 나왔다. 측정과 무관

## 결과
- `/sys/firmware/efi` 에 `efivars`, `fw_platform_size`, `mok-variables` → UEFI 로 부팅했다(`fw_platform_size` 의 내용은 보지 않음)
- 커널 명령행: `BOOT_IMAGE=(hd1,gpt2)/vmlinuz-5.14.0-687.66.1.el9_8.aarch64 root=/dev/mapper/rlm-root ro crashkernel=1G-4G:256M,4G-64G:320M,64G-:576M rd.lvm.lv=rlm/root rd.lvm.lv=rlm/swap`. 루트는 LVM(`rlm-root`)
- 기본 타깃: `multi-user.target`
- `systemd-analyze`: 커널 0.787초 + initrd 2.452초 + userspace 3.405초 = **6.645초**, `multi-user.target` 은 userspace 시작 후 2.078초에 도달
- `systemd-analyze blame | head -5`: `ttyS3.device`, `dev-ttyS3.device`, `dev-ttyS1.device`, `sys-devices-platform-serial8250-tty-ttyS1.device`, `...ttyS0.device` 가 각 약 2.83초
- `findmnt /boot /boot/efi`: **출력 없음**

## 읽은 것
- 부팅은 커널, initrd, userspace 세 단계로 나뉘고 가장 긴 것은 userspace 3.4초다. VM 이고 서비스가 적은 minimal 설치라 짧은 편으로 보이나 다른 서버나 물리 서버와 비교하지 않았다
- `blame` 의 상위가 직렬 포트 device 유닛인 것은 UTM 이 만든 가상 직렬 장치가 나타나기를 기다린 시간으로 보인다. 서비스가 느린 것이 아니며 부팅 경로를 늦췄는지는 `systemd-analyze critical-chain` 으로 봐야 한다(실행하지 않음, 미확인)
- `findmnt /boot /boot/efi` 가 조용했던 이유는 `findmnt` 가 인자 두 개를 "원본 대상" 짝으로 해석하기 때문으로 보인다(확인하지 않음). 두 마운트는 `findmnt /boot`, `findmnt /boot/efi` 로 따로 조회한다. lab02 의 `df -h` 에서는 `/boot` 가 `/dev/vda2`(960M), `/boot/efi` 가 `/dev/vda1`(599M)로 보였다(lab01 은 확인하지 않음, 같은 복제 이미지라 같을 것으로 짐작)
- `BOOT_IMAGE=(hd1,gpt2)` 는 GRUB 이 커널을 읽은 곳(디스크 `hd1` 의 두 번째 GPT 파티션)이다

## 한계와 미확인
- 이 시간은 커널 시작부터 userspace 완료까지다. UTM 이 VM 을 시작하는 시간, 펌웨어 단계, SSH 재접속 가능까지는 포함하지 않는다. 롤링 패치의 "재부팅 대기"(`docs/metrics.md` 8절, 18.25초, reboot 태스크가 복귀를 기다린 시간)와 직접 비교하지 않는다
- 한 번, 한 대
