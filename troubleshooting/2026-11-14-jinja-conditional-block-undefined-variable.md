# 템플릿의 조건부 블록을 켜자 `'lb_l4_port' is undefined` 로 설정 배포가 실패했다 (2026-11-14)

## 증상
맥에서 L4 비교 실험용 블록을 켜려고 실행했다.
```
ansible-playbook -i inventory/hosts.yml playbooks/deploy-lb.yml -K -e lb_l4_demo=true --tags config,firewall
```
`haproxy 설정 배포` 태스크가 실패했다.
```
[ERROR]: Task failed: 'lb_l4_port' is undefined
Origin: playbooks/deploy-lb.yml:13:7 ... templates/haproxy.cfg.j2
PLAY RECAP: lb01 ok=1 changed=0 failed=1
```

## 원인
- 템플릿 끝의 `{% if lb_l4_demo | bool %} … {% endif %}` 블록 안에서 `{{ lb_l4_port }}` 를 쓰는데, 그 변수를 `inventory/group_vars/loadbalancers.yml` 에 아직 추가하지 않았다.
- `lb_l4_demo` 는 명령줄 `-e` 로 넘겨서 참이 되었고, 그래서 블록 안으로 들어가 정의 없는 변수를 만났다. 평소처럼 `lb_l4_demo` 가 거짓이면 블록을 건너뛰어 같은 누락이 드러나지 않았을 것이다(켜 보기 전에는 모르는 종류의 오류).
- 실패한 곳은 설정 파일을 만드는 렌더링 단계라 lb01 의 `haproxy.cfg` 는 바뀌지 않았다(`changed=0`). `validate: 'haproxy -c -f %s'` 도 실행되기 전이었다.

## 해결
`inventory/group_vars/loadbalancers.yml` 끝에 두 변수를 추가한다.
```
lb_l4_demo: false
lb_l4_port: 81
```
같은 명령을 다시 실행해 성공했다(11/14 14:28 맥, 대상 lb01 한 대, 한 번 실행): `haproxy 설정 배포` changed, `방화벽 L4 비교 실험 포트` changed, `reload haproxy` 핸들러 changed, PLAY RECAP ok=6 changed=3 failed=0. 이 실행은 배포까지만 확인한 것이고 포트 81 의 응답은 보지 못했다(미확인).

## 참조
- 같은 작업에서 함께 잡은 두 가지. 학습 자료의 코드 조각에 `{% endif %}` 가 빠져 있었다(없으면 `TemplateSyntaxError: Unexpected end of template`, Jinja2 로 재현). 그리고 `{% if lb_l4_demo %}` 처럼 필터 없이 쓰면 `-e lb_l4_demo=false` 의 문자열 `"false"` 가 참으로 취급되므로 `| bool` 을 붙였다(Jinja2 로 재현).
- 빈 줄과 줄바꿈 처리는 Ansible 의 기본 `trim_blocks` 설정 가정으로 이 환경에서 렌더링 확인했다. HAProxy 자체의 문법 검사는 배포 때 `validate:` 가 lb01 에서 한다.
- 조건부 블록을 추가할 때는 블록 안의 변수를 모두 정의한 뒤에 켜 본다. 확인 도구: `ansible-playbook … --syntax-check`(변수 누락은 못 잡음), `--check --diff`(렌더링 결과 확인).
- 같은 날 기록: `ansible/lint-rules.md`, `network/lb-latency-direct-vs-lb.md`
