## --limit 로 새 서버를 지정했는데 skipping: no hosts matched

**상태**: 해결. 인벤토리 등록과 플레이 대상 확대 후 app01, db01 에서 태스크가 실행됐다.

**증상**
[맥] $ ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --limit app01,db01 -K
PLAY [서버 초기 구성] ****
skipping: no hosts matched
PLAY RECAP ****  (비어 있음)
- hardening.yml 도 같은 결과

**원인**
두 가지가 연달아 있었다.
1. 처음에는 `inventory/hosts.yml` 에 app01, db01 이 **등록되어 있지 않았다**
2. 인벤토리를 고쳐도 플레이의 `hosts: webservers` 때문에 `--limit` 이 **교집합**(webservers ∩ app01,db01)을 만들어 여전히 비었다. `--limit` 은 대상을 **좁히기만** 하고 넓히지 못한다

**해결**
- 인벤토리에 `appservers`(app01), `dbservers`(db01) 그룹과 둘을 포함하는 `threetier` 그룹 추가
- `bootstrap.yml`, `hardening.yml` 의 `hosts: webservers` 를 `hosts: threetier` 로 변경 (자료의 안은 `all`)
```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --limit app01,db01 --list-hosts   # 실제 대상 먼저 확인
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --limit app01,db01 -K
```
- `--list-hosts` 는 서버에 접속하지 않고 대상 목록만 보여준다

**참조**
- man ansible-playbook (--limit, --list-hosts), ansible/inventory-groups-vars.md

**교훈**
- "no hosts matched" 는 인벤토리 문제와 플레이 대상 문제를 둘 다 의심한다. `ansible-inventory --graph` 로 호스트가 그룹에 있는지, 플레이의 `hosts:` 가 그 그룹을 포함하는지 확인한다
- 실행 전에 `--list-hosts` 를 붙이는 습관이 이 혼란을 줄인다
- 일부 플레이북(`deploy-web.yml`, `patch.yml`)은 아직 `webservers` 대상이다. 서버가 늘 때마다 각 플레이의 대상이 맞는지 따로 본다
