# 엔드포인트 — 쿼리 파라미터와 POST

- 쿼리 파라미터 = 주소 뒤 ?name=값. 함수 매개변수로 선언하면 FastAPI가 자동 연결, 기본값 지정 가능
- GET = 조회(정보가 주소에 노출) / POST = 제출·생성(정보가 body에 담김)
- Pydantic BaseModel = 데이터 입구 검문소. 타입 안 맞으면 422 자동 응답
- POST 테스트는 브라우저 주소창이 아니라 /docs의 Try it out으로