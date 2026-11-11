## nginx 가 404, 앱은 정상 기동인데 /signup 라우트만 없다 (데코레이터의 @ 누락)

**상태**: 해결. 수정 후 `POST /api/signup` 이 201 을 반환했다 (2026-11-11 23:52 KST).

**증상**
```
curl -i -X POST http://192.168.64.2/api/signup ...
HTTP/1.1 404 Not Found
Server: nginx
Content-Type: application/json
{"detail":"Not Found"}
```
- 본문이 JSON 형식의 `{"detail":"Not Found"}` 라서 nginx 가 아니라 **앱(FastAPI)이 낸 404** 였다. 즉 웹 → 앱 경로는 정상이고 앱이 경로를 몰랐다
- 앱의 경로 목록: `ssh lab01 "curl -s http://192.168.64.4:8000/openapi.json" | python3 -m json.tool | grep '"/'` → `/`, `/hello`, `/items`, `/login`, `/me` (`/signup` 없음)
- 배포는 정상: `deploy-app.yml` 의 `소스 받아오기` ok, 커밋 `b6ad16a`(MySQL 전환)가 app01 에 있고 `main.py` 82번 줄에 `signup` 이 있었다

**nginx 접근 로그로 본 시간순 (lab01, `/var/log/nginx/access.log`)**
```
192.168.64.1 - - [11/Nov/2026:23:40:25 +0900] "POST /api/signup HTTP/1.1" 404 22 "-" "curl/8.7.1" "-"
192.168.64.1 - - [11/Nov/2026:23:46:16 +0900] "POST /api/signup HTTP/1.1" 404 22 "-" "curl/8.7.1" "-"
192.168.64.1 - - [11/Nov/2026:23:52:00 +0900] "POST /api/signup HTTP/1.1" 201 54 "-" "curl/8.7.1" "-"
```
- 같은 요청이 404, 404, 201 로 바뀐 것이 수정 전후의 증거다. 응답 크기도 22바이트(`{"detail":"Not Found"}`)에서 54바이트(가입 완료 JSON)로 바뀌었다
- 맥(192.168.64.1)에서 온 요청이 nginx 까지는 매번 도달했다. 즉 404 는 nginx 앞단(네트워크, 방화벽)이 아니라 뒤의 앱에서 났다

**잘못된 가설 두 개 (시간을 쓴 곳)**
1. 앱 저장소에 push 가 안 돼 옛 코드가 배포됐다 → app01 의 `git log` 가 최신 커밋을 보여서 기각
2. 코드는 맞는데 프로세스가 옛 코드를 들고 있다 → `systemctl restart` 후에도 `/signup` 이 없어서 기각. 재시작은 원인과 무관했다
- 이 두 가설은 `grep -n signup main.py` 의 82번 줄을 "코드가 있다"고 읽은 데서 나왔다. 줄 **앞머리**까지 읽지 않았다

**원인**
```
82:app.post("/signup", status_code=201)       ← @ 가 없다
83:def signup(req: SignupRequest, request: Request):
```
`@app.post(...)` 는 아래 함수를 라우트로 등록하는 **데코레이터**다. `@` 가 빠지면 `app.post(...)` 는 그냥 함수 호출이 되고, 반환값(등록용 함수)을 아무 데도 쓰지 않고 버린다. 문법상 올바른 Python 이라 다음이 모두 통과한다.
- `python3 -m py_compile main.py`: 통과
- 앱 기동과 systemd 상태: 정상 (`active (running)`)
- Ansible 배포와 `uri` 확인(`/` 200): 통과
- CI 의 `--syntax-check`: 앱 코드는 검사 대상이 아님
`signup` 함수는 만들어지기만 하고 라우터에 등록되지 않아, `/openapi.json` 에도 없고 요청에는 404 가 온다.

**진단 순서가 갈린 지점**
```bash
# 앱이 가진 경로를 그대로 본다 (nginx 와 배포를 건너뛰고 앱 자체를 확인)
ssh lab01 "curl -s http://192.168.64.4:<포트>/openapi.json" | python3 -m json.tool | grep '"/'
# 라우트 선언 줄을 한 번에 비교한다 (@ 유무)
grep -n "app\.\(get\|post\|put\|delete\)" main.py
```

**해결**
```bash
sed -i '' 's|^app\.post("/signup"|@app.post("/signup"|' main.py    # macOS sed 는 -i '' 필요
git commit -am "fix: signup 라우트 데코레이터의 @ 누락 수정" && git push origin main
ansible-playbook -i inventory/hosts.yml playbooks/deploy-app.yml -K   # 소스 changed → 재시작 핸들러
```
수정 후 `POST /api/signup` → `HTTP/1.1 201 Created`, `{"email":"test@example.com","message":"가입 완료"}`.

**참조**
- FastAPI 라우트 데코레이터(`@app.post`), `/openapi.json`
- troubleshooting/2026-11-11-target-python-libs-missing.md (같은 날 배포 문제)

**교훈**
- **배포 후 검증에 "앱이 가진 라우트 목록"을 넣는다.** `/` 200 과 서비스 active 만으로는 라우트 하나가 빠진 것을 알 수 없다. `/openapi.json` 에 기대한 경로가 있는지를 확인한다
- 증상이 404 일 때는 **누가 404 를 냈는지**부터 본다. JSON `{"detail":...}` 는 앱, HTML 은 nginx
- `py_compile` 은 문법만 본다. 호출 결과를 버리는 줄(`app.post(...)`)은 문법 오류가 아니다. 린트(flake8 의 `B018`, 또는 테스트 한 줄)로 잡을 수 있다
- 출력을 읽을 때 줄 **내용**만이 아니라 **앞머리 문자**까지 읽는다. 눈으로 "signup 이 있다"에서 멈추면 원인이 코드가 아닌 곳에 있다고 오판한다
