## YAML parsing failed: Tabs are usually invalid in YAML

**상태**: 해결 (공백으로 교체 후 문법 검사 통과)

**증상**
```
[ERROR]: YAML parsing failed: Tabs are usually invalid in YAML.
Origin: playbooks/deploy-web.yml:16:1
 16   - python3-libsemanage
```

**원인**
`vi` 에서 목록 항목을 추가하며 들여쓰기에 **탭 문자**가 들어갔다. YAML 은 들여쓰기에 공백만 허용한다. 터미널에서는 탭과 공백이 비슷하게 보여서 눈으로 구별하기 어렵다.

**진단**
```bash
grep -nP '\t' playbooks/deploy-web.yml        # 탭이 있는 줄 번호 출력 (없으면 출력 없음)
# grep -P 가 없으면
grep -n "$(printf '\t')" playbooks/deploy-web.yml
```

**해결**
- 탭이 든 줄을 공백으로 바꾼다 (`name:` 아래 목록은 부모보다 2칸 더 들여쓰기)
- vi 를 쓸 때는 `:set expandtab` 을 먼저 입력하면 Tab 키가 공백으로 입력된다
- 통째로 붙여넣을 때는 `:set paste` 후 붙여넣고 `:set nopaste` 로 되돌린다 (자동 들여쓰기가 내용을 밀어내는 것을 막는다)
- 전체 삭제는 Normal 모드에서 `gg` 후 `dG`, 또는 `:%d`

**교훈**
- 에디터 설정(`expandtab`)이나 `.editorconfig` 로 탭 입력 자체를 막는 편이 확인보다 싸다
- `yamllint` 는 탭을 잡아 주는 규칙을 이미 갖고 있다. push 전에 `yamllint . && ansible-lint` 를 습관으로 한다
