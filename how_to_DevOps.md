# 🚀 FastAPI + Cloud Deployment 완전 가이드

## 📋 목차
1. [프로젝트 구조 이해](#1-프로젝트-구조-이해)
2. [로컬 개발 환경 설정](#2-로컬-개발-환경-설정)
3. [Docker 컨테이너화](#3-docker-컨테이너화)
4. [Google Cloud Platform (GCP) 배포](#4-google-cloud-platform-gcp-배포)
5. [AWS 배포](#5-aws-배포)
6. [Azure 배포](#6-azure-배포)
7. [CI/CD 파이프라인 구축](#7-cicd-파이프라인-구축)
8. [모니터링 및 로깅](#8-모니터링-및-로깅)
9. [문제 해결 가이드](#9-문제-해결-가이드)
10. [보안 강화](#10-보안-강화)
11. [데이터베이스 운영 전략](#11-데이터베이스-운영-전략)
12. [Observability](#12-observability)
13. [테스트 전략](#13-테스트-전략)
14. [고급 운영 전략](#14-고급-운영-전략)


---

## 1. 프로젝트 구조 이해

### 📁 기본 디렉토리 구조
```
your-project/
├── app/                    # 메인 애플리케이션 코드
│   ├── main.py            # FastAPI 앱 진입점
│   ├── api/               # API 라우터들
│   ├── models/            # 데이터 모델
│   ├── services/          # 비즈니스 로직
│   └── utils/             # 유틸리티 함수들
├── database/              # 데이터베이스 관련
├── oauth/                 # OAuth 인증
├── pages/                 # Streamlit 페이지들
├── uploads/               # 업로드된 파일들
├── requirements.txt       # Python 의존성
├── Dockerfile            # Docker 이미지 정의
├── docker-compose.yml    # 로컬 개발용 Docker
├── .env.example          # 환경 변수 예시
├── .gitignore           # Git 무시 파일
├── README.md            # 프로젝트 설명
└── cloudbuild.yaml      # GCP Cloud Build 설정
```

### 🔧 핵심 파일들의 역할

#### `main.py` - FastAPI 애플리케이션 진입점
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Your API", version="1.0.0")

# CORS 설정
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
def read_root():
    return {"message": "Hello World"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

#### `requirements.txt` - 의존성 관리
```txt
# 핵심 웹 프레임워크
fastapi==0.116.1
uvicorn==0.35.0
starlette==0.47.2

# 데이터베이스
sqlmodel==0.0.24
psycopg2-binary==2.9.10

# 인증 및 보안
PyJWT==2.10.1
python-dotenv==1.1.0

# HTTP 클라이언트
httpx==0.28.1
requests==2.32.4

# 기타 필요한 패키지들...
```

---

## 2. 로컬 개발 환경 설정

### 🛠️ 필수 도구 설치

#### 1) Python 설치
```bash
# Windows
# https://www.python.org/downloads/ 에서 다운로드

# macOS
brew install python

# Ubuntu/Debian
sudo apt update
sudo apt install python3 python3-pip
```

#### 2) Git 설치
```bash
# Windows
# https://git-scm.com/download/win 에서 다운로드

# macOS
brew install git

# Ubuntu/Debian
sudo apt install git
```

#### 3) VS Code 설치 (권장)
- https://code.visualstudio.com/ 에서 다운로드
- Python, Docker, Git 확장 설치

### 🔧 로컬 개발 환경 구축

#### 1) 가상환경 생성
```bash
# 프로젝트 디렉토리로 이동
cd your-project

# 가상환경 생성
python -m venv venv

# 가상환경 활성화
# Windows
venv\Scripts\activate

# macOS/Linux
source venv/bin/activate
```

#### 2) 의존성 설치
```bash
# pip 업그레이드
pip install --upgrade pip

# 의존성 설치
pip install -r requirements.txt
```

#### 3) 환경 변수 설정
```bash
# .env 파일 생성
cp .env.example .env

# .env 파일 편집
# 데이터베이스 연결 정보, API 키 등 설정
```

#### 4) 로컬 서버 실행
```bash
# 개발 모드로 실행
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# 또는
python -m uvicorn main:app --reload
```

---

## 3. Docker 컨테이너화

### 🐳 Docker 기본 개념
- **컨테이너**: 애플리케이션과 그 실행 환경을 패키징한 것
- **이미지**: 컨테이너를 만들기 위한 템플릿
- **Dockerfile**: 이미지를 만들기 위한 명령어 모음

### 📝 Dockerfile 작성

#### 기본 Dockerfile
```dockerfile
# 1단계: 기본 이미지 선택
FROM python:3.11-slim

# 2단계: 시스템 패키지 설치
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 3단계: 작업 디렉토리 설정
WORKDIR /app

# 4단계: Python 환경 설정
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1

# 5단계: pip 업그레이드
RUN pip install --upgrade pip

# 6단계: 의존성 파일 복사 및 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 7단계: 소스 코드 복사
COPY . .

# 8단계: 포트 노출
EXPOSE 8000

# 9단계: 애플리케이션 실행
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### 멀티 서비스 Dockerfile (Nginx + FastAPI)
```dockerfile
# 1단계: 기본 이미지 선택
FROM python:3.11-slim

# 2단계: 시스템 패키지 설치
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    nginx \
    supervisor \
    gcc \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 3단계: 작업 디렉토리 설정
WORKDIR /app

# 4단계: Python 환경 설정
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1

# 5단계: pip 업그레이드
RUN pip install --upgrade pip

# 6단계: 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 7단계: 소스 코드 복사
COPY . .

# 8단계: 설정 파일 복사
COPY nginx.conf /etc/nginx/nginx.conf
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 9단계: 디렉토리 생성
RUN mkdir -p /var/log/supervisor /app/uploads

# 10단계: 포트 노출
EXPOSE 8080

# 11단계: 애플리케이션 실행
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### 🔧 Docker Compose (로컬 개발용)

#### `docker-compose.yml`
```yaml
version: '3.8'

services:
  # FastAPI 애플리케이션
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/dbname
    depends_on:
      - db
    volumes:
      - ./uploads:/app/uploads
    restart: unless-stopped

  # PostgreSQL 데이터베이스
  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=dbname
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  # Redis (캐싱용)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    restart: unless-stopped

volumes:
  postgres_data:
```

### 🚀 Docker 명령어

#### 로컬에서 Docker 실행
```bash
# 이미지 빌드
docker build -t your-app:latest .

# 컨테이너 실행
docker run -p 8000:8000 your-app:latest

# Docker Compose로 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f app

# 컨테이너 중지
docker-compose down
```

---

## 4. Google Cloud Platform (GCP) 배포

### ☁️ GCP 기본 개념
- **Cloud Run**: 서버리스 컨테이너 실행 환경
- **Cloud Build**: Docker 이미지 빌드 및 배포 자동화
- **Container Registry**: Docker 이미지 저장소

### 🔧 GCP 설정

#### 1) Google Cloud CLI 설치
```bash
# Windows
# https://cloud.google.com/sdk/docs/install 에서 다운로드

# macOS
brew install google-cloud-sdk

# Ubuntu/Debian
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
```

#### 2) GCP 프로젝트 설정
```bash
# 로그인
gcloud auth login

# 프로젝트 목록 확인
gcloud projects list

# 프로젝트 설정
gcloud config set project YOUR_PROJECT_ID

# 필요한 API 활성화
gcloud services enable cloudbuild.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable containerregistry.googleapis.com
```

### 📝 Cloud Build 설정

#### `cloudbuild.yaml`
```yaml
steps:
# 1단계: Docker 이미지 빌드
- name: 'gcr.io/cloud-builders/docker'
  args: [
    'build',
    '-t', 'gcr.io/$PROJECT_ID/your-app:latest',
    '.'
  ]
  timeout: '1800s'  # 30분

# 2단계: 이미지 푸시
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/your-app:latest']

# 3단계: Cloud Run 배포
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  entrypoint: gcloud
  args: [
    'run', 'deploy', 'your-app',
    '--image', 'gcr.io/$PROJECT_ID/your-app:latest',
    '--region', 'asia-northeast3',
    '--platform', 'managed',
    '--allow-unauthenticated',
    '--port', '8080',
    '--memory', '2Gi',
    '--cpu', '2',
    '--timeout', '300',
    '--set-env-vars', 'DATABASE_URL=$_DATABASE_URL'
  ]

images:
- 'gcr.io/$PROJECT_ID/your-app:latest'

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'
```

### 🚀 배포 명령어

#### 수동 배포
```bash
# Cloud Build로 빌드 및 배포
gcloud builds submit --config cloudbuild.yaml .

# 또는 단계별로 실행
# 1. 이미지 빌드
docker build -t gcr.io/YOUR_PROJECT_ID/your-app:latest .

# 2. 이미지 푸시
docker push gcr.io/YOUR_PROJECT_ID/your-app:latest

# 3. Cloud Run 배포
gcloud run deploy your-app \
  --image gcr.io/YOUR_PROJECT_ID/your-app:latest \
  --region asia-northeast3 \
  --platform managed \
  --allow-unauthenticated
```

### 🔐 환경 변수 설정

#### Cloud Run에서 환경 변수 설정
```bash
gcloud run services update your-app \
  --region asia-northeast3 \
  --set-env-vars DATABASE_URL="your-db-url",API_KEY="your-api-key"
```

#### Secret Manager 사용 (권장)
```bash
# 시크릿 생성
echo -n "your-secret-value" | gcloud secrets create my-secret --data-file=-

# Cloud Run에서 시크릿 사용
gcloud run services update your-app \
  --region asia-northeast3 \
  --set-secrets DATABASE_URL=my-secret:latest
```

---

## 5. AWS 배포

### ☁️ AWS 기본 개념
- **ECS (Elastic Container Service)**: 컨테이너 오케스트레이션
- **ECR (Elastic Container Registry)**: Docker 이미지 저장소
- **CodeBuild**: CI/CD 빌드 서비스

### 🔧 AWS 설정

#### 1) AWS CLI 설치
```bash
# Windows
# https://aws.amazon.com/cli/ 에서 다운로드

# macOS
brew install awscli

# Ubuntu/Debian
sudo apt install awscli
```

#### 2) AWS 계정 설정
```bash
# AWS 계정 설정
aws configure

# AWS Access Key ID 입력
# AWS Secret Access Key 입력
# Default region name 입력 (예: ap-northeast-2)
# Default output format 입력 (json)
```

### 📝 AWS 배포 설정

#### `buildspec.yml` (CodeBuild용)
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"your-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

#### `task-definition.json` (ECS용)
```json
{
  "family": "your-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "your-app",
      "image": "YOUR_ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/your-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DATABASE_URL",
          "value": "your-database-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/your-app",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 🚀 AWS 배포 명령어

#### ECR에 이미지 푸시
```bash
# ECR 리포지토리 생성
aws ecr create-repository --repository-name your-app

# 로그인
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 빌드 및 푸시
docker build -t your-app .
docker tag your-app:latest YOUR_ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/your-app:latest
docker push YOUR_ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/your-app:latest
```

#### ECS 서비스 배포
```bash
# 태스크 정의 등록
aws ecs register-task-definition --cli-input-json file://task-definition.json

# 서비스 생성
aws ecs create-service \
  --cluster your-cluster \
  --service-name your-app \
  --task-definition your-app:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-12345],securityGroups=[sg-12345],assignPublicIp=ENABLED}"
```

---

## 6. Azure 배포

### ☁️ Azure 기본 개념
- **Azure Container Instances (ACI)**: 서버리스 컨테이너
- **Azure Container Registry (ACR)**: Docker 이미지 저장소
- **Azure DevOps**: CI/CD 파이프라인

### 🔧 Azure 설정

#### 1) Azure CLI 설치
```bash
# Windows
# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows

# macOS
brew install azure-cli

# Ubuntu/Debian
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

#### 2) Azure 계정 설정
```bash
# 로그인
az login

# 구독 설정
az account set --subscription YOUR_SUBSCRIPTION_ID
```

### 📝 Azure 배포 설정

#### `azure-pipelines.yml`
```yaml
trigger:
- main

variables:
  dockerfilePath: '**/Dockerfile'
  imageRepository: 'your-app'
  containerRegistry: 'your-acr.azurecr.io'
  dockerfilePath: '**/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  displayName: Build and push stage
  jobs:
  - job: Build
    displayName: Build job
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: 'buildAndPush'
        repository: '$(imageRepository)'
        dockerfile: '$(dockerfilePath)'
        containerRegistry: '$(containerRegistry)'
        tags: |
          $(tag)
          latest

- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build
  jobs:
  - deployment: Deploy
    displayName: Deploy job
    pool:
      vmImage: 'ubuntu-latest'
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            displayName: Deploy to Azure Container Instances
            inputs:
              azureSubscription: 'Your-Azure-Subscription'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az container create \
                  --resource-group your-rg \
                  --name your-app \
                  --image $(containerRegistry)/$(imageRepository):$(tag) \
                  --dns-name-label your-app \
                  --ports 8080
```

### 🚀 Azure 배포 명령어

#### ACR에 이미지 푸시
```bash
# ACR 로그인
az acr login --name your-acr

# 이미지 빌드 및 푸시
docker build -t your-acr.azurecr.io/your-app:latest .
docker push your-acr.azurecr.io/your-app:latest
```

#### ACI에 배포
```bash
# 컨테이너 인스턴스 생성
az container create \
  --resource-group your-rg \
  --name your-app \
  --image your-acr.azurecr.io/your-app:latest \
  --dns-name-label your-app \
  --ports 8080 \
  --environment-variables DATABASE_URL="your-db-url"
```

---

## 7. CI/CD 파이프라인 구축

### 🔄 CI/CD 기본 개념
- **CI (Continuous Integration)**: 코드 변경사항을 자동으로 빌드하고 테스트
- **CD (Continuous Deployment)**: 테스트 통과한 코드를 자동으로 배포

### 📝 GitHub Actions 설정

#### `.github/workflows/deploy.yml`
```yaml
name: Deploy to Cloud

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    
    - name: Run tests
      run: |
        pytest

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    # GCP 배포
    - name: Deploy to GCP
      if: matrix.cloud == 'gcp'
      uses: google-github-actions/deploy-cloudrun@v1
      with:
        service: your-app
        image: gcr.io/${{ secrets.GCP_PROJECT_ID }}/your-app:${{ github.sha }}
        region: asia-northeast3
        credentials: ${{ secrets.GCP_SA_KEY }}
    
    # AWS 배포
    - name: Deploy to AWS
      if: matrix.cloud == 'aws'
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: task-definition.json
        service: your-app
        cluster: your-cluster
        wait-for-service-stability: true
```

### 🔐 시크릿 설정

#### GitHub Secrets 설정
1. GitHub 저장소 → Settings → Secrets and variables → Actions
2. 다음 시크릿 추가:
   - `GCP_PROJECT_ID`: GCP 프로젝트 ID
   - `GCP_SA_KEY`: GCP 서비스 계정 키 (JSON)
   - `AWS_ACCESS_KEY_ID`: AWS 액세스 키
   - `AWS_SECRET_ACCESS_KEY`: AWS 시크릿 키

---

## 8. 모니터링 및 로깅

### 📊 모니터링 도구

#### 1) GCP Cloud Monitoring
```python
# main.py에 헬스체크 엔드포인트 추가
@app.get("/health")
def health_check():
    return {
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "version": "1.0.0"
    }
```

#### 2) 로깅 설정
```python
import logging
from datetime import datetime

# 로깅 설정
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = datetime.now()
    response = await call_next(request)
    duration = datetime.now() - start_time
    
    logger.info(
        f"{request.method} {request.url.path} - "
        f"Status: {response.status_code} - "
        f"Duration: {duration.total_seconds():.3f}s"
    )
    
    return response
```

### 🔍 로그 확인 명령어

#### GCP Cloud Run 로그
```bash
# 실시간 로그 확인
gcloud logging tail "resource.type=cloud_run_revision AND resource.labels.service_name=your-app"

# 특정 시간대 로그 확인
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=your-app" --limit=50
```

#### AWS ECS 로그
```bash
# CloudWatch 로그 확인
aws logs tail /ecs/your-app --follow
```

---

## 9. 문제 해결 가이드

### 🚨 일반적인 문제들

#### 1) Docker 빌드 실패
```bash
# 캐시 없이 빌드
docker build --no-cache -t your-app .

# 상세한 빌드 로그
docker build --progress=plain -t your-app .
```

#### 2) 포트 충돌
```bash
# 사용 중인 포트 확인
netstat -tulpn | grep :8000

# 다른 포트 사용
uvicorn main:app --host 0.0.0.0 --port 8001
```

#### 3) 메모리 부족
```dockerfile
# Dockerfile에 메모리 제한 설정
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
```

#### 4) 환경 변수 문제
```bash
# 환경 변수 확인
echo $DATABASE_URL

# .env 파일 로드 확인
python -c "import os; from dotenv import load_dotenv; load_dotenv(); print(os.getenv('DATABASE_URL'))"
```

### 🔧 디버깅 도구

#### 1) 로컬 디버깅
```python
# main.py에 디버그 엔드포인트 추가
@app.get("/debug")
def debug_info():
    return {
        "environment": os.getenv("ENVIRONMENT", "development"),
        "database_url": os.getenv("DATABASE_URL", "not_set"),
        "timestamp": datetime.now().isoformat()
    }
```

#### 2) Docker 컨테이너 디버깅
```bash
# 실행 중인 컨테이너에 접속
docker exec -it container_name /bin/bash

# 컨테이너 로그 확인
docker logs container_name -f
```

## 10. 보안(Security) 강화
### 🔐 HTTPS 적용
- Cloud Run: 기본적으로 HTTPS 지원
- AWS ECS: ALB(Application Load Balancer) + ACM(AWS Certificate Manager)
- Azure: Front Door, Application Gateway

### 🔑 비밀 관리
- 환경 변수 대신 Secret Manager (GCP), AWS Secrets Manager, Azure Key Vault 사용 권장
```
# 예시: GCP Secret Manager에서 DATABASE_URL 불러오기
gcloud secrets versions access latest --secret=DATABASE_URL
```

### 👥 인증/인가
- JWT 외에도 OAuth2, OpenID Connect(OIDC) 고려
- RBAC(Role-Based Access Control) 로 권한 분리
---

## 11. 데이터베이스 운영 전략
### 🗂️ 마이그레이션
- Alembic 사용해 DB 스키마 버전 관리
```
alembic revision --autogenerate -m "add user table"
alembic upgrade head
```

### 💾 백업 및 복구
- Cloud SQL / RDS 자동 백업 활성화
- Point-in-Time Recovery(PITR) 로 특정 시점 복구 가능


### ⚡ 성능 최적화
- 인덱스 튜닝 (EXPLAIN 활용)
- Redis 캐싱 도입 → DB 부하 감소
- ORM(N+1 문제) 최적화

---

## 12. Observability (관찰성)
### 📊 메트릭 수집
- Prometheus + Grafana
- Cloud Monitoring, CloudWatch, Azure Monitor 도 활용 가능

### 🔍 분산 트레이싱
- OpenTelemetry 표준 사용 → Jaeger, Zipkin 연동 가능

### 🔔 알림
- 장애 감지 시 Slack, Discord, Email로 자동 알림 설정
```yaml
# Prometheus Alertmanager 예시
groups:
  - name: app-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status="500"}[5m]) > 5
        for: 2m
        labels:
          severity: critical
```

---

## 13. 테스트 전략
###🧪 단위 테스트
- pytest 로 함수 단위 검증
```
pytest -v
```

### 🔗 통합 테스트
- Docker Compose 로 DB/Redis 포함한 통합 테스트
- Testcontainers 라이브러리 활용 가능

###🚀 로드 테스트
- Locust 또는 k6 사용 → API 성능 검증

```
locust -f load_test.py --headless -u 100 -r 10 -t 5m
```

---

## 14. 고급 운영 전략
### 🔄 배포 전략
- Blue-Green 배포: 두 환경 운영 후 전환
- Canary 배포: 일부 트래픽만 새 버전에 전달

### ⚙️ Auto Scaling
- Cloud Run: 요청량 기반 자동 확장
- AWS ECS: CPU/Memory 기준 확장 정책 설정
- Kubernetes: HPA(Horizontal Pod Autoscaler)

### 💰 비용 최적화
- 서버리스 최대 활용 (Cloud Run, Lambda, ACI)
- Spot Instance / Preemptible VM 으로 저비용 운영

---

### 📞 지원 및 리소스

#### 유용한 링크
- [FastAPI 공식 문서](https://fastapi.tiangolo.com/)
- [Docker 공식 문서](https://docs.docker.com/)
- [GCP Cloud Run 문서](https://cloud.google.com/run/docs)
- [AWS ECS 문서](https://docs.aws.amazon.com/ecs/)
- [Azure Container Instances 문서](https://docs.microsoft.com/en-us/azure/container-instances/)

#### 커뮤니티
- [FastAPI GitHub](https://github.com/tiangolo/fastapi)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/fastapi)
- [Reddit r/FastAPI](https://www.reddit.com/r/FastAPI/)

---

## 🎯 마무리

이 가이드를 따라하면 FastAPI 애플리케이션을 다양한 클라우드 플랫폼에 성공적으로 배포할 수 있습니다. 
실무에서는 이 문서를 기반으로 DevOps 플레이북을 만들어두면 팀 전체가 효율적으로 협업할 수 있습니다.

### 📋 체크리스트
- [ ] 로컬 개발 환경 설정 완료
- [ ] Docker 컨테이너화 완료
- [ ] 클라우드 플랫폼 계정 설정 완료
- [ ] CI/CD 파이프라인 구축 완료
- [ ] 모니터링 및 로깅 설정 완료
- [ ] 프로덕션 환경 테스트 완료

### 🚀 다음 단계
1. **보안 강화**: HTTPS, 인증, 권한 관리
2. **성능 최적화**: 캐싱, CDN, 로드 밸런싱
3. **확장성**: 마이크로서비스 아키텍처 고려
4. **백업 및 재해 복구**: 데이터 백업 전략 수립

**Happy Coding! 🎉**

