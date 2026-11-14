# ansible unarchive 실패: "Failed to find handler ... Unable to find required 'tar'" (2026-11-14, lab02)

## 증상
`ansible-playbook -i inventory/hosts.yml playbooks/monitoring-exporters.yml --limit lab02 -K` (15:26:25, lab02 한 대, 첫 실행. 바로 앞의 첫 시도는 사용자가 중단) 에서 앞 태스크는 성공하고 `압축 풀기` 에서 실패했다.
```
TASK [실행 전용 계정 생성]                     changed
TASK [릴리스 압축 파일 내려받기 (체크섬 검증)]   changed  (1.98초)
TASK [압축 풀기]  fatal: Failed to find handler for "/tmp/node_exporter-1.9.1.linux-arm64.tar.gz".
  Make sure the required command to extract the file is installed.
  Unable to find required 'gtar' or 'tar' binary in the path
RECAP: ok=3 changed=2 failed=1, 4초
```
실패 시점까지 lab02 에는 `node_exporter` 시스템 계정과 `/tmp/node_exporter-1.9.1.linux-arm64.tar.gz` 가 만들어졌다(두 태스크가 changed).

## 원인
`ansible.builtin.unarchive` 는 대상 서버(여기서는 `remote_src: yes` 라서 lab02 자신)의 `tar` 를 실행한다. Rocky 9 minimal 설치에는 `tar` 가 없다고 보이며, 메시지가 `gtar` 또는 `tar` 를 찾지 못했다고 한다. 다운로드는 성공했으므로 lab02 가 GitHub 에 접속할 수 있고 체크섬 검증도 통과한 것이다(체크섬 비교 결과를 따로 보지는 않았고 `get_url` 이 오류 없이 끝난 것으로 판단). 다른 서버(lab01, lb01, app01, db01)에서도 같은지는 확인하지 않았다(같은 복제 이미지라 같을 것으로 짐작, 미확인). `unzip`, `zipinfo` 문구는 zip 용 검색이라 이 건과 무관하다.

## 해결 (적용 후 재실행으로 확인, lab02 한 대)
플레이북의 다운로드 앞에 도구 설치 태스크를 넣는다. `.tar.gz` 이므로 `gzip` 도 함께 지정한다(`gzip` 이 이미 있는지는 확인하지 않음).
```yaml
    - name: 압축 해제 도구 설치 (최소 설치에는 tar 가 없다)
      ansible.builtin.dnf:
        name: [tar, gzip]
        state: present
      tags: [install]
```
이미 설치된 서버에서는 아무것도 하지 않으므로(멱등) 재실행이 안전하다.

## 확인 결과 (15:28:26, lab02, `--limit lab02 -K`)
- PLAY RECAP: ok=9 changed=0 skipped=1, 소요 6초. `압축 해제 도구 설치` ok, `릴리스 압축 파일 내려받기` ok, `압축 풀기` skipping(`creates` 경로가 이미 있어 건너뜀), 실행 파일 배치, 유닛, 방화벽, 서비스, 메트릭 확인 모두 ok
- 이 실행은 이미 한 번 끝까지 성공한 뒤의 재실행으로 보인다. 모든 태스크가 ok 이고 `압축 풀기` 가 건너뛰어졌다는 것은 파일, 유닛, 방화벽 규칙, 서비스가 이미 있었다는 뜻이다. 그 첫 성공 실행의 출력(어느 태스크가 changed 였는지, 도구 설치를 이 플레이북이 했는지 손으로 했는지)은 받지 못했다(미확인)
- 붙여 넣은 태스크 이름에 오타가 있다(`...없다)가`). 이름 끝의 `가` 를 지워야 한다. 동작에는 영향이 없다
- 바로 이어서 `--check --diff` 를 `-K` 없이 `--limit` 도 없이 실행하면 5대 모두 `sudo: 암호가 필요합니다` 로 Gathering Facts 에서 실패한다(15:24, 15:28 두 번). 위 `-K` 누락과 같은 원인이며 측정이 아니라 기록하지 않는다

## 참조
- 같은 계열: 대상 서버에 모듈이 쓰는 도구와 라이브러리가 있어야 한다. 11/11 `python3-libselinux` 누락, 11/14 SELinux 플레이북의 `policycoreutils-python-utils` 설치 태스크.
- 확인 명령: `ssh lab02 'which tar gzip'`, `rpm -q tar gzip`
- 모듈이 어떤 도구를 쓰는지 알려면 모듈 문서의 Requirements 를 본다(`ansible-doc ansible.builtin.unarchive`, 이번에 실행하지 않음).
