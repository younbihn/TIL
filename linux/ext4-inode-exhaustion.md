# ext4 inode 고갈: 공간이 남았는데 "장치에 남은 공간이 없음" (lab02, 루프 장치 32M)

## 측정 조건
- 일시: 2026-11-14 15:10 KST 무렵(`mkfs` 가 알려 준 생성 시각 15:10:00), lab02(`192.168.64.3`, Rocky 9 aarch64), e2fsprogs 1.46.5, 한 번 실행
- 준비: `truncate -s 32M /var/tmp/e.img` → `mkfs.ext4 -F -q -N 256 -L labfs /var/tmp/e.img` (inode 수를 일부러 256 으로 제한) → `sudo mount -o loop /var/tmp/e.img /mnt/lab`. 첫 시도는 명령을 붙여 넣는 과정에서 `mkfs` 가 두 번 실행되어(두 번째에 "contains a ext4 file system labelled 'labfs'" 안내) 같은 이미지를 다시 만든 결과다
- 실험: `/mnt/lab` 에서 `for i in $(seq 1 300); do : > f$i || { echo "실패 지점: f$i"; break; }; done` 로 빈 파일 생성, 이어서 `touch zz`

## 결과
| 항목 | 만들기 전 | 실패 후 |
|---|---|---|
| `df -h` (크기 / 사용 / 가용 / 사용률) | 28M / 14K / 25M / 1% | 28M / 18K / 25M / 1% |
| `df -i` (inode 수 / 사용 / 남음 / 사용률) | 256 / 11 / 245 / 5% | 256 / 256 / 0 / 100% |

- `dumpe2fs -h`: Inode count 256, Block count 32768, Block size 1024, Reserved block count 1638, Free inodes 245, 기능에 `has_journal`, `extent`, `metadata_csum` 등
- 실패: `f246` 에서 `장치에 남은 공간이 없음`, `실패 지점: f246`. `touch zz` 도 같은 오류
- 계산: 시작 시 남은 inode 245 개(전체 256 중 11 개는 파일시스템이 만든 `lost+found` 등이 사용, 구성은 확인 안 함) → `f1` 부터 `f245` 까지 245 개가 성공하고 246 번째에서 실패. 실제 실패 지점과 일치

## 읽은 것
- 블록(데이터 공간)은 25M 가 남아 있는데도 파일을 만들 수 없었다. ext4 는 파일 하나당 inode 하나가 필요하고 inode 개수는 `mkfs` 때 정해지므로(`-N`, 기본은 용량 대비 비율) 작은 파일이 아주 많으면 공간보다 inode 가 먼저 바닥난다
- 증상은 같은 문구(`No space left on device`)로 나오기 때문에 `df -h` 만 보면 원인을 놓친다. **`df -h` 가 여유로운데 쓰기가 실패하면 `df -i` 를 본다**
- 파일 크기는 0 바이트였으므로 사용 공간은 14K 에서 18K 로 4K 만 늘었다(디렉터리 항목이 차지한 몫으로 보이나 확인 안 함)
- 전체 28M 인 것은 이미지 32M 에서 inode 테이블, 저널 등 메타데이터 몫을 뺀 값으로 짐작하나 계산하지 않았다(미확인)

## 한계와 미확인
- inode 를 억지로 256 개로 줄인 실험이다. 실제 서버에서의 기본 비율과 고갈 사례(예: 세션 파일, 메일 스풀, 캐시 디렉터리)는 이번에 다루지 않았다
- 해결(파일 삭제 후 `df -i` 가 회복되는지)은 이어서 `rm -f f*` 를 한 것으로 보이나 그 직후의 `df -i` 출력은 받지 못했다(미확인). inode 수는 `mkfs` 로만 늘릴 수 있고 ext4 는 사후에 늘릴 수 없다(알고 있는 지식, 이번에 시험하지 않음)
- 포트폴리오에서는 "디스크 모니터링에 `df -i` 도 포함한 이유"의 근거로 쓸 수 있다. `scripts/healthcheck.py` 가 inode 사용률을 점검하는지는 확인하지 않았다(미확인)
