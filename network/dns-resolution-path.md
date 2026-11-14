# 이름 해석 경로: hosts 파일이 DNS보다 우선하는 것 확인 (lab01)

## 측정 조건
- 일시: 2026-11-14 14:18 KST 무렵, 한 번 실행
- 위치: lab01 (192.168.64.2)의 셸. 자료는 lab02 에서 하라고 했으나 실제로는 lab01 에서 실행함
- 대상 이름: `lb.lab.test` (DNS 에 없는 시험용 이름), lb01 의 주소 `192.168.64.7` 로 hosts 에 임시 등록. 자료의 `192.168.64.6` 은 옛 주소라 `.7` 로 바꿔 사용
- 명령 네 개를 한꺼번에 붙여 넣어 순서대로 실행됨

## 실행한 명령과 결과
```
echo "192.168.64.7 lb.lab.test" | sudo tee -a /etc/hosts
getent hosts lb.lab.test
dig +short lb.lab.test
curl -s http://lb.lab.test/health
```
```
192.168.64.7 lb.lab.test          <- tee 가 hosts 에 쓴 줄을 그대로 출력
192.168.64.7    lb.lab.test       <- getent hosts 결과
(출력 없음)                        <- dig +short 결과
ok lab02                          <- curl 결과
```

## 명령 조각
| 조각 | 의미 |
|---|---|
| `echo "…" \| sudo tee -a /etc/hosts` | 문자열을 파이프로 `tee` 에 넘겨 파일 끝에 이어 쓴다(`-a`). `sudo echo … >> /etc/hosts` 는 리다이렉션을 sudo 밖의 내 셸이 하므로 권한 오류가 나서 이 관용구를 쓴다. `tee` 는 쓴 내용을 화면에도 출력하므로 첫 줄이 보인다 |
| `getent hosts 이름` | 시스템의 이름 해석 경로 전체를 거쳐 결과를 보여 준다. 프로그램들(curl, ssh, nginx)이 실제로 보는 결과와 같다 |
| `dig +short 이름` | DNS 서버에 직접 질의하고 답만 한 줄씩 출력한다. hosts 파일을 보지 않는다. 답이 없으면 출력이 비어 있다 |
| `curl -s http://이름/health` | 시스템 해석 경로로 이름을 IP 로 바꾼 뒤 접속한다 |

## 읽은 것
- **getent 는 hosts 의 값을 찾았다**: `192.168.64.7 lb.lab.test`. 이름 해석 경로가 DNS 보다 hosts 파일을 먼저 본다는 것과 맞는 결과다
- **dig 는 아무것도 돌려주지 않았다**: DNS 쪽에는 이 이름이 없다. dig 은 시스템 해석 경로를 거치지 않고 DNS 서버에 직접 묻기 때문에 hosts 에 넣은 값을 모른다. 같은 이름에 대해 getent 는 답하고 dig 은 답하지 않는 상황이 만들어졌다
- **curl 은 이름으로 접속에 성공했다**: 응답 `ok lab02`. 이름이 hosts 를 통해 lb01(192.168.64.7)로 풀렸고, HAProxy 가 lab02 로 보낸 응답이다(roundrobin 이라 lab01 일 수도 있었다). curl 은 getent 와 같은 경로를 쓴다는 설명과 일치한다
- **진단에 쓰는 법**: "curl 로는 되는데 dig 으로는 안 나온다(또는 반대)"를 만나면 hosts 파일을 먼저 본다. 서버를 옮겼는데 특정 PC 에서만 옛 서버로 가는 문제의 흔한 원인이다

## 정리 필요 (중요)
- 이 실습은 `/etc/hosts` 를 손으로 고쳤다(Ansible 밖의 변경). 시험이 끝나면 지워야 한다

🖥️ VM lab01
```
sudo sed -i '/lb.lab.test/d' /etc/hosts
getent hosts lb.lab.test; echo "종료 코드 $?"
```
- `sed -i '/패턴/d'` 는 패턴이 있는 줄을 제자리에서 삭제한다. 지운 뒤 `getent` 가 종료 코드 2(찾지 못함)를 돌려주면 정리된 것이다
- 정리했는지는 이 기록 시점에 확인하지 않음(미확인)

## 미확인
- `dig` 의 응답 상태(`status: NXDOMAIN` 인지 빈 응답인지): `+short` 라서 출력이 없을 뿐 상태를 보지 않았다. 오류 메시지가 없었으므로 dig 은 실행되었다고 본다
- lab01 의 `/etc/nsswitch.conf` 의 `hosts:` 줄과 `/etc/resolv.conf` 의 nameserver(어느 DNS 가 답했는지): 이번에 확인하지 않았다
- 정리(hosts 줄 삭제) 여부
- lab02 에서 같은 실습을 했을 때의 결과
