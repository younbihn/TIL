# systemd

## 개념
- 리눅스 부팅 시 가장 먼저 실행되는 프로그램. PID 1 = 모든 프로세스의 조상
- 역할: 서비스(데몬) 관리 — 부팅 시 순서대로 시작, 죽으면 재시작, 로그 수집
- 비유: 서버의 매니저
- 실제 관리 대상 예: sshd, chronyd, firewalld, NetworkManager

## systemctl — systemd에게 명령하는 도구 (system control)
| 명령 | 동작 |
|---|---|
| systemctl status 서비스 | 상태 + 최근 로그 확인 (sudo 불필요) |
| systemctl start / stop | 지금 켜기 / 끄기 |
| systemctl restart | 껐다 켜기 (설정 변경 후) |
| systemctl enable | 부팅 시 자동 시작 등록 |
| systemctl disable | 자동 시작 해제 |
| systemctl enable --now | 등록 + 즉시 시작 |

⚠ start와 enable의 차이
- start  = 지금만 켜기 → 재부팅하면 꺼짐
- enable = 다음 부팅부터 자동 → 지금은 안 켜짐
→ 실무 사고 단골: enable을 빠뜨려 재부팅 후 서비스 사라짐

## 유닛 파일
위치: /etc/systemd/system/<이름>.service

[Unit]
Description=Heartbeat demo service     # status에 표시될 설명
After=network.target                    # 네트워크 준비 후 시작 (순서만 지정)

[Service]
ExecStart=/opt/heartbeat.sh             # 실행할 명령 (절대경로 필수)
Restart=on-failure                      # 비정상 종료 시 자동 재시작

[Install]
WantedBy=multi-user.target              # enable 시 다중 사용자 모드에 연결

## daemon-reload
sudo systemctl daemon-reload
- systemd는 유닛 파일을 메모리에 캐싱 → 파일을 수정해도 모름
- 유닛 파일 생성/수정 후 반드시 실행
- 빠뜨리면 "고쳤는데 왜 그대로지?" 상황 발생

## journalctl — 로그 조회 (journal control)
journalctl -u heartbeat              # -u = unit, 특정 서비스만
journalctl -u heartbeat -f           # -f = follow, 실시간 (Ctrl+C 종료)
journalctl -u heartbeat -n 20        # -n = 최근 N줄
journalctl -xeu heartbeat            # -x 설명 추가 / -e 끝으로 이동 / -u 유닛
journalctl -p err -b                 # -p err 에러 이상만 / -b 이번 부팅
journalctl --no-pager                # 화면 폭으로 잘리는 것 방지 (끝의 > 표시)

## 실험: 자가 치유 (Restart=on-failure)
1. systemctl status로 Main PID 확인
2. sudo kill <PID> 로 프로세스 종료
3. status 재확인 → 다시 running, PID는 새 번호
→ Kubernetes의 self-healing과 동일한 개념

## 스크립트 작성 시 알게 된 문법
sudo tee 파일 > /dev/null <<'EOF' ... EOF
- 히어독(heredoc): EOF가 나올 때까지 여러 줄을 입력으로 전달
- 'EOF'처럼 따옴표를 붙이면 $(date) 등을 해석하지 않고 글자 그대로 기록
- > /dev/null : tee가 화면에도 출력하는 것을 버림
#!/bin/bash = 셔뱅(shebang), 이 파일을 실행할 프로그램 지정
chmod +x 로 실행 권한을 줘야 동작