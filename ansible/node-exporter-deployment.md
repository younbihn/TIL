# node_exporter 를 Ansible 로 5대에 배포하고 lab02 에서 상태 확인 (2026-11-14)

## 측정 조건
- 일시: 2026-11-14 15:26~15:31 KST, 맥에서 `ansible-playbook -i inventory/hosts.yml playbooks/monitoring-exporters.yml -K`, 대상 `hosts: all`(app01, db01, lab01, lab02, lb01), node_exporter 1.9.1(`linux-arm64`)
- 서버 확인: 15:31 에 lab02 로 ssh 해서 `systemctl status`, `ss -tlnp`, `ls -Z`, `sudo firewall-cmd --list-rich-rules`, `curl localhost:9100/metrics | head -5`. 한 번 실행. 5대 중 lab02 에서만 직접 확인했다(나머지는 플레이북의 `메트릭이 나오는지 확인` ok 로만 확인)

## 플레이북 구조 (사용자의 파일을 `cat` 으로 받은 내용, 변수 값과 템플릿은 보지 못함)
| 태스크 | 모듈 | 핵심 |
|---|---|---|
| 실행 전용 계정 | `user` | `system: yes`, `shell: /sbin/nologin`, `create_home: no` (로그인 못 하는 서비스 계정) |
| 압축 해제 도구 | `dnf` | `tar`, `gzip` (minimal 설치에는 없음, 트러블슈팅 `2026-11-14-ansible-unarchive-tar-missing-minimal.md`) |
| 내려받기 | `get_url` | `checksum: sha256:<릴리스의 sha256sums.txt URL>` 로 내려받은 파일의 해시를 검증 |
| 압축 풀기 | `unarchive` | `remote_src: yes`(서버가 직접 푼다), `creates:` 로 이미 있으면 건너뜀 |
| 실행 파일 배치 | `copy` | `remote_src: yes`, `/usr/local/bin/node_exporter`, root 소유 0755, 바뀌면 `restart` 핸들러 알림 |
| 유닛 배포 | `template` | `/etc/systemd/system/node_exporter.service`, 바뀌면 핸들러 알림 |
| 방화벽 | `ansible.posix.firewalld` | rich rule: 모니터링 서버 주소에서 오는 지정 포트 TCP 만 허용, `permanent` + `immediate` |
| 서비스 | `systemd` | `state: started`, `enabled: yes`, `daemon_reload: yes` |
| 확인 | `uri` | `http://127.0.0.1:<포트>/metrics` 에 `node_cpu_seconds_total` 이 있어야 통과, `changed_when: false` |
- 아키텍처는 사실 수집값 `ansible_facts.architecture` 로 `arm64` 와 `amd64` 를 고른다(`aarch64` 이면 `arm64`)

## 결과
- 실제 적용(15:30:50): app01, db01, lab01, lb01 각 ok=11 changed=9 failed=0, lab02 ok=9 changed=0 skipped=1. 13초
- lab02 서비스 상태: `Loaded: /etc/systemd/system/node_exporter.service; enabled; preset: disabled`, `Active: active (running) since 15:28:23 KST`, Main PID 20779, Tasks 4, Memory 9.8M(최대 11.6M 에서 11.8M), CPU 101ms. 실행 줄: `/usr/local/bin/node_exporter --web.listen-address=:9100 --collector.systemd`
- 수신: `ss -tlnp | grep 9100` → `LISTEN 0 4096 *:9100 *:*` (모든 인터페이스에서 수신, 프로세스 이름은 `sudo` 없이 안 보임)
- 실행 파일 SELinux 라벨: `system_u:object_r:bin_t:s0 /usr/local/bin/node_exporter`
- 방화벽: `rule family="ipv4" source address="192.168.64.1" port port="9100" protocol="tcp" accept` (lab02 의 rich rule 은 이것 하나뿐)
- `curl -s localhost:9100/metrics | head -5` → `# HELP go_gc_duration_seconds ...`, `# TYPE ... summary`, `go_gc_duration_seconds{quantile="0"} 1.2542e-05` 등 프로메테우스 텍스트 형식

## 읽은 것
- **서비스 시작 시각 15:28:23 이 두 번째 실행(15:28:26, changed=0)보다 3초 앞선다.** 첫 성공 실행이 15:28:23 무렵에 끝난 것으로 보이며(그 출력은 받지 못함), 이전에 "이미 한 번 성공한 뒤의 재실행으로 보인다"고 한 추정과 일치한다
- 모니터링 수집을 허용한 주소는 `192.168.64.1`(맥)이다. 포트 `9100` 은 모든 인터페이스에서 듣고 있으므로 접근 제어는 방화벽 규칙이 한다. 서비스 자체는 인증이 없다
- 로컬 `curl` 이 규칙에 없는 주소(127.0.0.1)인데도 통과한 것은 루프백 트래픽을 방화벽이 막지 않기 때문으로 보인다(알려진 동작, 이번에 별도로 시험하지 않음). 그래서 플레이북의 `uri` 확인은 방화벽 규칙과 무관하게 동작한다. 그 말은 **이 확인이 "외부에서 수집된다"는 증명은 아니라는 뜻**이다
- 실행 파일 라벨이 `bin_t` 인 것은 `copy` 가 파일을 새로 만들어 `/usr/local/bin` 의 기본 라벨이 붙은 것으로 보인다(SELinux 가 서비스를 막지 않은 이유로 보이나 AVC 를 확인하지 않음). 홈에서 `mv` 한 파일이 홈 라벨을 그대로 가져와 403 이 났던 11/14 사례(`linux/selinux-label-403-diagnosis.md`)와 대비된다
- `--collector.systemd` 옵션은 systemd 유닛 상태 수집기를 켠다(유닛 템플릿의 줄이며 값은 사용자의 파일 내용을 직접 보지 못함)

## 한계와 미확인
- 모니터링 서버(맥)에서 `http://192.168.64.3:9100/metrics` 접속 성공, 허용되지 않은 호스트(예: lab01)에서의 차단은 시험하지 않았다. 규칙이 "수집만 허용"인지 실제로 확인되지 않았다
- 다른 4대는 `ssh` 로 상태를 직접 보지 않았다. 두 번째 전체 실행(5대 changed=0)도 아직 안 했다
- 프로메테우스 쪽 설정(수집 대상 등록)은 이 플레이북의 범위가 아니다
- 로그 줄이 `…` 로 잘려 collector 목록 전체는 보지 못했다(`systemctl status -l` 또는 `journalctl -u node_exporter` 로 볼 수 있음, 실행 안 함)
- 한 번 실행. sudo 비밀번호를 물을 때 이전 명령 묶음을 붙여 넣어 `잘못된 암호 입력 시도 1번` 이 한 번 났으나 측정과 무관
