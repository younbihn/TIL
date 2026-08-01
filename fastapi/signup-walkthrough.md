# FastAPI 회원가입 코드 전체 해부

> first-come-api 프로젝트의 main.py를 한 줄씩 분석한 문서.
> 3주차(2026.07) 학습 정리 · 면접 시 프로젝트 설명 대본으로 활용.

---

## 0. 큰 그림

이 파일은 **식당 하나**다.
- `app` = 식당 건물
- `@app.get`, `@app.post`가 붙은 함수 = 각각의 주문 창구
- 손님(브라우저)이 특정 창구에 주문 → 연결된 함수 실행 → 음식(응답) 반환

현재 창구 4개: `/`, `/hello`, `/items`, `/signup`

---

## 1. import — 부품 가져오기

```python
from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel, EmailStr, Field
import bcrypt
import logging
```

### 두 가지 import 방식
| 방식 | 의미 | 사용법 |
|---|---|---|
| `import bcrypt` | 상자째 가져오기 | `bcrypt.hashpw(...)` |
| `from fastapi import FastAPI` | 특정 물건만 꺼내기 | `FastAPI` (앞에 모듈명 불필요) |

### 각 부품의 정체
| 부품 | 출신 | 역할 |
|---|---|---|
| `FastAPI` | fastapi | 식당 건물을 만드는 설계도 |
| `HTTPException` | fastapi | "손님, 이건 안 됩니다" 에러를 던지는 도구 |
| `Request` | fastapi | 손님이 들고 온 주문서 전체(IP, 헤더 등) |
| `BaseModel` | pydantic | 주문서 양식의 기본 틀 |
| `EmailStr` | pydantic | "이메일 형식이어야 함" 검사 도장 |
| `Field` | pydantic | "8자 이상" 같은 세부 조건 부착 도구 |
| `bcrypt` | bcrypt | 비밀번호를 되돌릴 수 없게 만드는 분쇄기 |
| `logging` | 파이썬 기본 | 무슨 일이 있었는지 기록하는 일지 |

- `logging`은 파이썬 내장 → 설치 불필요
- `fastapi`, `pydantic`, `bcrypt`는 pip으로 venv에 설치한 외부 부품 → requirements.txt에 기록됨

---

## 2. 로거 설정

```python
logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)
```

- `=` 는 "왼쪽 이름표를 오른쪽 물건에 붙인다"
- `__name__` = 파이썬이 자동으로 채우는 **현재 파일 이름**. main.py이므로 `"main"`
  → 로그 출력 `INFO:main:signup completed...` 의 가운데 `main`이 이것
  → 파일이 여러 개로 늘어나면 "어느 파일에서 나온 로그인지" 구분 가능
- 앞뒤 밑줄 두 개(`__name__`)는 파이썬이 특별 취급하는 이름이라는 표시 (`__init__`도 같은 가족)

### 로그 레벨
```
DEBUG(개발용 잡담) < INFO(정상 사건) < WARNING(경고) < ERROR(오류) < CRITICAL(치명)
```
- `level=INFO` = INFO 이상만 출력, DEBUG는 무시
- 실무: 개발 중엔 DEBUG로 시끄럽게 → 운영에선 INFO/WARNING으로 잡음 제거
- `level=` 처럼 이름 붙여 넘기는 방식 = **키워드 인자** (순서 외울 필요 없음)

---

## 3. 앱 생성

```python
app = FastAPI()
```

- `FastAPI`는 설계도, 뒤에 `()`를 붙이면 "이 설계도로 실물 하나 만들어라"
- 붕어빵 틀(`FastAPI`) → 붕어빵 하나(`app`). 이렇게 만들어진 실물 = **객체(인스턴스)**
- `uvicorn main:app` 의 `app`이 바로 이 변수명
  → **main = 파일명, app = 변수명**. 둘 중 하나 바뀌면 명령어도 바꿔야 함

---

## 4. 창구 1 — 생존 확인

```python
@app.get("/")
def root():
    return {"status": "alive"}
```

