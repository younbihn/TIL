# HAProxy 이중화 무중단 측정: lab01 nginx 중지 (2026-11-12)

웹 두 대(lab01, lab02) 앞에 lb01(HAProxy, roundrobin, 헬스체크 `/health`)을 두고, 한 대의 nginx를 멈췄을 때 사용자 요청이 끊기는지 쟀다.

## 측정 조건
- 측정 시작 15:22:03 (KST, 파일 이름 기준), 맥에서 `http://192.168.64.7/health` 로 0.2초 간격 60초
- 도구: `scripts/failover_probe.py` (파이썬 표준 라이브러리만 사용, 응답 본문 `ok lab01` / `ok lab02` 로 누가 답했는지 기록)
- 장애 주입: lab01 에서 `sudo systemctl stop nginx`, 이후 `sudo systemctl start nginx` (연결 거부 방식)
- HAProxy 설정: balance roundrobin, 헬스체크 inter 2s / fall 3 / rise 2 (문서 기준값. 적용된 값 확인은 미확인)

## 결과
| 항목 | 값 |
|---|---|
| 요청 수 | 296 |
| 성공 / 실패 | 296 / 0 (성공률 100.00%) |
| 응답 시간 | 중앙값 7 ms, p95 11 ms, 최대 39 ms |
| 실패 구간 | 없음 |

구간별 응답 수 (5초 단위, 일부):

| 구간 | ok lab01 | ok lab02 | 실패 |
|---|---|---|---|
| 20~25 s | 12 | 13 | 0 |
| 25~30 s | 2 | 22 | 0 |
| 30~35 s | 0 | 25 | 0 |
| 35~40 s | 0 | 25 | 0 |
| 40~45 s | 3 | 21 | 0 |
| 45~50 s | 12 | 13 | 0 |

## 해석
- lab01 이 멈춘 약 15초 동안(표에서 25~45 s 사이) lab01 응답이 0으로 줄었고 lab02 가 모두 받았다. 그런데도 실패는 0건이었다. 연결 거부는 즉시 돌아오므로 HAProxy가 같은 요청을 lab02 로 다시 보낸 것으로 해석한다(retries, redispatch 설정 영향은 확인하지 않았다).
- 복구 후 lab01 응답이 3건으로 돌아왔다가 다음 구간(45~50 s)에 12건으로 원래 비율이 됐다.

## HAProxy 로그 (lb01, `journalctl -u haproxy`, 2026-11-12 KST)
| 시각 | 측정 시작(15:22:03) 기준 | 내용 |
|---|---|---|
| 15:22:33 | 약 +30초 | `Server web_pool/lab01 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 14ms. 1 active ... left` |
| 15:22:47 | 약 +44초 | `Server web_pool/lab01 is UP, reason: Layer7 check passed, code: 200, check duration: 3ms. 2 active ... online` |

- 측정 시작 시각은 결과 파일 이름(152203) 기준이라 초 단위 오차가 있다.
- DOWN 사유가 연결 거부(Layer4)인 것은 nginx 프로세스를 멈춘 시나리오와 맞고, UP은 `/health` 가 200 을 돌려준 L7 검사로 판정됐다.
- 표를 보면 lab01 응답이 줄기 시작한 시점(약 +26초)이 DOWN 판정(약 +30초)보다 앞선다. 즉 DOWN 판정 전 약 3~4초 동안 요청이 이미 멈춘 lab01 로 배정될 수 있었는데도 실패는 0건이었다. 같은 요청이 lab02 로 다시 보내진 것으로 해석한다(추정).

## 미확인
- 실제 중지와 시작 시각: 계획은 시작 10초 뒤 중지였으나, 표로 보면 실제 중지는 약 25~26초 부근이다. `date` 출력은 기록하지 못했다.
- 감지 지연: 중지 시각이 없어 측정값이 아니다. 표 기준 추정은 약 3~4초이고, 설계값(inter 2s × fall 3)의 이론값 4~6초보다 짧다. 중지 시각 추정 오차이거나 적용된 헬스체크 값이 문서와 다를 수 있어 `templates/haproxy.cfg.j2` 로 확인이 필요하다.
- 시나리오 2 (UTM에서 lab01 VM 일시정지, 무응답): 측정 전
- 한계: `/health` 만 호출한 결과이며 `/api/` 경로의 장애는 이 이중화로 막지 못한다. LB 자신(lb01)은 여전히 단일 장애점이다.

## 배운 점
- `def main()` 만 있고 `if __name__ == "__main__": main()` 이 없으면 오류도 출력도 없이 종료코드 0 으로 끝난다.
- `ssh 서버 'sudo 명령'` 은 터미널이 없어 비밀번호를 못 묻는다. `ssh -t` 를 쓰거나 접속한 세션에서 `sudo -v` 로 미리 인증한다.

---

# 시나리오 2: lab01 VM 일시정지 (무응답, 2026-11-12 15:48 무렵)

## 측정 조건
- 시나리오 1과 같음: 맥에서 `http://192.168.64.7/health` 로 0.2초 간격 60초, `scripts/failover_probe.py`
- 장애 주입: UTM 에서 lab01 VM 을 Pause 하고 이후 Resume (연결 시도에 거부도 응답도 오지 않는 방식)
- 측정 시작·일시정지·재개 시각은 기록하지 못했다. 프로브 출력과 HAProxy 로그로 추정한다.

