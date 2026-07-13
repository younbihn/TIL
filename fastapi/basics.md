# FastAPI 기초 — 첫 서버 가동

- 역할 분담: uvicorn = 홀(네트워크 요청 접수·응답 서빙, ASGI 서버), FastAPI = 주방(요청 처리 로직)
- `uvicorn main:app --reload` = "main.py 안의 app 객체를 서빙해줘, 저장하면 자동 재시작(개발 모드)"
- `@app.get("/")` 데코레이터 = "GET 방식 + / 경로" 요청이 오면 이 함수를 실행하라고 등록하는 것
- 함수가 리턴한 파이썬 dict는 FastAPI가 JSON으로 자동 변환해서 응답
- `/docs` = 내가 만든 게 아니라 FastAPI가 내 코드를 읽고 자동 생성한 API 문서. 코드가 곧 문서