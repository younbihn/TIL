# Prometheus 첫 기동 확인과 첫 PromQL 결과 (우리 VM 실측)

## 측정 조건
- 2026-11-14 15:52~15:54, 💻 맥(colima) 위 compose 로 Prometheus v3.5.0, Alertmanager v0.28.1, Grafana 11.6.0 기동
- 수집 간격 15초, VM 5대(Rocky 9 aarch64, UTM). 부하를 걸지 않은 상태로 보임(별도 확인은 안 함)
- 브라우저 localhost:9090 Graph 탭의 instant query. 평가 시각은 화면에 안 나와 기록하지 못함. 1회 측정이라 변동 폭은 모름

## 수집 대상 상태 (기동 직후 약 40초 뒤)
| job | 대상 | 상태 | 비고 |
|---|---|---|---|
| node | 192.168.64.2 / .3 / .4 / .5 / .7 (:9100) | 전부 up | 컨테이너에서 VM 으로 수집 성공. 출발지 주소가 192.168.64.1 로 보이는지는 미확인 |
| prometheus | localhost:9090 | up | |
| nginx | 192.168.64.2 / .3 (:9113) | down | nginx exporter 미배포(예상된 결과) |
| first-come-api | 192.168.64.4:8000 | down | 오류 문구가 잘려 원인 미확인. 앱 쪽 방화벽 또는 metrics 미배포 추정 |

## CPU 사용률  `1 - avg by (host) (rate(node_cpu_seconds_total{mode="idle"}[1m]))`
| host | 값 | % |
|---|---|---|
| lab01 | 0.00731 | 0.73 |
| lab02 | 0.00578 | 0.58 |
| lb01 | 0.00833 | 0.83 |
| app01 | 0.01107 | 1.11 |
| db01 | 0.01462 | 1.46 |

## 메모리 사용률  `1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes`
| host | 값 | % |
|---|---|---|
| lab01 | 0.2320 | 23.2 |
| lab02 | 0.2314 | 23.1 |
| lb01 | 0.3902 | 39.0 |
| app01 | 0.2067 | 20.7 |
| db01 | 0.3681 | 36.8 |

## 읽을 점
- `avg by (host)` 를 쓰면 결과에서 `instance`, `job`, `tier` 가 사라지고 host 만 남는다. 메모리 식은 `by` 가 없어 레이블이 그대로 남는다.
- 유휴 상태의 CPU 는 1% 안팎. 알람 기준(CpuBusyHigh 90%)과 큰 거리가 있어 임계값 검증에는 부하가 필요하다(1-14 k6).
- 메모리는 lb01, db01 이 다른 서버보다 높다. 원인은 확인하지 않았다(서비스 구성 차이 추정).
- 위 값은 1회 측정이다. 문서(09절)의 CPU 70%, 메모리 11% 는 다른 환경 값이라 쓰지 않는다.
