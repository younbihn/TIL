# 사용자와 그룹

## 명령어 문법 기본
- 구조: `명령어 [옵션] [인자]`
- `-m` 단축형 / `--create-home` 완전형, 단축형은 `-aG`처럼 합칠 수 있음
- 모르면: `man 명령어` (q 종료, /검색), `명령어 --help`

## useradd — 사용자 생성
sudo useradd -m -s /bin/bash testuser
- -m: 홈 디렉토리 생성 (없으면 로그인 시 문제)
- -s: 로그인 셸 지정. 서비스 계정엔 /sbin/nologin으로 로그인 차단
- passwd <계정>: 비밀번호 설정

## usermod -aG — 그룹 추가
- -a(append) 없이 -G만 쓰면 기존 그룹이 전부 덮어써짐 ⚠️
- 그룹의 존재 이유: 권한을 개인이 아닌 그룹 단위로 관리

## 사용자 정보 저장 위치
/etc/passwd 7필드: 계정명:x:uid:gid:설명:홈:셸
- 리눅스는 이름이 아니라 uid로 사용자를 식별. uid 0 = root
- 비밀번호 해시는 /etc/shadow에 분리 (passwd는 전체 공개, shadow는 root 전용)

## 오늘 확인한 것
- (직접: sudo 없이 /etc/shadow 열어본 결과 → )

## 권한 (permission)

### ls -l 읽기
-rw-r--r--  → [종류][소유자rwx][그룹rwx][기타rwx]
- 첫 글자: - 일반파일 / d 디렉토리
- r=읽기 w=쓰기 x=실행

### chmod — 권한 변경
- 숫자: r4 w2 x1 을 더함. 640 = rw-r-----
  - 8진수인 이유: r/w/x가 3비트라 0~7 조합이 1:1 대응
  - 자주 쓰는 값: 644(파일) 755(실행/디렉토리) 600(비밀)
- 기호: u/g/o/a + +추가 -제거 =지정 (예: chmod u+x,g-r)

### chown — 소유권 변경 (소유자:그룹), sudo 필요
### umask — 새 파일에서 빼앗을 권한. 022면 파일 644 / 디렉토리 755
  - 파일 기본이 666인 이유: 새 파일이 자동 실행되면 위험하므로 x는 수동으로만

### 실험 결과
- 디렉토리의 x = 진입/접근 권한, r = 목록 조회 권한 (x 없으면 cd 불가)
- 경로상 모든 디렉토리에 x가 있어야 파일에 도달 가능
  → "권한 줬는데 왜 안 되지?"의 주원인

## chown vs chmod
- chown = change owner : "누구의 것인가" (주체)
- chmod = change mode  : "무엇을 할 수 있는가" (행위)
- 비유: chown=등기부 소유자 변경 / chmod=현관 잠금 설정
- 둘은 짝: rw-r----- 라는 설정(chmod)도 그룹이 누구냐(chown)에 따라 실효가 달라짐

| | chown | chmod |
|---|---|---|
| 대상 | 소유자/그룹 | r/w/x 권한 |
| 문법 | chown user:group 대상 | chmod 644 대상 |
| sudo | 거의 항상 필요 | 소유자면 불필요 |
| ls -l 위치 | 4~5번째 칸 | 맨 앞 10글자 |

- chown이 sudo를 요구하는 이유: 소유권 떠넘기기(할당량 회피)·탈취 방지
- 옵션: chown -R (재귀). 경로 오타 시 피해가 크므로 실행 전 확인

## sudoers
### 확인
sudo -l            # 내게 허용된 명령 목록
sudo -l -U 사용자   # 특정 사용자 기준 조회
sudo visudo -c     # 문법 검사

### 편집 — 반드시 visudo
sudo visudo                            # 메인 /etc/sudoers
sudo visudo -f /etc/sudoers.d/사용자명   # 개별 파일 (실무 방식)
- vi로 직접 열면 안 되는 이유: visudo는 저장 시 문법 검사 → 오타로 sudo 전체 마비 방지
- 개별 파일로 나누는 이유: 충돌 방지, OS 업데이트 시 보존, 자동화(Ansible)와 궁합

### 문법
사용자 호스트=(실행주체) 명령
testuser ALL=(ALL) /usr/bin/systemctl status sshd
- 명령은 절대경로. ALL이면 전체 허용
- %그룹명 으로 그룹 단위 지정
- NOPASSWD: 비번 생략 (자동화 계정용, 남용 시 위험)

### 권한 부여/회수 (RHEL 계열 간편법)
sudo usermod -aG wheel 사용자     # wheel 그룹 = 관리자 (기본 규칙 내장)
sudo gpasswd -d 사용자 wheel      # 회수
→ 재로그인해야 적용 (그룹은 로그인 시 읽힘)
sudo rm /etc/sudoers.d/사용자     # 개별 파일 방식 회수

### 안전 수칙
- 수정 중엔 별도 터미널에 root 세션 유지 (잠금 시 복구용)

## 실험 B 결과 (2026-08-12)
- file.txt에 그룹(developers) 읽기 권한을 줬으나 testuser는 "허가 거부"
- 원인: /home/minkyluv 가 drwx------ (700) → 경로 진입 불가
- 교훈: 파일 권한만으로 부족. 경로상 모든 디렉토리에 x가 있어야 도달 가능
- 부수 발견: testuser는 sudoers 미등록 상태 → "sudoers 파일에 없습니다"

## sudo의 거절 방식
허용되지 않은 명령이어도 sudo는 비밀번호부터 요구한 뒤 거절한다.
→ 즉시 거절하면 "어떤 명령이 허용되는지" 정보가 새어나가기 때문 (탐색 방지)
→ 비밀번호가 없는 계정은 허용 목록 외의 명령을 아예 실행할 수 없음

## sudo -l 출력 읽기
    (ALL) NOPASSWD: /usr/bin/systemctl
- (ALL)      : 어떤 사용자 자격으로든 실행 가능 (root 포함)
- NOPASSWD:  : 비밀번호를 묻지 않음
- 뒤의 경로   : 허용된 명령. 이것만 실행 가능
기본 항목 중 알아둘 것:
- env_reset   : sudo 실행 시 환경변수 초기화 (조작된 변수로 권한 탈취 방지)
- secure_path : 명령 탐색 경로 고정 (가짜 실행파일 실행 방지)

## 배포용 계정의 표준 형태 (최소 권한 원칙)
- 비밀번호 미설정
- SSH 키 인증만 허용
- sudo는 필요한 명령 하나만 NOPASSWD로 허용
→ 계정이 탈취돼도 피해 범위가 그 명령으로 제한됨
→ 자동화 도구(Ansible 등)가 사용할 계정의 표준 구성