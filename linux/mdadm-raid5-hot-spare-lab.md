# mdadm RAID5 실습: 핫스페어 자동 복구와 장애 중 읽기 (lab02, 루프 장치 4개)

## 측정 조건
- 일시: 2026-11-14 15시대(정확한 시각은 기록 못 함), lab02(`192.168.64.3`, Rocky 9 aarch64, VM 한 대 안), mdadm 4.4-4.el9_8.2 (aarch64, baseos, 425 kB). 설치로 `mdmonitor.service` 와 `raid-check.timer` 가 활성화 링크로 생성됨
- 장치: `sudo truncate -s 256M /var/tmp/d{1..4}.img` → `sudo losetup -f --show` 로 `/dev/loop0` ~ `/dev/loop3`. 가상 디스크 위의 loop 장치라 실제 디스크 장애, 성능과 무관
- 생성: `sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 --spare-devices=1 /dev/loop0 /dev/loop1 /dev/loop2 /dev/loop3`. mdadm 이 "write-intent bitmap 을 켜겠느냐"고 물어 `y`. 메타데이터 1.2
- 파일시스템: `sudo mkfs.xfs /dev/md0` → `/mnt/raid` 마운트 → `yes "raid test" | head -c 50M > /mnt/raid/data.bin` → `sha256sum` 저장
- 장애: `sudo mdadm /dev/md0 --fail /dev/loop1` 직후 해시 비교, 이어서 `--remove`. 한 번 실행

## 결과
- 생성 직후 `/proc/mdstat`: `md0 : active raid5 loop2[4](S) loop3[3](S) loop1[1] loop0[0]`, 520192 blocks, level 5, 512k chunk, algorithm 2, `[3/2] [UU_]`, `bitmap: 1/1 pages [4KB], 65536KB chunk`. 활성 3 + 예비 1 으로 만들었는데 이 순간에는 loop2, loop3 둘 다 `(S)`로 보인다. RAID5 는 만들 때 마지막 장치로 패리티를 채우는 초기 복구를 하므로 그 과정으로 보이나 확인하지 않았다(미확인)
- 용량: 520192 KiB(약 508 MiB). 구성원 256 MiB 짜리 3 개에서 패리티 1 개 몫을 뺀 크기에 가깝다(메타데이터와 비트맵 몫으로 조금 작음, 계산은 대략)
- `mkfs.xfs`: `small data volume, ignoring data volume stripe unit 1024 and stripe width 2048` 경고, 결과는 `sunit=0 swidth=0`, `blocks=130048`(블록 4096 바이트, 약 508 MiB), `agcount=4`. md 가 알려 준 스트라이프 정보(512k 청크, 데이터 디스크 2 개)를 XFS 가 볼륨이 작아서 무시했다는 뜻
- 쓰기: 50 MiB, `sha256` `d2b7928d23bbc85876367c723e0ff92fb207e403580bea31172617e86bdd8851`
- `--fail /dev/loop1` 직후:
  - mdstat: `loop2[4] loop3[3](S) loop1[1](F) loop0[0]`, `[3/2] [U_U]`
  - **`sha256sum … | diff - before.sha` → `데이터 동일`** (장애 상태에서 읽은 해시가 장애 전과 같음)
  - `mdadm --detail`: `State : clean, degraded, recovering`, `Failed Devices : 1`, `Spare Devices : 1`, `Rebuild Status : 0% complete`
- `--remove /dev/loop1`: `hot removed /dev/loop1 from /dev/md0`. 이때 mdstat: `loop2[4] loop3[3] loop0[0]`, `[3/3] [UUU]`. 예비였던 loop3 가 정식 구성원이 되어 복구가 끝난 상태

## 읽은 것
- RAID5 는 디스크 한 개가 빠져도 나머지 디스크의 데이터와 패리티로 읽는다. 장애 직후에도 해시가 같았다
- 예비 디스크(`(S)`)가 있으면 장애 즉시 자동으로 복구(`recovering`)가 시작된다. `Rebuild Status 0%` 에서 시작해 다음 확인 때 `[UUU]` 가 되어 있었다
- 복구에 걸린 시간은 재지 않았다(`--fail` 과 `--remove` 사이의 시간은 기록 못 함, 미확인). 구성원이 작은 가상 디스크라 실제 디스크의 복구 시간과 직접 비교할 수 없다
- `[UU_]`, `[U_U]` 의 `_` 는 빠져 있는 칸이고 순서는 RAID 슬롯 번호다. 장애 전 `[UU_]` 와 장애 후 `[U_U]` 에서 빠진 칸이 달라진 것은 각각 초기 복구 중인 슬롯과 `loop1` 이 있던 슬롯이 다르기 때문으로 보이나 `mdadm --detail` 의 슬롯 표를 받지 못해 확인하지 못했다(미확인)

## 한계와 미확인
- 장애 중 쓰기, 복구 중 두 번째 장애(RAID5 는 견디지 못한다), 재부팅 후 자동 조립(`/etc/mdadm.conf` 를 만들지 않음)은 시험하지 않았다
- `--detail` 의 줄 전체 출력은 `grep` 으로 걸러져 구성원 표가 비어 있다. 복구 완료 시각과 `mdmonitor` 의 메일 알림 설정은 확인하지 않았다
- 정리 순서(`--remove` → `umount` → `--stop` → `--zero-superblock` → `losetup -d` → 이미지 삭제)와 `rm` 실패는 `troubleshooting/2026-11-14-rm-operation-not-permitted-var-tmp-raid-cleanup.md` 참조