### 데코레이터 (`@`)
- 바로 아래 함수에 **꼬리표를 붙이는 장치**
- "너는 GET 방식으로 `/` 주소에 요청이 오면 실행되는 담당자야"라고 등록
- 식당 비유: 함수 = 요리사, 데코레이터 = **"1번 창구: 물 주문 받음"** 간판
  → 간판이 없으면 요리사가 있어도 손님이 주문 넣을 방법이 없음

### def와 들여쓰기
- `def` = 함수 정의(define)
- 콜론(`:`) 다음 줄부터 **들여쓰기된 부분이 함수의 몸통**
- 파이썬은 중괄호 대신 들여쓰기로 범위 구분 → 들여쓰기가 문법 그 자체 (스페이스 4칸 표준)

### return과 dict
- `{"키": "값"}` = **dict(딕셔너리)** = 폰켓몬 문제에서 만난 **해시 테이블**
  → 이름표(키)로 물건(값)을 즉시 꺼냄, 조회 평균 **O(1)**
- 파이썬 dict를 리턴했는데 브라우저는 JSON을 받음
  → JSON = 서로 다른 언어끼리 데이터를 주고받는 세계 공통 문자 형식
  → 이 변환은 **FastAPI가 자동 처리**

---

## 5. 창구 2 — 쿼리 파라미터

```python
@app.get("/hello")
def hello(name: str = "익명"):
    return {"message": f"안녕하세요, {name}님"}
```

### `name: str = "익명"` 해부
| 조각 | 의미 |
|---|---|
| `name` | 받아올 값의 이름 |
| `: str` | **타입 힌트** — 문자열이어야 함 |
| `= "익명"` | **기본값** — 안 주면 "익명" |

- 파이썬은 타입을 안 적어도 되지만, **FastAPI는 이 힌트를 읽어서 자동 검증 + 문서 생성**
  → 적는 순간 일을 대신 해주므로 FastAPI에서는 필수 습관

### FastAPI 규칙
함수 매개변수에 **단순 타입**을 선언하면 → 주소 뒤 `?name=값`(쿼리 파라미터)에서 자동 추출
- `/hello?name=철수` → `name`에 `"철수"` 주입
- **주소를 파싱하는 코드를 한 줄도 안 씀**

### f-string
- 따옴표 앞 `f` = 중괄호 안 변수가 실제 값으로 치환
- `f"안녕하세요, {name}님"` → `"안녕하세요, 철수님"`
- ⚠️ 로그에서는 f-string 대신 `%s` 사용 (아래 11번 참고) — 상황에 따라 다름

---

## 6. 창구 3 — POST와 요청 본문

```python
class Item(BaseModel):
    name: str
    price: int
```

### class와 상속
- `class` = 설계도를 **내가** 만드는 키워드 (`FastAPI`는 남이 만든 설계도)
- `(BaseModel)` = **상속** — "Pydantic BaseModel의 능력을 전부 물려받겠다"
  → 그 능력 = **자동 검증**
  → 나는 양식만 적었는데 검사 기능은 부모에게서 공짜로 받음
- CS 개념: **상속(inheritance)** = 객체지향(OOP)의 핵심 기둥
  → 계획서 7월 항목의 OOP를 이론이 아닌 **필요한 자리에서** 만난 것 (탑다운)

```python
@app.post("/items")
def create_item(item: Item):
    return {"created": item.name, "price": item.price}
```

### GET vs POST
| | GET | POST |
|---|---|---|
| 목적 | 조회 | 제출·생성 |
| 데이터 위치 | 주소에 노출 | **요청 본문(body)** |
| 제약 | 길이 제한 | 대용량 가능 |
| 용도 | 검색, 조회 | 비밀번호 등 노출 금지 데이터 |

### 타입 하나로 동작이 바뀐다
- `name: str` (단순 타입) → **쿼리 파라미터**에서 추출
- `item: Item` (Pydantic 모델) → **요청 본문 JSON**에서 추출 + 검증
- 타입만 바꿨을 뿐인데 동작이 완전히 달라짐 = FastAPI의 설계 철학

### 점(`.`) 표기
- dict는 `["키"]`로 꺼냄 / 객체는 `.속성`으로 꺼냄
- `item.name` = "그 물건의 속성 꺼내기"

---

## 7. 저장소

```python
users_db: dict[str, dict] = {}   # 인메모리 저장소 (8월에 PostgreSQL로 교체)
```

