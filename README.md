## hansung-agent-rag-ops

Ansible 기반으로 Hansung Agent(RAG) 인프라를 구성/운영하는 플레이북입니다.  
현재는 RAM 최적화, AWS CLI 설정, Docker 실행환경, Nginx+SSL(HTTPS), CloudWatch 관측, Grafana 대시보드 자동화를 포함합니다.

### 프로젝트 구성

```text
hansung-agent-rag-ops/
 ├── ansible.cfg
 ├── requirements.yml
 ├── inventory/
 │   ├── dev.ini
 │   ├── staging.ini
 │   └── production.ini
 ├── group_vars/
 │   ├── all.yml
 │   └── secrets.yml
 ├── playbooks/
 │   └── nginx_clean.yml
 ├── roles/
 │   ├── ram_optimization/
 │   ├── aws_config/
 │   ├── docker_setup/
 │   ├── nginx/
 │   ├── cloudwatch/
 │   └── grafana/
 └── site.yml
```

### 사전 준비

- 로컬/WSL 환경에서 `ansible-playbook` 실행 가능해야 합니다.
- 필요 컬렉션 설치:
  - `ansible-galaxy collection install -r requirements.yml`
- 대상 EC2는 SSH 접속 가능해야 합니다.
- `secrets.yml`은 민감 정보를 포함하므로 `ansible-vault`로 암호화하여 관리하는 것을 권장합니다.

### 주요 설정 파일

- `ansible.cfg`
  - 기본 inventory: `./inventory/staging.ini`
- `inventory/*.ini`
  - 환경별 대상 서버 접속 정보
- `group_vars/all.yml`
  - 공통 변수
  - 예: `aws_region`, `aws_output_format`, `vm_swappiness`, `zram_percent`, `zram_priority`
  - Nginx/SSL 변수
    - `certbot_email`
    - `nginx_apps` (`name`, `internal_port`, `server_name`)
  - CloudWatch 변수
    - `cw_agent_config_path`, `cw_dashboard_name`, `cw_dashboard_region`
  - Grafana 변수
    - `grafana_endpoint`
  - 공통 경로 변수
    - `log_path`, `app_name`
- `group_vars/secrets.yml`
  - 민감 정보 저장(필요 시 Ansible Vault 사용)

### 실행 방법

기본 실행 (스테이징):

```bash
ansible-playbook -i inventory/staging.ini site.yml
```

운영:

```bash
ansible-playbook -i inventory/production.ini site.yml

# 개발 환경
ansible-playbook -i inventory/dev.ini site.yml
```

태그별 실행 예시:

```bash
ansible-playbook site.yml --tags ram
ansible-playbook site.yml --tags aws
ansible-playbook site.yml --tags docker
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --tags cloudwatch
ansible-playbook site.yml --tags grafana
```

### 역할(Role) 설명

#### ram_optimization (`--tags ram`)

메모리 압박 상황에서 서비스가 OOM으로 종료되는 위험을 줄입니다.

- `zram-tools` 설치
- `/etc/default/zramswap` 설정 배포
- `vm.swappiness` 적용
- `zramswap.service` 활성화/재시작

#### aws_config (`--tags aws`)

AWS 연동 기본 구성을 담당합니다.

- AWS CLI v2 설치
- `/root/.aws/config` 배포
- 정적 자격증명 사용 시 `/root/.aws/credentials` 배포
- 정적 자격증명이 없으면 인스턴스 IAM Role 기반 사용

#### docker_setup (`--tags docker`)

Docker 실행 기반을 준비합니다.

- Docker apt key/repo 구성
- `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin` 설치
- `/etc/docker/daemon.json` 로그 로테이션 설정
- 접속 사용자를 `docker` 그룹에 추가 및 `newgrp docker` 적용

#### nginx (`--tags nginx`)

Nginx 리버스 프록시 및 HTTPS 구성을 자동화합니다.

- Nginx 설치
- Certbot(`python3-certbot-nginx`) 설치
- `nginx_apps` 루프 기반 인증서 발급 (`certbot certonly --nginx`)
- **SSL 자동 갱신 설정**: 매일 03:17에 `certbot renew`를 실행하도록 크론탭(Cron) 등록
  - 갱신 성공 시 `systemctl reload nginx`를 통해 인증서 즉시 반영
- `/etc/nginx/sites-available/nginx_apps.conf` 템플릿 배포
- 기본 사이트 비활성화, 심볼릭 링크 생성, `nginx -t` 문법 검사
- 설정 변경 시 `reload nginx` 핸들러 동작

#### cloudwatch (`--tags cloudwatch`)

모니터링/관측 구성을 자동화합니다.

- CloudWatch Agent 설치 및 설정 배포
- Agent 재시작 핸들러 실행
- Dashboard JSON 템플릿 배포 및 `put-dashboard` 적용

#### grafana (`--tags grafana`)

Grafana 데이터소스/대시보드 구성을 자동화합니다.

- CloudWatch 데이터소스 생성/갱신
  - 정적 키가 있으면 `keys` 인증
  - 없으면 인스턴스/IAM Role 기반 `default` 인증
- Datasource UID를 조회 후 대시보드 JSON에 치환
- `llm-observability` 대시보드 import 및 결과 검증

### Nginx 앱 추가 방법

`group_vars/all.yml`의 `nginx_apps`에 항목을 추가하면 됩니다.

```yaml
nginx_apps:
  - name: "rag-be"
    internal_port: 8080
    server_name: "rag-be.yeoun.org"
  - name: "another-app"
    internal_port: 8001
    server_name: "example.com"
```
