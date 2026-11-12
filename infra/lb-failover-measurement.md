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
