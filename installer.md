# AWS Infrastructure Installer

boto3를 사용하여 AWS 인프라 리소스를 생성하는 Python 스크립트입니다.  
[langgraph-ecs-project](./)의 Streamlit + LangGraph Agent 애플리케이션을 **ECS Fargate**에 배포하고, Bedrock Knowledge Base·AgentCore Memory·**S3 Files 세션 스토리지**를 함께 프로비저닝합니다.

## 목차

1. [개요](#개요)
2. [설정값](#설정값)
3. [생성되는 리소스](#생성되는-리소스)
4. [주요 함수](#주요-함수)
5. [실행 방법](#실행-방법)
6. [배포 순서](#배포-순서)
7. [application/config.json](#applicationconfigjson)
8. [배포 완료 후](#배포-완료-후)

---

## 개요

이 스크립트는 LangGraph 기반 AI 채팅 애플리케이션을 위한 전체 AWS 인프라를 자동으로 생성합니다.

### 아키텍처 요약

```
사용자 → CloudFront → ALB → ECS Fargate (Streamlit, private subnet)
                              ├─ Bedrock (VPC endpoint)
                              ├─ OpenSearch Serverless (Knowledge Base)
                              ├─ AgentCore Memory / Web Search Gateway
                              └─ S3 Files 마운트 (/mnt/workspace) → LangGraph SQLite checkpoint
S3 버킷 ← docs/, artifacts/, agentcore-sessions/ (S3 Files 동기화)
```

### 주요 특징

- **ECS Fargate 배포**: Docker 이미지를 ECR에 push하고 Fargate 서비스로 운영
- **S3 Files 세션 스토리지**: Agent (Chat) 모드 checkpoint를 `/mnt/workspace`에 영속화 ([langgraph-runtime](https://github.com/kyopark2014/langgraph-runtime)과 동일 패턴, ECS `s3filesVolumeConfiguration` 사용)
- **완전 자동화**: 단일 스크립트로 전체 인프라 배포
- **멱등성**: 이미 존재하는 리소스는 재사용
- **부분 실패 복구**: `finally` 블록에서 `application/config.json`을 가능한 범위까지 갱신

---

## 설정값

```python
project_name = "langgraph-ecs-project"   # 프로젝트 이름 (최소 3자)
region = "us-west-2"
git_name = "langgraph-ecs-project"
AGENTCORE_GATEWAY_REGION = "us-east-1"
AGENTCORE_WEBSEARCH_GATEWAY_NAME = "gateway-websearch"

# 자동 생성
account_id = sts_client.get_caller_identity()["Account"]
bucket_name = f"storage-for-{project_name}-{account_id}-{region}"
vector_index_name = project_name

# S3 Files (LangGraph checkpoint 영속화)
S3_FILES_SESSION_PREFIX = "agentcore-sessions/"
SESSION_STORAGE_MOUNT_PATH = "/mnt/workspace"
S3_FILES_VOLUME_NAME = "session-storage"

# CloudFront ↔ ALB 커스텀 헤더
custom_header_name = "X-Custom-Header"
custom_header_value = f"{project_name}_12dab15e4s31"
```

---

## 생성되는 리소스

### 1. S3 버킷

- **이름**: `storage-for-{project_name}-{account_id}-{region}`
- **설정**:
  - CORS 활성화 (GET, POST, PUT)
  - 퍼블릭 액세스 차단
  - **버저닝 `Enabled`** (S3 Files file system 요구사항)
  - `docs/`, `artifacts/` 폴더 자동 생성
- **S3 Files 동기화 prefix**: `agentcore-sessions/` (LangGraph checkpoint SQLite 등)

### 2. IAM 역할

| 역할 | 설명 |
|------|------|
| `role-knowledge-base-for-{project_name}-{region}` | Bedrock Knowledge Base |
| `role-agent-for-{project_name}-{region}` | Bedrock Agent |
| `role-ecs-task-for-{project_name}-{region}` | ECS 태스크 역할 (Bedrock, S3, OpenSearch, S3 Files mount 등) |
| `role-ecs-execution-for-{project_name}-{region}` | ECS 실행 역할 (ECR pull, CloudWatch Logs) |
| `role-agentcore-memory-for-{project_name}-{region}` | AgentCore Memory |
| `role-agentcore-gateway-websearch-for-{project_name}` | AgentCore Web Search Gateway (`us-east-1`) |
| `role-s3files-sync-for-{project_name}` | S3 Files ↔ S3 버킷 동기화 (EFS trust) |
| `role-lambda-rag-for-{project_name}-{region}` | Lambda RAG (헬퍼, 필요 시) |
| `role-ec2-for-{project_name}-{region}` | 레거시 EC2용 (현재 main 배포 경로에서는 미사용) |

ECS task role 인라인 정책 예: Knowledge Base, S3, OpenSearch, Bedrock, **`s3files-policy-for-{project_name}`** (S3 Files mount)

### 3. AgentCore Memory

- Memory 인스턴스 생성 또는 기존 인스턴스 재사용
- `application/config.json`에 `memory_id`, `agentcore_memory_role` 저장

### 4. AgentCore Web Search Gateway

- 리전: `us-east-1` (`bedrock-agentcore-control`)
- Gateway 이름: `gateway-websearch`
- `application/config.json`에 gateway ID·URL·role ARN 저장

### 5. OpenSearch Serverless

- **컬렉션**: Vector 검색용 서버리스 컬렉션 (`langgraph-ecs-project`)
- **정책**: 암호화, 네트워크, 데이터 액세스 (ECS task role + Knowledge Base role)
- **인덱스**: KNN 벡터 검색 (1024차원, Titan Embed v2)

### 6. Bedrock Knowledge Base

- **스토리지**: OpenSearch Serverless
- **임베딩**: Amazon Titan Embed Text v2
- **데이터 소스**: 위 S3 버킷 (`docs/` 등)

### 7. VPC 네트워킹

```
VPC (10.20.0.0/16)
├── Public Subnets (2 AZ)
│   ├── Internet Gateway
│   └── NAT Gateway
├── Private Subnets (2 AZ)
│   ├── ECS Fargate 태스크
│   └── S3 Files mount targets (NFS 2049)
├── Security Groups
│   ├── alb-sg-for-{project_name} (80)
│   ├── ecs-sg-for-{project_name} (8501 from ALB, 443)
│   └── s3files-mount-sg-for-{project_name} (2049 from ECS SG)
└── VPC Endpoints
    └── bedrock-runtime (private subnet, ECS SG)
```

### 8. S3 Files 세션 스토리지 `[5.5/10]`

LangGraph **Agent (Chat)** checkpoint를 ECS 태스크 재시작 후에도 유지하기 위한 리소스입니다.

| 리소스 | 설명 |
|--------|------|
| File system | S3 버킷 + `agentcore-sessions/` prefix |
| Mount targets | private subnet별 NFS 엔드포인트 |
| Access point | POSIX uid/gid `0/0`, `/mnt/workspace` 마운트용 |
| File system policy | ECS task role에 `ClientMount` / `ClientWrite` 허용 |
| Sync IAM role | `role-s3files-sync-for-{project_name}` |

### 9. Application Load Balancer

- **이름**: `alb-for-{project_name}`
- **타입**: Internet-facing ALB (HTTP 80)
- **타겟 그룹**: `TG-for-{project_name}` — **IP 타입**, 포트 8501 (ECS Fargate)

### 10. CloudFront 배포

- **기본 오리진**: ALB (Streamlit 동적 컨텐츠)
- **경로 오리진**: `/images/*`, `/docs/*`, `/artifacts/*` → S3
- **OAI** + S3 버킷 정책으로 정적 객체 제공

### 11. ECR · ECS Fargate `[8–9/10]`

| 리소스 | 이름/설정 |
|--------|-----------|
| ECR repository | `ecr-for-{project_name}` |
| ECS cluster | `cluster-for-{project_name}` |
| ECS service | `service-for-{project_name}` (desiredCount=1, private subnet) |
| Task definition | `task-for-{project_name}` — cpu 1024, memory 2048 |
| Container | `app` — 포트 8501, `APP_CONFIG_JSON`, `SESSION_STORAGE_DIR=/mnt/workspace` |
| S3 Files volume | `session-storage` → container `/mnt/workspace` |
| CloudWatch Logs | `/ecs/app-for-{project_name}` |

Docker 이미지는 [Dockerfile](./Dockerfile) 기준으로 `linux/amd64` 빌드 후 ECR에 push합니다.

---

## 주요 함수

### 인프라 생성

| 함수 | 단계 | 설명 |
|------|------|------|
| `create_s3_bucket()` | [1/10] | S3 버킷, CORS, versioning, `docs/`·`artifacts/` |
| `create_knowledge_base_role()` 등 | [2/10] | IAM 역할 (KB, Agent, ECS, Memory, Gateway) |
| `create_ecs_roles()` | [2/10] | ECS task / execution role |
| `create_agentcore_memory()` | [2/10] | AgentCore Memory 인스턴스 |
| `get_or_create_agentcore_websearch_gateway()` | [2/10] | Web Search Gateway |
| `create_opensearch_collection()` | [3/10] | OpenSearch Serverless 컬렉션·정책 |
| `create_knowledge_base_with_opensearch()` | [4/10] | Bedrock Knowledge Base + S3 데이터 소스 |
| `create_vpc()` | [5/10] | VPC, subnet, NAT, ALB/ECS SG, Bedrock VPC endpoint |
| `create_s3_files_session_storage()` | [5.5/10] | S3 Files file system, mount targets, access point |
| `attach_ecs_task_s3files_policy()` | [5.5/10] | ECS task role S3 Files IAM |
| `create_alb()` | [6/10] | Application Load Balancer |
| `create_cloudfront_distribution()` | [7/10] | CloudFront (ALB + S3 hybrid) |
| `create_ecr_repository()` | [8/10] | ECR repository |
| `build_and_push_docker_image()` | [8/10] | Docker build & ECR push |
| `create_ecs_log_group()` | [9/10] | CloudWatch log group |
| `deploy_ecs_service()` | [9/10] | Task definition(S3 Files volume) + Fargate service |
| `build_app_environment()` | — | 컨테이너 `APP_CONFIG_JSON` 페이로드 |
| `apply_s3_files_config()` | — | config.json에 S3 Files 키 병합 |
| `write_application_config()` | — | `application/config.json` 저장 |
| `check_application_ready()` | — | CloudFront URL 헬스 체크 |

### S3 Files 관련 (내부)

| 함수 | 설명 |
|------|------|
| `_get_or_create_s3files_sync_role()` | S3 Files 동기화 IAM role |
| `_get_or_create_s3files_file_system()` | File system (`agentcore-sessions/` prefix) |
| `_ensure_s3files_mount_targets()` | Private subnet mount target |
| `_get_or_create_s3files_access_point()` | Access point 생성/재사용 |
| `_ensure_s3files_file_system_policy()` | ECS task role용 resource policy |
| `_s3files_ecs_task_policy_document()` | Task role IAM policy document |

### 레거시 (main 배포 경로 외)

| 함수 | 설명 |
|------|------|
| `create_ec2_instance()` | EC2 기반 배포 (레거시, `--run-setup` 등) |
| `run_setup_on_existing_instance()` | SSM으로 EC2 setup 스크립트 실행 |

---

## 실행 방법

### 기본 실행 (전체 인프라 배포)

```bash
pip install -r requirements.txt
python installer.py
```

로컬에 Docker가 필요합니다 (이미지 빌드·ECR push).

### Docker 빌드 생략 (기존 ECR 이미지 사용)

```bash
python installer.py --skip-docker-build
```

### 레거시 EC2 옵션

```bash
python installer.py --run-setup              # EC2에 setup 스크립트 (SSM)
python installer.py --run-setup i-xxxxxxxx   # 특정 인스턴스 ID
python installer.py --verify-deployment      # EC2 private subnet 검증
```

인프라 제거는 [uninstaller.py](./uninstaller.py)를 사용합니다.

```bash
python uninstaller.py --yes
```

---

## 배포 순서

```
[1/10] S3 버킷 생성 (versioning Enabled)
       ↓
[2/10] IAM 역할 생성
       • Knowledge Base, Agent, ECS task/execution
       • AgentCore Memory, Web Search Gateway
       ↓
[3/10] OpenSearch Serverless 컬렉션
       ↓
[4/10] Bedrock Knowledge Base + S3 데이터 소스
       ↓
[5/10] VPC (public/private subnet, NAT, ALB/ECS SG, Bedrock VPC endpoint)
       ↓
[5.5/10] S3 Files 세션 스토리지 (ECS /mnt/workspace)
       • sync role, file system, mount targets, access point
       • ECS task role S3 Files IAM + file system policy
       ↓
[6/10] Application Load Balancer
       ↓
[7/10] CloudFront (ALB + S3 hybrid)
       ↓
[8/10] ECR repository + Docker build/push
       ↓
[9/10] ECS Fargate 서비스 배포
       • Task definition (S3 Files volume mount)
       • ALB target group (IP) 연결
       • application/config.json 갱신
       ↓
애플리케이션 준비 상태 확인 (CloudFront)
       ↓
완료 — 배포 요약 출력
```

---

## application/config.json

installer가 생성·갱신하는 주요 키:

| 키 | 설명 |
|----|------|
| `projectName`, `accountId`, `region` | 프로젝트 메타 |
| `knowledge_base_id`, `knowledge_base_role` | Bedrock KB |
| `collectionArn`, `opensearch_url` | OpenSearch Serverless |
| `s3_bucket`, `s3_arn` | 스토리지 버킷 |
| `sharing_url` | CloudFront URL |
| `agentcore_memory_role`, `memory_id` | AgentCore Memory |
| `agentcore_websearch_gateway_*` | Web Search Gateway |
| `s3_files_file_system_id` | S3 Files file system |
| `s3_files_access_point_arn` | S3 Files access point |
| `s3_files_mount_path` | `/mnt/workspace` |
| `ecs_session_vpc_subnets` | ECS private subnets |
| `ecs_session_security_groups` | ECS security group |

컨테이너에는 동일 내용이 환경 변수 `APP_CONFIG_JSON`으로 전달됩니다.

---

## 배포 완료 후

배포가 완료되면 다음과 유사한 요약이 출력됩니다:

```
================================================================
Infrastructure Deployment Completed Successfully!
================================================================
Summary:
  S3 Bucket: storage-for-langgraph-ecs-project-{account_id}-us-west-2
  VPC ID: vpc-xxxxxxxxx
  Public Subnets: subnet-xxx, subnet-yyy
  Private Subnets: subnet-aaa, subnet-bbb
  ALB DNS: http://alb-for-langgraph-ecs-project-....elb.amazonaws.com/
  CloudFront Domain: https://xxxxxxxxx.cloudfront.net
  ECS Service: service-for-langgraph-ecs-project (Fargate in private subnet)
  ECR Image: {account}.dkr.ecr.us-west-2.amazonaws.com/ecr-for-langgraph-ecs-project:latest
  S3 Files Access Point: arn:aws:s3files:...
  ECS Session Mount Path: /mnt/workspace
  OpenSearch Endpoint: https://....aoss.amazonaws.com
  Knowledge Base ID: XXXXXXXXXX
  AgentCore Memory ID: mem-xxxxxxxx
================================================================
```

### Docker 컨테이너

ECS 태스크는 [Dockerfile](./Dockerfile)로 빌드된 이미지를 실행합니다.

- **Base**: `python:3.13-slim`
- **런타임**: Streamlit (`application/app.py`, 포트 8501)
- **Agent**: LangGraph, MCP, Skills, Bedrock
- **Checkpoint**: `langgraph-checkpoint-sqlite`, `aiosqlite`
- **세션 스토리지**: `SESSION_STORAGE_DIR=/mnt/workspace` (S3 Files 마운트)

### 주의사항

- CloudFront·ECS 서비스 안정화에 **15–20분** 걸릴 수 있습니다.
- S3 Files → S3 동기화에는 **~60초** 지연이 있을 수 있습니다.
- `Agent (Chat)` 모드 + User ID별 checkpoint는 `/mnt/workspace/{user_id}/langgraph_checkpoints.sqlite`에 저장됩니다.
- 배포 실패 시에도 `finally`에서 `application/config.json`이 부분적으로 저장될 수 있습니다.

---

## 에러 처리

| 상황 | 처리 방법 |
|------|----------|
| 리소스 이미 존재 | 기존 리소스 재사용 |
| S3 Files 리소스 존재 | file system / access point / mount target 재사용 |
| 서브넷 부족 | 자동 생성 또는 기존 VPC 서브넷 분류 |
| CIDR 충돌 | 대체 CIDR 자동 선택 |
| OpenSearch 정책 이름 길이 초과 | SHA256 digest로 축약 |
| Docker 미설치 | `build_and_push_docker_image()` 실패 — Docker CLI 필요 |
| ECS 서비스 이미 존재 | `update_service` + `forceNewDeployment` |

배포 실패 시 에러 메시지와 스택 트레이스가 출력됩니다. S3 Files·VPC 관련 오류는 Security Group(NFS 2049)·private subnet·버킷 versioning을 확인하세요.

---

## 관련 문서

- [README.md](./README.md) — AgentCore Memory, S3 Files Session Storage, 운영 가이드
- [uninstaller.py](./uninstaller.py) — 위 리소스 역순 삭제 (ECS, S3 Files 포함)
