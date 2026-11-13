# healthcheck.py IndentationError: unexpected indent (2026-11-13)

## 증상
```
File "scripts/healthcheck.py", line 233
    if args.out_dir:
IndentationError: unexpected indent
```
`python3 scripts/healthcheck.py` 와 `python3 -m py_compile scripts/healthcheck.py` 모두 같은 오류. 시험용 코드의 `import healthcheck` 도 같은 곳에서 실패했다.

## 원인
`if args.out_dir:` 한 줄만 앞 공백이 8칸이었다. 같은 함수(`main`) 안의 윗줄(`print(text)`)은 4칸이어서 파이썬이 "아무 블록도 열지 않았는데 들여쓰기가 깊어졌다"고 판단했다. 탭은 아니었고 공백 개수 문제였다(`cat -et` 로 확인). 코드를 편집기에 붙여 넣을 때 자동 들여쓰기가 공백을 더한 것으로 추정한다(확인하지 않음).

## 해결
233번째 줄만 4칸으로 줄였다.
```
sed -i '' '233s/^ *if args.out_dir:/    if args.out_dir:/' scripts/healthcheck.py
python3 -m py_compile scripts/healthcheck.py && echo "문법 OK"
```
이후 `python3 scripts/healthcheck.py` 가 정상 실행됐다(5대 OK, 종료 코드 0).

## 참조
- 눈에 안 보이는 공백 문제는 `sed -n '시작,끝p' 파일 | cat -et` 로 줄 끝(`$`)과 탭(`^I`)을 보면 찾는다. macOS 의 `cat` 은 `-e`, `-t` 를 지원한다.
- 붙여 넣기 전에 vim 에서 `:set paste` 를 켜면 자동 들여쓰기를 막는다.
- 실행 전에 `python3 -m py_compile` 로 문법만 먼저 검사하는 습관이 오류 위치를 빨리 알려 준다.
- 같은 종류의 문제: `troubleshooting/2026-11-11-yaml-tab-indentation.md`
