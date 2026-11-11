# GitHub Actions CI

> 최초 작성: 2026-11-11 (11/09 자료 1-3 CI/CD)

## 왜 CI인가
- 지금까지는 플레이북을 고치면 맥에서 손으로 `--syntax-check`를 돌렸다
- 바쁜 날은 건너뛰게 되고, 건너뛴 날 깨진 코드가 main에 들어간다
- CI는 이 검사를 **사람의 성실함이 아니라 구조로** 보장한다
- push마다 GitHub가 깨끗한 컴퓨터(러너)를 빌려 저장소를 받아 검사하고, 결과를 커밋 옆 초록 체크 / 빨간 X로 남긴다

## 용어

| 용어 | 뜻 |
|---|---|
| CI | Continuous Integration. 올릴 때마다 자동 검사 |
| CD | 검사를 통과한 코드를 자동 배포. 이번엔 하지 않음 |
| 워크플로 | `.github/workflows/` 아래 YAML 파일 하나 |
| 잡(job) | 워크플로 안의 작업 묶음. **잡마다 새 가상 머신** |
| 스텝(step) | 잡 안의 명령 하나. 같은 머신에서 위에서 아래로 순서대로 |
| 러너 | 잡을 실제로 실행하는 머신 (ubuntu-latest) |
| 액션 | 남이 만든 재사용 스텝. `uses:` 로 부른다 |

## 이 저장소의 파이프라인 흐름
```
push / PR → GitHub 가 ci.yml 을 읽고 실행 예약
  → 러너(새 Ubuntu)
    ① checkout            러너는 빈 머신이라 저장소부터 받는다
    ② Python 3.12
    ③ ansible-lint, yamllint 설치 (버전 고정)
    ④ ansible.posix 컬렉션 설치
    ⑤ yamllint .          YAML 서식
    ⑥ ansible-lint        Ansible 작성 관례
    ⑦ ansible-playbook --syntax-check   해석 가능한가
  → 전부 통과: 초록 체크 / 하나라도 실패: 빨간 X, 그 스텝에서 멈춤
```
CI는 lab01, lab02 에 **전혀 접속하지 않는다.** 러너는 인터넷 위에 있고 VM은 맥 안의 사설망에 있다.

## ci.yml 에서 기억할 것
- 경로는 반드시 `.github/workflows/`. `workflow`(단수)로 쓰면 **오류 없이 그냥 무시**된다
- `on:` 은 언제 돌릴지. push(main), pull_request(main), workflow_dispatch(수동 버튼)
- `jobs.<id>.name` 이 화면에 보이는 이름이고, **브랜치 보호의 필수 체크 이름**이 된다 (이 저장소: `플레이북 검증`)
- `uses: actions/checkout@v4` 의 `@v4` 는 버전 고정. 안 붙이면 액션 쪽이 바뀌는 날 내가 안 건드렸는데 파이프라인이 깨진다
- `python-version: "3.12"` 는 문자열로. 따옴표 없는 `3.10` 은 YAML이 숫자 3.1 로 읽는다
- `run: |` 의 기본 셸은 `bash -e`. 한 줄이라도 실패하면 그 스텝이 실패하고 **뒤 스텝은 실행되지 않는다**
- `echo "::group::이름"` / `"::endgroup::"` 은 GitHub가 알아듣는 출력. 로그를 접히는 묶음으로 보여줘서 어느 플레이북에서 실패했는지 찾기 쉽다
- 플레이북 루프(`for pb in playbooks/*.yml`)로 돌려서 새 플레이북을 추가해도 워크플로를 고칠 필요가 없다
- `paths:` 필터는 일부러 넣지 않았다. 필수 체크를 켠 상태에서 필터에 안 걸린 PR(README만 수정 등)은 검사가 아예 안 돌아 **통과도 실패도 아닌 채로 영원히 대기**한다

## 중괄호 세 종류

| 표기 | 주인 | 값이 되는 때 |
|---|---|---|
| `{{ 변수 }}` | Ansible Jinja2 | 플레이북 실행 때 (CI 문법 검사는 대부분 계산 안 함) |
| `${{ 식 }}` | GitHub Actions | GitHub가 워크플로를 읽을 때 |
| `$VAR`, `${VAR}` | bash | 러너 셸이 그 줄을 실행할 때 |

## 맥에서는 되는데 러너에서는 실패 (collections/requirements.yml)
- 플레이북이 `ansible.posix.firewalld`, `ansible.posix.mount`, `ansible.posix.authorized_key` 를 쓴다
- 맥에는 예전에 손으로 `ansible-galaxy collection install ansible.posix` 를 해둬서 문제가 안 보인다
- 러너에는 선언된 것만 있다. 컬렉션 없이 검사하면 `syntax-check[unknown-module]` 로 실패한다
- 해결: `collections/requirements.yml` 에 선언하고 `ansible-galaxy collection install -r` 로 설치. 맥과 러너가 같은 목록을 쓴다
- 1-4 에서 `community.mysql` 을 쓰면 이 파일에 한 줄 추가해야 같은 에러를 피한다
- 검사 도구는 별도 가상환경(`~/.venvs/lint`)에 설치. ansible-lint 가 자기에게 맞는 ansible-core 를 끌고 와서 기존 Ansible 버전을 건드릴 수 있기 때문

## 초록 체크가 보증하는 범위

| 검사 | 확인된 것 | 모르는 것 |
|---|---|---|
| yamllint | YAML 문법과 서식 | Ansible 의미 |
| ansible-lint | 모듈 이름 존재, 알려진 위험 작성법 없음 | 내 서버에서 의도대로 동작하는지 |
| --syntax-check | Ansible 이 구조를 해석할 수 있음 | **변수가 맞는지.** `{{ }}` 안 오타는 통과함 |

- 초록은 "배포해도 된다"가 아니라 "배포해볼 만하다"
- 실행 검증은 `--check --diff`, 실제 실행 후 2회차 changed=0, /health 로 따로 한다

## CD 를 넣지 않은 이유
1. **닿지 않는다**: 러너는 인터넷, lab01 은 맥 안의 192.168.64.0/24 사설망
2. 개인키를 GitHub Secrets 에 맡겨야 한다. 편의를 위해 신뢰 범위를 넓히는 일
3. 인프라 변경이 push 만으로 시작되면 실수가 바로 운영에 닿는다. 능력이 아니라 설계 선택
- 실무 대안: 셀프호스티드 러너 / CI·CD 분리 / `ansible-pull` 같은 풀 방식
- 면접에서는 "못 했다"가 아니라 "안 했다, 이유는 이렇다"로 말한다

## 배지와 브랜치 보호
- 배지: `![CI](https://github.com/<계정>/<저장소>/actions/workflows/ci.yml/badge.svg)`. 깨진 상태면 깨진 대로 보인다
- 브랜치 보호(Rulesets)에서 Require status checks 를 켜고 `플레이북 검증` 선택
- 검사 이름은 **워크플로가 한 번 실행된 뒤에** 검색에 나타난다
- 필수 체크를 켜면 main 에 **직접 push 가 거절**된다 (GH013). 브랜치를 만들어 push 하고 PR 을 열어 체크 통과 후 머지
  → `troubleshooting/2026-11-11-push-rejected-required-status-check.md`
