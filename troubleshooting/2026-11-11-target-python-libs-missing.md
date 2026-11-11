## 모듈이 대상 서버의 Python 라이브러리를 못 찾아 실패 (packaging, libselinux)

**상태**: 해결. 깨끗한 서버(lab02)에서 플레이북만으로 설치부터 통과를 확인했고, `deploy-web.yml` 2회차 changed=0 도 확인했다. app01 도 이후 `deploy-app.yml` 이 pip 태스크까지 통과했고 재실행에서 ok=11 changed=0 이었다.

**증상 두 가지 (원인 하나)**
1. app01, `deploy-app.yml` 의 pip 태스크
```
Failed to import the required Python library (packaging) on app01's Python /usr/bin/python3.9
```
2. lab01, lab02, `deploy-web.yml` 의 seboolean 태스크
```
Failed to import the required Python library (libselinux-python) on lab01's Python /usr/bin/python3.9
```

**원인**
Ansible 모듈은 대상 서버에서 `ansible_python_interpreter`(여기서는 시스템 Python 3.9)로 실행된다. `ansible.builtin.pip` 은 `packaging`, `ansible.posix.seboolean` 은 SELinux 바인딩(`selinux`, `semanage`)을 **그 3.9 쪽에서** import 한다. Rocky 9 minimal 에는 둘 다 기본 설치되어 있지 않았다.
- 앱의 가상환경(.venv)이나 Python 3.12 와는 무관하다. 모듈 실행용 Python 과 앱 실행용 Python 은 별개다
- 메시지의 `libselinux-python` 은 예전 패키지 이름이고, Rocky 9 의 이름은 `python3-libselinux`, `python3-libsemanage`

**진단 (lab01 에서 직접)**
```bash
rpm -q python3-libselinux python3-libsemanage        # 설치되어 있지 않습니다
/usr/bin/python3.9 -c "import selinux"               # ModuleNotFoundError: No module named 'selinux'
sudo dnf install -y python3-libselinux python3-libsemanage   # appstream 에서 설치됨
/usr/bin/python3.9 -c "import selinux, semanage; print('ok')" # ok
```

**해결**
플레이북이 필요한 패키지를 선언하게 한다 (서버에서 손으로 설치하지 않는다).
```yaml
    - name: SELinux 정책 도구
      ansible.builtin.dnf:
        name:
          - python3-libsemanage
          - python3-libselinux
        state: present
      tags: [selinux]
```
app01 의 pip 태스크는 같은 방식으로 `python3-packaging` 을 설치 태스크의 패키지 목록에 추가한다.

**검증: 손으로 고치지 않은 서버에서 확인**
lab01 은 진단하면서 수동 설치를 해 버려서 "플레이북이 스스로 설치하는지"를 증명하지 못했다. 아직 손대지 않은 lab02 로 확인했다.
```
rpm -q ...                      → 두 패키지 모두 설치되어 있지 않습니다 (실행 전)
--tags selinux --limit lab02    → "Installed: python3-libselinux-3.6-3.el9.aarch64", "Installed: python3-libsemanage-3.6-5.el9_6.aarch64", seboolean changed
deploy-web.yml 전체 1회차        → lab01, lab02 모두 ok=11 changed=2 (nginx 설정 배포, reload 핸들러)
deploy-web.yml 전체 2회차        → lab01, lab02 모두 ok=10 changed=0
```

**원인 미확인 항목**
- 최초 실패 실행에서 dnf 태스크가 `changed` 로 보고됐는데, 이어서 서버의 `rpm -q` 에는 패키지가 없었다. 이 불일치는 설명하지 못했다 (미확인). 이후 lab02 검증에서는 dnf 가 설치 결과(`Installed: ...`)를 정상적으로 보고했다

**참조**
- ansible.builtin.pip (요구 라이브러리 packaging), ansible.posix.seboolean (libselinux-python, libsemanage-python)
- troubleshooting/2026-11-11-when-placeholder-left-in-playbook.md

**교훈**
- 플레이북은 앱 의존성뿐 아니라 **모듈 자신이 대상 서버에서 쓰는 라이브러리**도 선언해야 한다. 이 라이브러리는 서버마다 다시 필요하니, 장기적으로는 bootstrap 의 공통 패키지(`python3-packaging`, `python3-libselinux`)로 옮기는 것이 낫다
- 진단하다 서버를 직접 고치면 그 서버는 검증 대상에서 빠진다. 재현성은 **손대지 않은 서버**에서 확인한다
- 모듈 오류 메시지는 "어느 Python 에서"를 알려준다. 서버에서 `그 Python -c "import ..."` 로 직접 확인하면 원인이 바로 갈린다
