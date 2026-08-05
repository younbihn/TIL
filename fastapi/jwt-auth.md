# JWT 인증

- JWT = header.payload.signature. 페이로드는 Base64일 뿐 암호화 아님 → 민감정보 금지
- 서명(SECRET_KEY 기반)으로 위조를 탐지. "숨기는 기술"이 아니라 "위조를 막는 기술"
- HTTP는 무상태 → 매 요청 신원 증명 필요 → 토큰 방식이면 서버가 세션 저장 불필요(확장 유리)
- 표준 클레임: sub(주인), exp(만료), iat(발급시각). exp는 라이브러리가 자동 검사
- 로그인 실패 문구는 통일 → "이메일 없음/비밀번호 틀림" 구분 시 계정 열거 공격 가능
- Depends = 의존성 주입. 인증 로직을 한 곳에 모으고 엔드포인트마다 한 줄만 붙임
- try/except는 구체적 예외를 먼저 (ExpiredSignatureError → InvalidTokenError)
- 토큰은 발급 시점 스냅샷 → 탈퇴·차단 반영 위해 DB 재조회 필요
- SECRET_KEY 유출 = 토큰 위조 가능 → 환경변수로 분리