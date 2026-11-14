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

## 해결 (적용 전 제안, 결과 미확인)
플레이북의 다운로드 앞에 도구 설치 태스크를 넣는다. `.tar.gz` 이므로 `gzip` 도 함께 지정한다(`gzip` 이 이미 있는지는 확인하지 않음).
```yaml
    - name: 압축 해제 도구 설치 (최소 설치에는 tar 가 없다)
      ansible.builtin.dnf:
        name: [tar, gzip]
        state: present
      tags: [install]
```
이미 설치된 서버에서는 아무것도 하지 않으므로(멱등) 재실행이 안전하다. 앞 실행이 만든 계정과 압축 파일도 두 번째 실행에서 ok 로 지나갈 것으로 예상한다(미확인).

## 참조
- 같은 계열: 대상 서버에 모듈이 쓰는 도구와 라이브러리가 있어야 한다. 11/11 `python3-libselinux` 누락, 11/14 SELinux 플레이북의 `policycoreutils-python-utils` 설치 태스크.
- 확인 명령: `ssh lab02 'which tar gzip'`, `rpm -q tar gzip`
- 모듈이 어떤 도구를 쓰는지 알려면 모듈 문서의 Requirements 를 본다(`ansible-doc ansible.builtin.unarchive`, 이번에 실행하지 않음).