- `{}` = 빈 dict 생성 (아직 아무도 가입 안 한 회원 명부)
- `: dict[str, dict]` = 타입 힌트. "키는 문자열(이메일), 값은 또 다른 dict"
  → 파이썬이 강제하진 않음, 사람과 편집기를 위한 표지판

### 왜 dict인가
- 핵심 연산이 "이 이메일 이미 있나?" → dict는 해시 테이블이라 **회원 100만 명이어도 평균 O(1)**
- 리스트였다면 처음부터 훑어서 **O(N)**
- 폰켓몬에서 배운 원리가 그대로 실전 투입된 지점

### "인메모리"의 의미
- 이 명부는 **RAM(작업 기억)**에 존재 → 전원 끊기면 사라지는 **휘발성**
- 서버 재시작이나 `--reload` 시 회원 전부 증발
- → 8월에 **PostgreSQL(디스크 영구 저장)**이 필요한 이유
- 지금은 "DB 없이도 일단 동작"을 확인하는 단계

---

## 8. 비밀번호 함수

```python
def hash_password(plain: str) -> str:
    return bcrypt.hashpw(plain.encode(), bcrypt.gensalt()).decode()
```

- `-> str` = **반환값 타입 힌트**. "이 함수는 문자열을 돌려준다"

### 안쪽부터 읽기
1. `plain.encode()` — 문자열 → **바이트(bytes)**
   → 컴퓨터는 글자를 모르고 숫자만 앎. bcrypt는 암호 계산기라 바이트만 받음
2. `bcrypt.gensalt()` — **소금(salt)** 생성. 매번 다른 무작위 값
3. `bcrypt.hashpw(비밀번호, 소금)` — 둘을 섞어 분쇄 (결과는 바이트)
4. `.decode()` — 다시 문자열로 (저장·전송 편의)

### CS 관점 3가지
1. **단방향** — 해싱은 복원 불가 → 그래서 "비밀번호 찾기"가 없고 "재설정"만 존재
2. **salt** — 같은 비밀번호도 결과가 매번 다름 → 미리 계산된 해시 사전(레인보우 테이블) 무력화
3. **의도적 느림** — bcrypt는 일부러 느림(cost factor)
   → 보통 프로그램은 빠를수록 좋지만, 여기선 **느림이 방어막**
   → 공격자의 초당 수억 번 시도를 차단

```python
def verify_password(plain: str, hashed: str) -> bool:
    return bcrypt.checkpw(plain.encode(), hashed.encode())
```

- `-> bool` = True/False만 반환
- 저장된 해시는 못 되돌리는데 어떻게 검증하나?
  → **입력받은 비밀번호를 저장된 해시 안의 salt로 똑같이 분쇄해서 결과 비교**
  → bcrypt가 salt를 결과 문자열 안에 담아두기 때문에 가능
- `/login`에서 사용 예정

---

## 9. 마스킹 함수

### 이메일 마스킹
```python
def mask_email(email: str) -> str:
    local, _, domain = email.partition("@")
```

- `partition("@")` — @ 기준 **세 조각**으로 분리
  `"abc@test.com"` → `("abc", "@", "test.com")`
- 한 줄에 변수 3개 = **언패킹** (세 조각을 왼쪽 세 이름에 순서대로 배분)
- `_`(밑줄) = "이 값은 안 쓴다"는 **관례적 표시**. 문법적으론 그냥 변수명이지만 개발자 사이의 신호
- `split("@")` 대신 `partition`을 쓴 이유: @가 없는 입력이 와도 에러 없이 세 조각 반환 (**방어적 코딩**)

```python
    if len(local) <= 2:
        masked = local[0] + "*"
    else:
        masked = local[:2] + "*" * (len(local) - 2)
    return f"{masked}@{domain}"
```

| 문법 | 의미 |
|---|---|
| `if / else` | 조건 분기. 들여쓰기가 소속 결정 |
| `len(local)` | 길이 측정. 크기를 미리 저장하므로 **O(1)** |
| `local[0]` | 첫 글자. 파이썬은 **0부터** (K번째수의 -1 보정과 같은 이유) |
| `local[:2]` | 슬라이싱 — 처음부터 2번 인덱스 직전까지 = 앞 두 글자 |
| `"*" * 5` | 문자열 곱셈 = **반복** → `"*****"` |
| `+` | 문자열 이어붙이기 → `"yo" + "******"` = `"yo******"` |

