## hansung-agent-rag-ops

Ansible 기반으로 Hansung Agent(RAG) 운영 환경을 구성/관리하는 플레이북입니다.  
현재는 CloudWatch Agent 설치 및 로그/메트릭 수집, CloudWatch Dashboard 반영을 중심으로 구성되어 있습니다.

### 구성

```
hansung-agent-rag-ops/
 ├── ansible.cfg
 ├── inventory/
 │   ├── production.ini
 │   └── staging.ini
 ├── group_vars/
 │   ├── all.yml
 │   └── secrets.yml
 ├── roles/
 │   ├── aws_config/
 │   │   └── tasks/main.yml
 │   ├── common/
 │   │   └── tasks/main.yml
 │   ├── docker_setup/
 │   │   └── tasks/main.yml
 │   └── cloudwatch/
 │       ├── handlers/main.yml
 │       ├── tasks/main.yml
 │       └── templates/
 │           ├── amazon-cloudwatch-agent.json.j2
 │           └── amazon-cloudwatch-llm-observability.j2
 └── site.yml
```

### 사전 준비

- Ansible 실행 환경(로컬/WSL)에서 `ansible-playbook` 사용 가능해야 합니다.
- 대상 EC2 인스턴스(원격)에는 아래가 필요합니다.
  - SSH 접속 가능(보안그룹/키 파일)
  - CloudWatch Dashboard 반영을 위해 인스턴스 프로파일(IAM Role)에 `cloudwatch:PutDashboard` 권한 필요
  - CloudWatch Agent 로그/메트릭 수집을 위해서는 `CloudWatchAgentServerPolicy` 권한을 추가로 권장

### 설정 파일

- `inventory/*.ini`: 대상 서버 IP/접속 정보
- `group_vars/all.yml`: 공통 변수
  - `cw_agent_config_path`: CloudWatch Agent 설정 파일 경로
  - `cw_dashboard_name`, `aws_region`: Dashboard 생성/갱신에 사용
  - `vm_swappiness`, `zram_percent`, `zram_priority`: 메모리 압박 완화(common role) 설정
- `group_vars/secrets.yml`: AWS 접근 키/Bedrock/S3 등 민감 변수 (Ansible-Vault 사용)

### 실행 방법


스테이징 기준:

```bash
ansible-playbook -i inventory/staging.ini site.yml
```

운영 기준:

```bash
ansible-playbook -i inventory/production.ini site.yml
```

태그별 실행 예시:

```bash
ansible-playbook -i inventory/staging.ini site.yml --tags common
ansible-playbook -i inventory/staging.ini site.yml --tags docker
ansible-playbook -i inventory/staging.ini site.yml --tags aws
ansible-playbook -i inventory/staging.ini site.yml --tags cloudwatch
```

### 역할(Role) 설명

#### common (`--tags common`)

메모리 압박 상황에서 서비스가 OOM으로 죽는 위험을 줄이기 위한 역할입니다.

- `zram-tools` 설치
- `/etc/default/zramswap` 설정 배포 (`zstd`, 비율/우선순위 변수화)
- `vm.swappiness` 적용
- `zramswap.service` 활성화 및 설정 변경 시 재시작

#### docker_setup (`--tags docker`)

Docker 실행 기반을 안정적으로 준비하는 역할입니다.

- Docker 공식 apt key/repo 설정
- `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin` 설치
- `/etc/docker/daemon.json` 로그 로테이션 설정(디스크 과사용 방지)
- 접속 사용자를 `docker` 그룹에 추가

#### aws_config (`--tags aws`)

AWS 연동 기본 구성을 담당합니다.

- AWS CLI v2 설치 (Ubuntu 24.04 호환)
- `/root/.aws/config` 및 정적 자격증명(설정 시) 배포
- 정적 자격증명이 없으면 인스턴스 IAM Role 기반 사용

#### cloudwatch (`--tags cloudwatch`)

모니터링/관측 구성을 자동화합니다.

- CloudWatch Agent 설치 및 설정 배포
- Agent 재시작 핸들러 실행
- CloudWatch Dashboard 템플릿 배포 및 `put-dashboard` 적용
