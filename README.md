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
 │   ├── cloudwatch/
 │   │   ├── tasks/main.yml
 │   │   └── templates/
 │   │       ├── amazon-cloudwatch-agent.json.j2
 │   │       └── amazon-cloudwatch-llm-observability.j2
 │   └── 
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

### CloudWatch 역할(role)

`roles/cloudwatch`는 다음을 수행합니다.

- CloudWatch Agent 설치
- Agent 설정 JSON 배포
- CloudWatch Dashboard JSON 배포 및 `put-dashboard`로 대시보드 생성/갱신

### AWS 설정 역할(role)

`roles/aws_config`는 다음을 수행합니다.

- AWS CLI v2 설치 (Ubuntu 24.04 호환)
- `/root/.aws/credentials`, `/root/.aws/config` 배포
- AWS CLI 버전 확인
