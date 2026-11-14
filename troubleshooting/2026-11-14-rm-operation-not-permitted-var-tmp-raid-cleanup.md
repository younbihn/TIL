# RAID 실습 정리 중 `rm` 이 "명령을 허용하지 않음"으로 실패 (2026-11-14, lab02)

## 증상
lab02 에서 loop 장치 기반 RAID5 실습을 정리하며 한꺼번에 붙여 넣은 명령 중 `rm -f /var/tmp/d?.img /var/tmp/before.sha` 가 다음 오류를 냈다.
```
rm: cannot remove '/var/tmp/d1.img': 명령을 허용하지 않음   (d2, d3, d4 도 같음)
```
같은 묶음의 `mdadm --remove`, `umount`, `mdadm --stop`, `--zero-superblock`, `losetup -d` 는 오류 없이 끝났다. `sudo rm -f` 로 다시 실행하자 오류 없이 지워졌다.

## 원인 (추정, 확인 못 함)
"명령을 허용하지 않음"은 EPERM(Operation not permitted)의 한국어 메시지다. 이미지 파일이 `sudo` 로 만들어져 root 소유였고, `/var/tmp` 는 sticky 비트가 있는 디렉터리라 파일 소유자가 아닌 일반 사용자는 지울 수 없었던 것으로 짐작한다. 파일이 이미 지워져 소유자, `ls -ld /var/tmp`, `lsattr` 는 확인하지 못했다(미확인). 파일 속성 `chattr +i` 도 같은 증상을 내므로 그 가능성도 배제하지 못한다.

## 해결
`sudo rm -f /var/tmp/d?.img /var/tmp/before.sha`. 다음에는 확인 명령을 먼저 쓴다.
```
ls -l /var/tmp/d?.img
ls -ld /var/tmp
lsattr /var/tmp/d1.img
```

## 두 번째 실행의 오류는 정상이다
첫 실행에서 이미 정리가 끝났기 때문에 같은 명령 묶음을 다시 돌리면 아래 오류가 나는 것이 맞다.
- `mdadm: error opening /dev/md0: No such file or directory`: 배열이 이미 중지됨
- `umount: /mnt/raid: not mounted.`: 이미 언마운트됨
- `Unrecognised md component device - /dev/loopN`, `losetup: detach failed: 그런 장치 혹은 주소가 없음`: loop 장치가 이미 분리되어 비어 있음
- `cat /proc/mdstat` 가 `unused devices: <none>` 만 출력: 배열이 없음

## 참조
- 정리 순서: `--fail`/`--remove` → `umount` → `mdadm --stop` → `--zero-superblock` → `losetup -d` → 이미지 삭제. 배열이 살아 있는 동안은 구성 장치를 해제할 수 없고, 이미지를 먼저 지우면 loop 장치가 남는다.
- 정리 확인: `cat /proc/mdstat`, `losetup -a`, `lsblk`, `ls /var/tmp/d?.img`. `/etc/fstab` 이나 `/etc/mdadm.conf` 에 실습 항목을 넣었다면 지워야 한다(넣었는지는 미확인).
- 교훈: 여러 줄을 한꺼번에 붙여 넣으면 한 줄의 실패가 눈에 안 띄므로, 줄마다 출력과 종료 코드를 본다(`echo $?`).