### IP 마스킹
```python
def mask_ip(ip: str) -> str:
    parts = ip.split(".")
    if len(parts) == 4:
        return ".".join(parts[:3] + ["*"])
    return ip.rsplit(":", 1)[0] + ":*"
```

| 문법 | 의미 |
|---|---|
| `split(".")` | 점 기준으로 잘라 **리스트** 생성 → `["192","168","0","5"]` |
| `len(parts) == 4` | 조각 4개면 IPv4로 판별 |
| `==` vs `=` | `==`는 "같다"를 **묻는** 비교 / `=`는 **대입**. 초보 실수 1순위 |
| `parts[:3] + ["*"]` | 리스트끼리 `+`는 이어붙이기 → `["192","168","0","*"]` |
| `".".join(리스트)` | 요소들을 점으로 연결해 문자열로 → `"192.168.0.*"` (**split의 반대**) |
| `rsplit(":", 1)[0]` | **오른쪽에서부터 1번만** 잘라 앞부분 취함 (IPv6 마지막 블록 가림) |

**왜 마지막 옥텟을 가리나**: 그 부분이 개별 기기를 특정하는 데 가장 결정적.
앞 3옥텟은 대략적 네트워크(지역·통신사)까지만 알려줌 → 프라이버시는 지키되 통계는 남기는 절충.

---

## 10. 가입 요청 양식

```python
class SignupRequest(BaseModel):
    email: EmailStr                      # 이메일 형식 자동 검증
    password: str = Field(min_length=8)  # 8자 미만이면 422
```

- `EmailStr` — 단순 `str`이 아니라 "@가 있고 도메인 형태가 맞는가"까지 검사하는 특별 타입
  → Pydantic 본체에 없어서 `pip install "pydantic[email]"`로 별도 설치
  → 대괄호 안의 것을 **extra(선택 부품)**라고 부름
- `Field(min_length=8)` — 필드에 세부 조건 부착

### 핵심: 선언적 프로그래밍
- 이 조건들을 검사하는 `if`문을 **하나도 안 씀**
- 양식만 선언하면 FastAPI가 위반 시 **422(검증 실패)** 자동 응답
- **"어떻게 검사할지"가 아니라 "무엇이 옳은지"만 적는 방식**

---

## 11. 창구 4 — 회원가입

```python
@app.post("/signup", status_code=201)
def signup(req: SignupRequest, request: Request):
```

- `status_code=201` — 성공 시 기본값 200 대신 201
  → **201 Created = "새로운 자원이 만들어졌다"**
  → 조회 성공(200)과 생성 성공(201)을 구분하는 게 REST API의 예의

### 매개변수 두 개의 성격 차이
| 매개변수 | 내용 |
|---|---|
| `req: SignupRequest` | 손님이 적어낸 **주문 내용** (이메일·비밀번호) |
| `request: Request` | 요청에 딸린 **부가 정보 전체** (IP, 헤더, 쿠키) — FastAPI가 타입 보고 자동 주입 |

⚠️ 이름이 `req`/`request`로 비슷해 헷갈림 → `payload`나 `body`로 변경 권장

```python
    if req.email in users_db:
        raise HTTPException(status_code=409, detail="이미 가입된 이메일입니다")
```

- `in` 연산자 — "이 dict에 이 키가 있나?" → 해시 테이블이라 **평균 O(1)**
  → 회원이 아무리 많아져도 가입 처리가 느려지지 않는 이유
- `raise` — 에러를 **던지는** 명령
  → `return`과의 결정적 차이: 던지는 순간 **함수 실행 즉시 중단**, 아래 코드 실행 안 됨
  → 중복 이메일이 저장되는 사고를 구조적으로 차단
- **409 Conflict** — "요청 자체는 올바른데 현재 상태와 충돌한다"
  → 404(없음), 422(형식 오류)와 구분해서 쓰는 게 좋은 API의 조건

```python
    users_db[req.email] = {
        "email": req.email,
        "hashed_password": hash_password(req.password),
    }
```

