# backend-v3 로컬 실행 방법

로컬 실행 자체는 달라진 부분이 없으므로, [backend-v2](https://github.com/ddcn4-1/backend-v2)의 실행 방법을 참고할 것

# [KubeSpray 팀] Backend-v3 GitHub Actions 기반 Backend CI: ECR Build & Push 자동화
- 버전 / 작성일: v1 / 2025.11.27
- 작성자: 윤효정
- 목차:
  1. Overview
  2. 아키텍처 & Flow
  3. Requirements
  4. 소스 코드 / 레포지토리 구조
  5. Setup & Installation
  6. Troubleshooting
  7. Appendix

## 1. Overview (개요)
### 목표
- 대상 브랜치 : feat/yhj
- GitHub Actions를 활용하여 Backend 애플리케이션(main, queue, admin)의 Docker 이미지를 자동 빌드
- Docker 이미지를 ECR에 자동으로 푸시하는 CI 파이프라인 구성
- AWS 인증을 GitHub OIDC 방식으로 변경 (github → AWS IAM Role assume 방식)
    - backend-v1 (Github Actions token)
    - backend-v2 (Github Actions token)
  
### 실사용 영상

추가 예정

## 2. 아키텍처 & Flow
### 파이프라인 흐름
1. 개발자가 feature 또는 dev 브랜치에 코드를 push
2. GitHub Actions가 워크플로우 실행
3. GitHub OIDC를 통해 AWS IAM Role assume
4. Docker 이미지 빌드
5. 태그는 Git commit SHA8 형태로 자동 생성
6. ECR 로그인 후 이미지 push
7. ArgoCD Image Updater가 새 이미지 태그를 감지하여 CD로 이어짐 (레포지토리 [kbsp-argocd-v3](https://github.com/ddcn4-1/kbsp-argocd-v3) 참조)

### 아키텍처 다이어그램

- GitHub Actions → AWS IAM Role(OIDC) → ECR 인증 구조
- Git commit → 이미지 빌드 → ECR push 플로우
- 전체 CICD 구조: GitHub Actions(CI) ↔ ECR ↔ ArgoCD(Image Updater)

## 3. Requirements (사전 준비 사항)

1. AWS IAM > Identity Providers에 GitHub OIDC 등록
  - Provider URL: https://token.actions.githubusercontent.com
  - Audience: sts.amazonaws.com
2. Backend 각 앱(admin, main, queue)에 Dockerfile 존재
  - 루트 경로 또는 서브폴더 경로는 GitHub Actions에서 빌드 시 명시 필요

## 4. 소스 코드 / 레포지토리 구조
### 레포지토리 구조
```
.github/workflows/ci-backend.yaml
```

### 주요 보완 사항
- sha_short 로 태그 제어
- 각 앱별로 Docker build 명령어
- main/admin/queue 모두 빌드 가능하도록 구조 보완
- permissions: contents → write는 코드 변경 시 필요하지만, 단순 CI에는 read로도 충분


## 5. Setup & Installation

### Step 1. OIDC 기반 IAM Role 생성

IAM Role 구성:
- Trusted entity type: Web identity
- Provider: token.actions.githubusercontent.com (GitHub OIDC)
- Audience: sts.amazonaws.com
- Conditions: GitHub Organization / Repository / Branch 제한 입력

IAM Role 권한: `AmazonEC2ContainerRegistryPowerUser`
- 또는 최소 권한 정책 직접 구성

### Step 2. GitHub Actions Workflow에 IAM Role 반영
- ci-backend.yaml에서 다음 값을 적용
- AWS Account ID는 공개되어도 문제없으나, 필요에 따라 비공개로 유지해도 된다.
```
role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<생성한 IAM 역할 이름>
```

## 6. Troubleshooting
### 1. GitHub Actions에서 AWS 인증 실패(OIDC Assume 실패)
확인사항:
- IAM Role의 trust policy에 GitHub OIDC Provider 등록 여부
- Repository / Branch Condition이 정확한지
- Audience가 sts.amazonaws.com인지

### 2. Docker 빌드 시 경로 오류
확인사항:
- Dockerfile이 존재하는 경로에서 build가 실행되는지
- 폴더 이름이 main/admin/queue와 정확히 일치하는지

### 3. ECR push 실패
확인사항:
- IAM Role 권한에 ECR push 권한이 포함되어 있는지
- REGISTRY 주소가 올바른지
- aws-actions/amazon-ecr-login@v2가 성공했는지

### 4. SHA 환경변수 미적용
- echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_ENV 사용 여부
- ${{ env.sha_short }} 구문이 정확한지

## 7. Appendix
### 참고 레퍼런스
- GitHub Actions 공식 문서
- AWS OIDC 공식 문서
- ECR push 정책 문서
- aws-actions 공식 GitHub 저장소

### 용어 설명
- OIDC: GitHub Actions가 AWS를 인증할 때 사용하는 Web Identity 방식
- ECR: AWS Elastic Container Registry
- SHA Short: Git commit ID의 앞 7~8자, Docker 이미지 태그로 흔히 사용
- role-to-assume: GitHub Actions가 AWS IAM Role을 대신 실행(AssumeRole)하는 설정
- PowerUser Policy: ECR에서 push, pull을 모두 가능하게 하는 광범위한 권한
