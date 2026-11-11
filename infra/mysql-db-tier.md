## DB 계층: MySQL 을 Ansible 로 구성하고 최소 권한 계정 만들기

**한 줄 요약**: `deploy-db.yml` 로 db01 에 MySQL 을 설치하고, app01 주소에서만 접속하는 SELECT, INSERT 전용 계정을 만들었다.

**플레이북 구조 (deploy-db.yml, hosts: dbservers)**
| 순서 | 태스크 | 모듈 | 태그 |
|---|---|---|---|
| 1 | 비밀번호 수신 확인 (12자 이상) | ansible.builtin.assert | always |
| 2 | mysql-server, python3-PyMySQL 설치 | ansible.builtin.dnf | install |
| 3 | bind-address = 0.0.0.0 명시 | ansible.builtin.lineinfile + handler | config |
| 4 | mysqld 시작과 부팅 자동 시작 | ansible.builtin.systemd | service |
| 5 | 데이터베이스 생성 (utf8mb4) | community.mysql.mysql_db | schema |
| 6 | users 테이블 존재 조회 | community.mysql.mysql_query | schema |
| 7 | users 테이블 생성 (없을 때만) | community.mysql.mysql_query + when | schema |
| 8 | 앱 계정 생성 (앱 서버 주소별) | community.mysql.mysql_user + loop | schema |

**핵심 개념**
- **비밀번호를 코드에 두지 않는다**: `lookup('env', 'DB_PASSWORD')` 로 맥의 환경 변수에서 받는다. 비어 있거나 짧으면 `assert` 로 시작 전에 멈추고, 계정 태스크에는 `no_log: true` 를 붙여 로그에 남지 않게 한다. 공개 레포라서 필수였다
- **앱 서버 주소를 인벤토리에서 계산한다**: `groups['appservers'] | map('extract', hostvars, 'ansible_host') | list`. 앱 서버가 늘면 계정이 서버 수만큼 자동으로 생긴다 (`loop`)
- **소켓으로 로컬 관리 접속**: `login_unix_socket: /var/lib/mysql/mysql.sock` 으로 root 소켓 접속을 쓴다. 관리용 비밀번호를 따로 두지 않는다
- **멱등성 만들기**: `CREATE TABLE` 은 두 번 실행하면 오류가 나므로, 먼저 `information_schema.tables` 를 조회(`changed_when: false`)하고 결과가 비었을 때만 생성한다
- **MySQL 계정은 `이름@호스트` 가 한 쌍**: `fcapp@192.168.64.4` 와 `fcapp@%` 는 다른 계정이다. 호스트를 app01 주소로 한정해서 다른 서버에서 같은 이름과 비밀번호로 접속하는 것을 막는다
- **최소 권한**: `priv: firstcome.users:SELECT,INSERT` 처럼 DB.테이블:권한 형식으로 준다. 테이블 단위로 주면 UPDATE, DELETE, DDL, 다른 DB 접근이 모두 막힌다

**검증 (2026-11-11, db01)**
```
SELECT user, host FROM mysql.user WHERE user='fcapp';   → fcapp | 192.168.64.4
SHOW GRANTS FOR 'fcapp'@'192.168.64.4';
  GRANT USAGE ON *.* ...
  GRANT SELECT, INSERT ON `firstcome`.`users` ...
DESCRIBE users;   → email(PK varchar255), hashed_password(varchar100), created_at(timestamp, 기본 현재 시각)
ss -tlnp | grep 3306   → 0.0.0.0:3306 (mysqld), *:33060 (mysqld)
```
- `GRANT USAGE ON *.*` 는 "권한 없음"이라는 뜻의 기본 줄이다. 실제 권한은 아래 줄 하나뿐이다
- 이 결과는 `loop` 가 빈 목록이 아니었다는 증거이기도 하다. 빈 목록이면 계정이 조용히 생략된다

**두 겹의 접근 제어**
1. 방화벽 rich rule: db01 의 3306 은 app01 만 허용 (infra/3tier-access-control.md)
2. MySQL 계정 호스트 제한: 방화벽이 풀려도 app01 주소가 아니면 로그인할 수 없다
- `bind-address = 0.0.0.0` 은 모든 인터페이스에서 받는다는 뜻이라 첫 번째 방어(방화벽)에 의존한다. 그래서 방화벽을 먼저 적용해 두는 순서가 중요하다

**미확인 / 후속**
- 2회차 실행 `changed=0` 여부: 확인 전, 후속 확인 예정. 앱 계정 태스크가 매번 changed 라면 `update_password: on_create` 검토
- `*:33060` (X Protocol)도 모든 인터페이스에서 열려 있다. 방화벽이 막고 있으나 쓰지 않는 포트이므로 닫을지 검토한다 (`mysqlx=OFF` 또는 `mysqlx-bind-address`)
- app01 → db01:3306 연결 성공은 다음 단계(앱 계층 연결)에서 확인한다
- `community.mysql` 모듈 폐기 예정 경고: `ansible.mysql` 로 이관 (제출 이후)

**참조**
- 설치 경로: Rocky 9 AppStream `mysql-server` (설치된 정확한 버전은 아직 확인하지 않음), 소켓 `/var/lib/mysql/mysql.sock`
- collections/requirements.yml 에 `community.mysql` 선언 필요
