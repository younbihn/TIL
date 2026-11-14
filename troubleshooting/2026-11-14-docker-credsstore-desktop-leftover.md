# docker pull 이 docker-credential-desktop 때문에 실패 (Docker Desktop 없이 colima 사용)

## 환경
- 💻 맥 (M1), colima (docker 런타임), Homebrew docker CLI. Docker Desktop 은 설치하지 않음
- 2026-11-14 15:4x, `monitoring/` 에서 `docker compose up -d`

## 증상
```
error getting credentials - err: exec: "docker-credential-desktop": executable file not found in $PATH, out: ``
```
`docker run` 과 `docker compose up` 모두 이미지를 받기 전에 같은 오류로 멈춤.

## 원인
`~/.docker/config.json` 에 Docker Desktop 시절의 `"credsStore": "desktop"` 이 남아 있었다.
도커가 이미지를 받을 때 이 값에 해당하는 `docker-credential-desktop` 프로그램을 찾는데 없어서 실패.
`which docker-credential-osxkeychain docker-credential-desktop` 도 둘 다 없음.
(같은 뿌리로 보이는 증상: `~/.docker/cli-plugins/*` 가 `/Volumes/Docker/...` 를 가리키며 깨져 있었음. 인과는 미확인)

## 해결
백업 후 `credsStore` 키만 제거. 공개 이미지(prom/prometheus, prom/alertmanager, grafana/grafana)는 로그인 정보가 필요 없다.
```bash
cp ~/.docker/config.json ~/.docker/config.json.bak.$(date +%Y%m%d-%H%M)
python3 - <<'PY'
import json, pathlib
p = pathlib.Path.home() / ".docker" / "config.json"
c = json.loads(p.read_text())
c.pop("credsStore", None)
p.write_text(json.dumps(c, indent=2) + "\n")
PY
```
이후 `docker pull prom/alertmanager:v0.28.1` 성공, compose 이미지 3개 pull 성공.

## 주의
- 비공개 레지스트리를 쓰게 되면 그때 credential helper(osxkeychain 등)를 별도로 설정해야 한다.
- 함께 겪은 실수: 저장소 루트에서 `docker run -v "$PWD/alertmanager":/a` 를 실행하면 없는 경로를 도커가 빈 디렉터리로 만들 수 있다. 실행 위치(`pwd`)를 먼저 확인한다.
