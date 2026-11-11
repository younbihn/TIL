## main 에 push 가 거절됨 (Required status check "플레이북 검증" is expected)

**상태**: 원인과 우회 경로 확인. PR 생성과 CI 통과 후 머지는 미확인.

**증상**
[맥] $ git push origin main
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote: - Required status check "플레이북 검증" is expected.
 ! [remote rejected] main -> main (push declined due to repository rule violations)

**원인**
브랜치 보호 규칙(Rulesets)에서 main 에 **Require status checks to pass** 를 켜 두었다. 직접 push 한 커밋에는 CI 가 돈 기록이 없어서 필수 체크 `플레이북 검증` 이 비어 있다고 판단해 거절한다. 규칙이 제대로 동작하는 것이다.
- 필수 체크 이름은 워크플로가 한 번 실행된 뒤에야 검색에 나타난다 (규칙을 만들기 전에 push 해서 한 번 돌려야 함)
- 이름은 워크플로 이름이 아니라 **job 의 `name:`** 값

**해결**
이미 만든 커밋을 브랜치로 옮겨 PR 로 올린다.
```bash
git switch -c docs/readme              # 현재 커밋을 가진 새 브랜치
git branch -f main origin/main         # 로컬 main 을 원격 main 위치로 되돌림 (작업 폴더는 그대로)
git push -u origin docs/readme         # 브랜치는 규칙 대상이 아니라 통과
```
GitHub 웹에서 PR 생성 → `플레이북 검증` 이 초록이 되면 Merge 버튼 활성화 → 머지 후 `git switch main && git pull`.
- `gh` 가 없으면 push 직후 출력되는 `pull/new/<브랜치>` 링크로 웹에서 만든다

**참조**
- GitHub Docs: Rulesets, Available rules for rulesets (Require status checks)
- cicd/github-actions.md

**교훈**
- 필수 체크를 켜면 main 변경은 전부 브랜치 → PR → CI 통과 → 머지. 포트폴리오에는 이 기록(PR 과 초록 체크)이 한 번쯤 남는 게 좋다
- `git reset --hard origin/main` 은 커밋하지 않은 수정까지 지우므로 쓰지 않는다. `git branch -f` 는 작업 폴더를 건드리지 않는다
- 로컬 브랜치를 PR 전에 삭제해도 원격 브랜치에 커밋이 있으면 PR 은 웹에서 만들 수 있다
