# Zabbix Agent 2 Migration Ansible

기존 Zabbix Agent 1을 Agent 2로 순차 전환하는 Ansible 저장소입니다.
Passive check 설정을 유지하고, 필요한 경우 Active check용 `ServerActive`와 `Hostname`,
기존 `UserParameter`를 이전합니다. Agent 1 패키지와 설정은 삭제하지 않습니다.

## 주요 동작

- Debian/Ubuntu에서 공식 Zabbix 저장소 구성 및 Agent 2 설치
- RHEL 계열에서 사전 구성된 저장소를 이용해 Agent 2 설치
- 요청한 Zabbix 메이저 버전과 실제 패키지 버전 검증
- Agent 1의 직접 설정 및 include 파일에서 `UserParameter=` 이전
- Agent 2 설정과 지정된 아이템 키 검증
- Agent 1 중지 후 Agent 2 시작
- 전환 실패 시 Agent 1 자동 복구
- 선택적으로 `zabbix` 사용자를 Docker 그룹에 추가하고 `docker.info` 검증

## 요구사항

- Debian/Ubuntu 또는 RHEL 계열 Linux
- 기존 Zabbix Agent 1과 `/etc/zabbix/zabbix_agentd.conf`
- 대상 호스트에 대한 SSH 및 root 권한 상승
- Ansible 2.14 이상
- RHEL 계열은 Zabbix 저장소 사전 구성

## 빠른 시작

저장소 루트에서 실행합니다.

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
cp server-vars.yml.example server-vars.yml
vi inventory/hosts.ini
vi server-vars.yml
```

`inventory/hosts.ini`에서 첫 전환 대상을 파일럿 그룹에 지정합니다.

```ini
[zabbix_agents]
server01 ansible_host=10.0.0.11 ansible_user=ansible
server02 ansible_host=10.0.0.12 ansible_user=ansible

[zabbix_agent2_pilot]
server01
```

연결과 구문을 확인합니다.

```bash
ansible zabbix_agents -m ping
ansible-playbook --syntax-check playbooks/migrate.yml
```

파일럿 한 대를 먼저 전환하고 검증합니다.

```bash
ansible-playbook -e @server-vars.yml playbooks/migrate.yml --limit zabbix_agent2_pilot
ansible-playbook playbooks/verify.yml --limit zabbix_agent2_pilot
```

문제가 없으면 전체를 순차 전환합니다. 기본 묶음 크기는 10대입니다.

```bash
ansible-playbook -e @server-vars.yml playbooks/migrate.yml

# 묶음 크기 변경
ansible-playbook -e @server-vars.yml playbooks/migrate.yml \
  -e zabbix_agent2_migration_serial=20
```

롤백은 Agent 2를 중지하고 Agent 1을 다시 활성화합니다.

```bash
ansible-playbook playbooks/rollback.yml --limit server01
```

## 변수

실제 Zabbix Server 주소는 커밋하지 않습니다. `server-vars.yml.example`을 복사한
`server-vars.yml`에 입력하고 migrate 실행 시 `-e @server-vars.yml`로 전달합니다.
이 파일은 Git에서 제외됩니다.

```yaml
---
zabbix_server_passive_allowlist: "10.0.0.10"
zabbix_server_active: "10.0.0.10:10051"
```

나머지 공통값은 `inventory/group_vars/zabbix_agents.yml`에서 관리합니다.

| 변수 | 기본 예시 | 설명 |
|---|---:|---|
| `zabbix_server_passive_allowlist` | `CHANGE_ME` | `server-vars.yml`에서 반드시 지정 |
| `zabbix_server_active` | 빈 문자열 | Active check 대상, 미사용 시 생략 |
| `zabbix_repository_major_version` | `7.0` | 요구하는 Zabbix 메이저 버전 |
| `zabbix_manage_repository` | `true` | Debian/Ubuntu 저장소 자동 관리 |
| `zabbix_agent2_listen_port` | `10050` | Agent 2 passive 포트 |
| `zabbix_agent2_timeout` | `3` | Agent 2 아이템 제한시간 |
| `zabbix_migrate_inline_userparameters` | `true` | 메인 설정의 UserParameter 이전 |
| `zabbix_migrate_include_userparameters` | `true` | include 파일의 UserParameter 이전 |
| `zabbix_agent2_validation_keys` | `agent.ping` | 전환 전후 실행할 키 목록 |
| `zabbix_agent2_manage_docker_access` | `false` | Docker 그룹 권한과 키 검증 활성화 |

호스트별 값은 `inventory/host_vars/<inventory_hostname>.yml`에 지정합니다.

```yaml
---
zabbix_agent2_hostname_override: "server01"
zabbix_agent2_timeout: 15
```

`ServerActive`가 설정됐는데 기존 Agent 1의 메인 설정에서 `Hostname=`을 찾지 못하면
전환을 중단합니다. 이 경우 위처럼 `zabbix_agent2_hostname_override`를 지정합니다.

추가 Agent 2 설정과 커스텀 검증 키도 공통 또는 호스트 변수로 지정할 수 있습니다.

```yaml
zabbix_agent2_extra_config_lines:
  - 'UnsafeUserParameters=1'

zabbix_agent2_validation_keys:
  - agent.ping
  - custom.example
```

## Docker 모니터링 호스트

Docker 템플릿을 사용하는 호스트만 다음 그룹에 추가합니다.

```ini
[zabbix_agent2_docker]
server01

[zabbix_agent2_docker:vars]
zabbix_agent2_manage_docker_access=true
```

Docker 그룹은 높은 시스템 권한을 제공하므로 필요한 호스트에만 사용합니다.

## 비밀번호 SSH와 `su -`

일반 사용자로 로그인한 후 `su - root`가 필요한 환경은 전용 예시를 사용합니다.

```bash
cp inventory/hosts-su.ini.example inventory/hosts.ini
ansible-playbook -e @server-vars.yml playbooks/migrate.yml \
  --limit zabbix_agent2_pilot \
  --ask-pass --ask-become-pass
```

비밀번호를 자동화해야 하면 `ansible_password`와 `ansible_become_password`를
Ansible Vault로 암호화합니다.

## 범위와 주의사항

- Agent 1 패키지와 설정은 롤백을 위해 유지합니다.
- Debian/Ubuntu만 저장소를 자동 구성합니다.
- include 파일에서는 `UserParameter=` 줄만 이전합니다.
- TLS, 기타 Agent 1 설정은 `zabbix_agent2_extra_config_lines` 등에 명시해야 합니다.
- UserParameter가 호출하는 스크립트와 권한은 기존 경로에 남아 있어야 합니다.
- `verify.yml`은 Agent 2 설정, 키, 서비스, 포트와 Agent 1 중지 상태를 확인합니다.
- 충분히 검증한 뒤 Agent 1 패키지를 별도 유지보수 작업으로 제거하십시오.
