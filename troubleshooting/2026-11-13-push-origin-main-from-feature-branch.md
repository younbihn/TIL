# git push origin main 을 했는데 원격 저장소에 파일이 없다 (2026-11-13)

## 증상
스터디 문서의 커밋 명령(`git add` → `git commit` → `git push origin main`)을 실행한 뒤 GitHub 저장소에서 `scripts/healthcheck.py` 가 보이지 않았다. 원격 저장소를 확인하니 브랜치는 `main` 하나뿐이었고 `main` 은 PR #3 병합 시점(6e07b1c)에 머물러 있었으며 `feat/lb01` 브랜치도 없었다.

## 원인
- 작업 브랜치가 `feat/lb01` 이어서 커밋이 `main` 이 아니라 `feat/lb01` 에 쌓였다. `git log --oneline -3` 출력: `4118a84 (HEAD -> feat/lb01) feat(scripts): 전 서버 상태 점검 healthcheck.py ...`, `8ac5ce1 feat(lb): HAProxy 이중화 ...`, `6e07b1c (origin/main, main) Merge pull request #3 ...`
- `git push origin main` 은 "내 컴퓨터의 `main` 브랜치"를 원격 `main` 으로 보내는 명령인데, 맥의 `main` 에는 새 커밋이 없어 보낼 것이 없었다.
- 설령 `main` 에서 커밋했어도 `protect-main` 규칙(필수 상태 검사 `플레이북 검증`, 우회 대상 없음) 때문에 직접 push 는 거절된다. 스터디 문서의 `git push origin main` 은 이 규칙을 만들기 전 흐름이다.

## 해결
브랜치를 원격에 올리고 PR 로 병합한다. 이 항목은 방법만 정리했고 실행 결과는 미확인이다.
```
git status --short                 # 커밋하지 않은 파일이 남았는지 확인
ansible-lint && yamllint .         # CI 와 같은 검사를 먼저
git push -u origin feat/lb01       # 브랜치를 원격에 올림 (-u 는 원격 브랜치와 짝을 기억)
# GitHub 에서 PR 생성 → CI 통과 → 병합
git switch main && git pull        # 병합 뒤 로컬 main 갱신
```

## 참조
- 확인 명령: `git branch --show-current`, `git log --oneline -3` (HEAD 가 가리키는 브랜치와 `origin/main` 위치), `git branch -vv`, `git ls-remote --heads origin` (원격에 있는 브랜치)
- 같은 규칙에서 막힌 사례: `troubleshooting/2026-11-11-push-rejected-required-status-check.md`
- `git push origin <브랜치>` 는 현재 브랜치와 상관없이 지정한 로컬 브랜치를 보낸다. 현재 브랜치를 보내려면 `git push -u origin HEAD` 처럼 쓴다.

## 재발 (2026-11-14 14:31)
- 상황: `feat/lb01` 브랜치에서 L4 실험 변경을 커밋(`a1092f5`, 3개 파일, 23줄 추가)한 뒤 학습 자료의 마지막 줄 `git push origin main` 을 다시 실행했다.
- 결과: `! [rejected] main -> main (fetch first)`. 이번에는 "보낼 것이 없음"이 아니라 거절이었다. 원격 `main` 에 로컬 `main` 에 없는 커밋이 있어 로컬 `main` 이 뒤처진 상태라는 뜻이다(어떤 커밋인지는 확인하지 않음, 미확인). 어느 쪽이든 작업 커밋이 있는 곳은 `feat/lb01` 이다.
- 대응: `git push -u origin feat/lb01` 로 브랜치를 올리고 PR 로 병합한다(위 해결 절차). 원격 `main` 의 새 커밋은 `git fetch origin` 후 `git log --oneline HEAD..origin/main` 으로 확인하고, 있으면 `feat/lb01` 을 그 위로 옮길지(`git rebase origin/main`) PR 화면의 충돌 여부를 보고 정한다.
- 교훈: 학습 자료의 `git push origin main` 줄은 보호 규칙을 만들기 전 흐름이다. 자료의 커밋 명령을 그대로 붙이지 않고 push 줄만 `git push -u origin 현재브랜치` 로 바꿔 쓴다.

## 재발 세 번째 (2026-11-14 15:19)
- 상황: `docs/readme-lb-l4` 브랜치에서 `selinux-web-content.yml` 을 커밋(`9cd6662`, 1개 파일, 41줄 추가)한 뒤 학습 자료의 마지막 줄 `git push origin main` 을 또 실행했다.
- 결과: `Everything up-to-date`. 오류가 없어서 푸시가 된 것처럼 보이지만 로컬 `main` 에는 새 커밋이 없어 아무것도 올라가지 않았고, 커밋 `9cd6662` 는 로컬 브랜치에만 있다. 11/13(보낼 것 없음 계열), 11/14 14:31(거절), 이번(보낼 것 없음)으로 세 번째다.
- 대응: `git push -u origin HEAD`(현재 브랜치를 그 이름으로 올림). 올린 뒤 `git ls-remote --heads origin docs/readme-lb-l4` 로 원격에 생겼는지 확인한다. 이 방법은 git 설정을 바꾸지 않는다.
- 교훈: "Everything up-to-date" 는 성공이 아니라 "내가 지정한 브랜치에는 보낼 것이 없다"는 뜻이다. 커밋 직후 `git status -sb` 에서 `ahead N` 이 사라졌는지 본다. 이번에도 같은 줄(자료의 push 줄)에서 발생했으므로 자료의 커밋과 push 블록은 한꺼번에 붙여 넣지 않는다.
