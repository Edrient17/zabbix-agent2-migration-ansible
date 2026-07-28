# Zabbix Agent 2 Migration Ansible

Zabbix Agent 1을 Agent 2로 순차 전환하는 Ansible 저장소입니다.
공통 `Server` 설정으로 Passive check를 유지하면서 필요한 경우 `ServerActive`와
기존 `Hostname`도 이전합니다.

## 동작

1. Agent 2 패키지 설치
2. 공통 `Server` 설정을 적용해 Passive check 유지
3. Agent 1의 메인 설정에 직접 작성된 `UserParameter` 이전
4. 일반적인 Agent 1 include 디렉터리의 `*.conf`에서 `UserParameter=` 줄 이전
5. 기존 Agent 1의 `Hostname`을 서버별로 Agent 2에 이전
6. 공통 `ServerActive` 설정 적용
7. Agent 2 설정 및 아이템 키 검증
8. Agent 1 중지 후 Agent 2 시작
9. 실패 시 Agent 1 자동 재시작

Agent 1 패키지와 설정은 삭제하지 않으므로 별도 롤백 플레이북을 사용할 수 있습니다.

## 전제조건

- Debian/Ubuntu 또는 RHEL 계열 Linux
- 대상 서버에 기존 Zabbix Agent 1이 설치되어 있음
- 대상 서버에 Zabbix 7.0 공식 패키지 저장소가 설정되어 있음
- Ansible 제어 노드에서 SSH 및 root 권한 상승 가능
- Passive check용 `10050/TCP` 정책은 기존 상태 유지

이 저장소는 Zabbix 패키지 저장소 자체를 추가하지 않습니다. 운영체제와 버전에 맞는
Zabbix 7.0 공식 저장소를 먼저 설정해야 합니다.

## 1. 인벤토리 준비

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
vi inventory/hosts.ini
```

먼저 전환할 한 대를 `zabbix_agent2_pilot` 그룹에 지정합니다.

```ini
[zabbix_agents]
server01 ansible_host=10.0.0.11 ansible_user=ansible
server02 ansible_host=10.0.0.12 ansible_user=ansible

[zabbix_agent2_pilot]
server01
```

## 2. 변수 설정

`group_vars/zabbix_agents.yml`에서 Zabbix Server 주소를 변경합니다.

```yaml
zabbix_server_passive_allowlist: "10.0.0.10"
zabbix_server_active: "10.0.0.10:10051"
```

`zabbix_server_active`는 모든 대상 서버에 공통 적용됩니다. `Hostname`은 기존
`zabbix_agentd.conf`에서 자동 추출하므로 서버별로 다시 작성할 필요가 없습니다.
특정 호스트의 이름을 바꿔야 한다면 `host_vars`에서 지정할 수 있습니다.

```yaml
zabbix_agent2_hostname_override: "server01"
```

Active check를 사용하지 않을 대상 그룹은 다음처럼 비활성화할 수 있습니다.

```yaml
zabbix_server_active: ""
```

기존 Agent 1의 `Include` 경로가 다르면 다음 목록을 수정합니다.

```yaml
zabbix_agent1_include_dirs:
  - /etc/zabbix/zabbix_agentd.d
  - /etc/zabbix/zabbix_agentd.conf.d
```

`UserParameter` 외에 필요한 설정은 명시적으로 추가합니다.

```yaml
zabbix_agent2_extra_config_lines:
  - 'UnsafeUserParameters=1'
```

커스텀 UserParameter 키도 검증 목록에 추가합니다.

```yaml
zabbix_agent2_validation_keys:
  - agent.ping
  - custom.example
```

## 3. 연결 및 구문 확인

```bash
ansible zabbix_agents -m ping
ansible-playbook --syntax-check playbooks/migrate.yml
```

SSH 또는 권한 상승 암호가 필요하면 `--ask-pass --ask-become-pass`를 추가합니다.

## 4. 한 대에서 먼저 전환

```bash
ansible-playbook playbooks/migrate.yml --limit zabbix_agent2_pilot
```

Zabbix Server에서 확인합니다.

```bash
zabbix_get -s <AGENT_IP> -p 10050 -k agent.ping
```

웹 UI의 최신 데이터와 Agent 2 로그도 확인합니다.

```bash
journalctl -u zabbix-agent2 --since "10 minutes ago"
```

## 5. 전체 순차 전환

기본값은 한 번에 10대입니다.

```bash
ansible-playbook playbooks/migrate.yml
```

실행 시 묶음 크기를 변경할 수 있습니다.

```bash
ansible-playbook playbooks/migrate.yml \
  -e zabbix_agent2_migration_serial=20
```

## 검증

```bash
ansible-playbook playbooks/verify.yml
```

## 롤백

```bash
ansible-playbook playbooks/rollback.yml --limit server01
```

전체 롤백:

```bash
ansible-playbook playbooks/rollback.yml
```

롤백은 Agent 2를 중지하고 Agent 1을 다시 활성화합니다. Agent 2 패키지와 설정은
삭제하지 않습니다.

## 주의사항

- `ServerActive`가 설정되면 기존 Agent 1의 `Hostname`을 Agent 2에 자동 이전합니다.
- Active check의 `Hostname`은 Zabbix UI의 호스트명과 정확히 일치해야 합니다.
- include 디렉터리의 `*.conf`에서는 `UserParameter=` 줄만 추출합니다.
- 그 밖의 설정은 `zabbix_agent2_extra_config_lines`에 명시해야 합니다.
- UserParameter가 호출하는 스크립트와 실행 권한은 기존 경로에 그대로 존재해야 합니다.
- Agent 1 패키지는 충분히 검증한 후 별도 유지보수 작업으로 제거하십시오.
