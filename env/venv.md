# venv (가상환경)

- venv = 프로젝트 전용 파이썬 공구함. 프로젝트마다 패키지 버전이 달라도 충돌 안 나게 방을 분리하는 것
- 생성 `python3 -m venv venv` / 입장 `source venv/bin/activate` / 퇴장 `deactivate`
- activate의 실체 = 현재 터미널의 PATH 맨 앞에 venv/bin을 끼워넣기 → 그때부터 python, pip이 방 안 것으로 실행됨
- activate는 전역 명령이 아니라 venv 폴더 안의 스크립트 → 그래서 `source + 경로`로 실행
- venv 폴더는 git에 절대 안 올림(.gitignore). 소모품이라 지우고 다시 만들어도 됨