## 결과
| 항목 | 값 |
|---|---|
| 요청 수 | 240 (프로브가 순차 요청이라 지연된 동안 보내지 못해 시나리오 1의 296보다 적다) |
| 성공 / 실패 | 240 / 0 (성공률 100.00%) |
| 응답 시간 | 중앙값 5 ms, p95 9 ms, 최대 3012 ms |
| 3초대 요청 | 4건 (측정 시작 +26.2, +29.2, +32.2, +35.2초, 각각 3.01초, 모두 `ok lab02` 응답) |

구간별 응답 수 (5초 단위, ok lab01 / ok lab02 / 실패): 20~25 s 13/12/0 · 25~30 s 2/5/0 · 30~35 s 0/1/0 · 35~40 s 0/10/0 · 40~45 s 0/25/0 · 55~60 s 0/24/0

## HAProxy 로그 (lb01)
- 15:48:24 `Server web_pool/lab01 is DOWN, reason: Layer4 timeout, check duration: 2006ms. 1 active and 0 backup servers left. 1 sessions active`
- 15:48:52 `Server web_pool/lab01 is UP, reason: Layer7 check passed, code: 200, check duration: 2ms. 2 active and 0 backup servers online.`

## 해석
- 일시정지된 lab01 은 연결을 거부하지도 응답하지도 않으므로 요청이 3초 동안 대기한 뒤 lab02 로 다시 보내졌다. 3.01초는 HAProxy 의 connect 타임아웃(설정 확인 필요, 3초로 추정)과 맞는다. 사용자가 본 결과는 실패가 아니라 "3초 지연"이다.
- 시나리오 1(연결 거부)은 최대 39 ms 였고 이번은 3012 ms 였다. 같은 서버 한 대의 장애여도 프로세스가 죽는 경우와 서버가 통째로 사라지는 경우의 사용자 경험이 다르다.
- p95 가 9 ms 로 보이는 것은 지연이 4건(1.7%)뿐이어서다. 이런 짧은 장애는 p95 에 안 잡히고 최대값과 지연 요청 목록에서 보인다.
- DOWN 사유가 시나리오 1은 Layer4 connection refused(즉시), 이번은 Layer4 timeout(검사 하나가 2초)이다.
- DOWN 판정 때 활성 세션이 1개였고, 이는 마지막 지연 요청(+35.2초에 시작)이 진행 중이었던 것과 맞는다. 이로부터 DOWN 은 측정 시작 약 35~38초, 일시정지는 약 26초로 보면 감지 지연은 약 9~12초로 추정된다(측정값 아님). 시나리오 1(약 3~4초)보다 긴 이유는 검사 하나가 타임아웃(약 2초)까지 걸리기 때문일 수 있으나 확인하지 않았다.

## 미확인과 한계
- 일시정지와 재개 시각, 측정 시작 시각(결과 파일 이름 기준 확인 필요), 감지 지연의 정확한 값
- 복구: UP 로그(15:48:52)는 프로브가 끝난 뒤였다(60초 구간 내 lab01 응답 0). 재개 후 복귀 과정은 프로브로 관측하지 못했다.
- 프로브가 순차 요청이라 동시 접속자가 있을 때 몇 명이 3초 지연을 겪는지는 이 결과로 알 수 없다. DOWN 판정 전 lab01 로 배정된 요청은 모두 3초를 기다렸을 것이다.
- 적용된 HAProxy 타임아웃과 헬스체크 값은 `templates/haproxy.cfg.j2` 로 확인 필요

---

# 적용된 HAProxy 설정 확인 (templates/haproxy.cfg.j2, 2026-11-12)

```
option redispatch
retries 3
timeout connect 3s
timeout check 3s
default-server inter {{ lb_check_inter }} fall {{ lb_fall }} rise {{ lb_rise }}
balance {{ lb_balance }}
```
(`timeout client/server 30s`, `http-request 10s`, `http-keep-alive 10s` 도 설정됨. `lb_*` 변수의 실제 값은 `inventory/group_vars/loadbalancers.yml` 에서 확인 필요)

## 설정으로 설명되는 것
- **시나리오 2의 3.01초**: `timeout connect 3s` 와 일치한다. 응답 없는 lab01 로 연결을 시도하다 3초에 타임아웃이 나고, `retries 3` 와 `option redispatch` 로 재시도가 lab02 로 가서 곧바로 성공한 모양이다.
- **시나리오 1의 실패 0건**: 연결 거부는 즉시 돌아오므로 같은 재시도 경로로 lab02 에 갔다. 응답이 지연 없이(최대 39 ms) 끝난 것과 맞는다.
- **DOWN 로그의 `check duration: 2006ms`**: HAProxy 문서 기준으로 헬스체크의 연결 대기 시간은 `min(timeout connect, inter)` 로 알고 있어, `inter` 가 2초라면 2006 ms 와 맞는다. `lb_check_inter` 값 확인과 문서 재확인이 필요하다(미확인).

## 아직 설명이 안 된 것
- 감지 지연 약 9~12초(시나리오 2, 추정): 검사가 타임아웃(약 2초)까지 걸리는 데다 다음 검사까지 간격이 더해졌다고 가정하면 `fall 3` 일 때 10~12초가 나온다. 다음 검사 시점이 이전 검사 시작 기준인지 종료 기준인지는 확인하지 않았다.
- 감지 지연 약 3~4초(시나리오 1, 추정): 이론값 4~6초보다 1초쯤 짧다. 중지 시각을 기록하지 못했고 구간 표로 역산한 값이라 오차 범위로 본다.
- HAProxy 버전(`haproxy -v`)