- dict에 새 항목 추가. **키는 이메일, 값은 또 다른 dict**
- 값 안에 **평문 비밀번호가 없다**는 게 핵심 → "DB가 털려도 비밀번호는 지켜진다"

```python
    client_ip = request.client.host if request.client else "unknown"
```

### 조건부 표현식(삼항 연산자)
읽는 순서: **가운데 → 왼쪽/오른쪽**
> `request.client`가 있으면 → `request.client.host`를, 없으면 → `"unknown"`을 담아라

- **왜 확인하나**: 테스트 환경 등에서 클라이언트 정보가 없을(`None`) 수 있음
  → 그때 `.host`를 꺼내면 프로그램이 죽음(`AttributeError`)
- 이것이 **방어적 프로그래밍** — 서버는 한 요청의 실수로 전체가 멈추면 안 됨

```python
    logger.info(
        "signup completed: email=%s ip=%s",
        mask_email(req.email),
        mask_ip(client_ip),
    )
```

- `%s` = **자리 표시자**. 뒤에 넘긴 값들이 순서대로 삽입
- **f-string 대신 `%s`를 쓰는 이유**: 로그 레벨 때문에 출력되지 않을 경우
  **문자열 조립 자체를 건너뛰어** 낭비 제거
- **마스킹을 여기서 하는 이유**: 원본은 dict에 저장하되, **밖으로 나가는 기록에는 가린 값만**
  → 로그는 파일 → 수집 서비스 → 백업 → 다수 열람으로 퍼짐
  → 가장 잘 지킨 데이터가 가장 허술한 곳으로 복사되는 것을 방지

```python
    return {"email": req.email, "message": "가입 완료"}
```

- 응답에 **비밀번호도 해시도 없음** → 클라이언트에 필요한 최소 정보만 제공

---

## 12. 전체 흐름 한 번에

`POST /signup` 에 `{"email":"younbihn@test.com","password":"mypassword"}` 전송 시:

1. **uvicorn**이 8000번 포트에서 요청 수신 → FastAPI에 전달
2. FastAPI가 "POST + /signup" 담당자로 `signup` 함수 지목
3. **Pydantic 검문** — 이메일 형식·비밀번호 8자 검사 → 실패 시 여기서 종료(**422**)
4. 통과 → 함수 진입 → 중복 확인 → 있으면 **409**로 중단
5. `hash_password`로 비밀번호 분쇄(salt 포함) → `users_db`에 저장
6. IP 추출 → 이메일·IP 마스킹해서 로그 기록
7. dict 리턴 → FastAPI가 JSON 변환 → **201**과 함께 응답

> 내가 짠 건 실질적으로 **4~7단계뿐**, 나머지는 프레임워크가 처리.
> **프레임워크를 쓴다는 게 이런 의미.**

---

## 13. 등장 개념 총정리

### 파이썬 문법
import 두 방식 · 변수 대입 · f-string · `%s` 로그 포맷 · `def`와 들여쓰기 ·
타입 힌트(`: str`, `-> bool`) · 기본값 매개변수 · 키워드 인자 · dict와 리스트 ·
인덱싱/슬라이싱 · 문자열 반복(`*`)과 연결(`+`) · `split`/`join`/`partition` ·
언패킹과 `_` · `if/else` · 조건부 표현식 · `in` 연산자 · `==` vs `=` ·
`class`와 상속 · `raise` · 데코레이터

### CS 기초
해시 테이블과 O(1) · 휘발성 메모리 vs 영구 저장 · 단방향 해싱과 salt ·
의도적 느림(cost factor) · HTTP 메서드(GET/POST) · 상태코드(201/409/422) ·
JSON과 직렬화 · 문자열 vs 바이트 · 개인정보 마스킹과 로그 확산성 ·
방어적 프로그래밍 · 선언적 프로그래밍 · 객체지향의 상속

---

## 14. TODO

- [ ] 중복된 import 줄 정리 (`from fastapi import ...` 두 번 → 한 번)
- [ ] `req` / `request` 이름 혼동 개선 (`payload` 또는 `body`로 변경 검토)
- [ ] `/login` 구현 시 `verify_password` 활용
- [ ] 8월: `users_db` → PostgreSQL 교체