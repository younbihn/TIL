# SELinux 라벨 때문에 403: AVC 로그로 확정하고 restorecon 으로 고치기 (lab02)

## 측정 조건
- 일시: 2026-11-14 15:17 KST, lab02(192.168.64.3, Rocky 9 aarch64), `getenforce` = Enforcing, nginx(`httpd_t`), 한 번 실행
- 절차: `echo "selinux lab" > ~/lab-label.html` → `sudo mv ~/lab-label.html /usr/share/nginx/html/lab-label.html` → `ls -lZ` → `curl -s -o /dev/null -w '%{http_code}\n' http://localhost/lab-label.html`

## 결과
| 단계 | 출력 |
|---|---|
| `ls -lZ` | `-rw-r--r--. minkyluv minkyluv unconfined_u:object_r:user_home_t:s0 12 11월 14 15:17 ...` |
| `curl` | `403` |
| `nginx error.log` | `15:17:55 [error] open() "/usr/share/nginx/html/lab-label.html" failed (13: Permission denied), client: 127.0.0.1 ...` |
| `ausearch -m AVC -ts recent` | `avc: denied { read } for pid=14450 comm="nginx" name="lab-label.html" dev="dm-0" ino=34380239 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0` |
| `setenforce 0` 후 `curl` | `200` (이후 `setenforce 1`) |
| `restorecon -Rv /usr/share/nginx/html` | `Relabeled /usr/share/nginx/html/lab-label.html from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0` |
| `curl` | `200` |

## 읽은 것
- 파일 모드(644)는 열려 있고 소유자와 무관하게 읽혀야 하는데 403 이었다. 오류 로그의 `(13: Permission denied)` 와 감사 로그의 `denied { read }` 가 같은 순간(15:17:55)이고 `permissive=0` 이 강제 모드임을 보여 준다. `scontext` 의 `httpd_t`(nginx)가 `tcontext` 의 `user_home_t`(홈 라벨)를 읽지 못한 것이 원인이다
- `setenforce 0` 에서 200 이 나온 것은 "SELinux 가 막은 것이다"를 확정하는 증거다. 이 변경은 증거 수집용이며 해결이 아니다. 해결은 정책을 건드리지 않는 `restorecon` 이었다
- `mv` 는 파일의 라벨을 그대로 옮기므로 홈의 `user_home_t` 가 따라왔다. 경로 규칙(웹 콘텐츠 디렉터리 = `httpd_sys_content_t`)이 이미 있으므로 규칙 쪽을 만들 필요 없이 파일을 규칙에 맞춘 것이다
- 해결 선택 순서(학습 자료): 라벨 복원 → 경로 규칙 등록 → boolean → 포트 등록 → 사용자 정의 모듈(마지막). 이번 건은 첫 단계다
- 같은 세션의 포트 목록: `semanage port -l | grep -w http_port_t` → `tcp 80, 81, 443, 488, 8008, 8009, 8443, 9000`. 8088 시험 추가(`semanage port -a`)는 오류 없이 실행됐으나 추가 뒤 목록 출력이 붙여 넣기에서 잘렸고, 이어서 한 삭제(`-d`)의 완료도 확인하지 못했다(미확인). 이 목록에 81, 8008 이 있다는 것은 학습 자료가 L4 실험에서 포트 81 이 가능했던 이유로 든 것이며, lb01 의 HAProxy 에는 해당 domain 의 정책이 따로 적용되므로 같은 설명이 되는지는 확인하지 않았다(미확인)

## 한계와 미확인
- `setenforce 1` 뒤 `getenforce` 출력을 받지 못했다(미확인). 이후 `restorecon` 과 `curl` 200 이 Enforcing 복귀 후의 결과로 보이나 직접 확인한 것은 아니다
- 감사 로그의 `uid=996` 이 nginx 사용자인지는 확인하지 않았다
- 수동 확인 하나뿐이며, `fixfiles` 로 전체 재라벨했을 때 규칙이 유지되는지는 시험하지 않았다
