# ⚙️ Team4 Config Repository (GitOps)

Team4 Diary Application의 Kubernetes 매니페스트 및 ArgoCD 설정을 관리하는 GitOps 레포지토리입니다.

---

## 🔗 관련 레포지토리

| 레포지토리 | 설명 |
|:---|:---|
| [team4_app](https://github.com/CLD-05/team4_app) | Spring Boot 애플리케이션 소스 코드 |
| [team4_terraform](https://github.com/CLD-05/team4_terraform) | AWS 인프라 프로비저닝 (EKS, RDS, VPC 등) |
| [team4_config](https://github.com/CLD-05/team4_config) | Kubernetes 매니페스트 및 ArgoCD 설정 (현재 레포) |

---

## 🚀 1. 프로젝트 개요 (Overview)

- **목적**: Kubernetes 클러스터에 배포되는 애플리케이션의 모든 설정을 코드로 관리 (GitOps)
- **CD 도구**: ArgoCD
- **구성 관리**: Kustomize (base + overlays)
- **환경**: dev / prod 분리 운영
- **클러스터**: AWS EKS (`team4-cluster`, ap-northeast-2)

---

## 🏗️ 2. 시스템 아키텍처 (Architecture)

```
GitHub Actions (CI/CD)
        │
        ├── ECR에 Docker 이미지 푸시
        └── team4_config 이미지 태그 자동 업데이트
                │
                ▼
        ArgoCD (GitOps)
                │
                ├── diary-app-dev (네임스페이스)
                └── diary-app-prod (네임스페이스)
                        │
                        ▼
                AWS EKS 클러스터
                        │
                        ├── ALB (AWS Load Balancer Controller)
                        ├── RDS (MySQL 8.0)
                        └── S3 + CloudFront (이미지 스토리지)
```

---

## 📂 3. 디렉토리 구조 (Project Structure)

```
team4_config/
├── apps/
│   └── diary-app/
│       ├── base/                   # 공통 매니페스트
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── ingress.yaml
│       │   ├── configmap.yaml
│       │   ├── serviceaccount.yaml
│       │   └── kustomization.yaml
│       └── overlays/               # 환경별 설정
│           ├── dev/
│           │   ├── kustomization.yaml
│           │   └── patch-deployment.yaml
│           └── prod/
│               ├── kustomization.yaml
│               └── patch-deployment.yaml
└── argocd/
    ├── project.yaml
    ├── application-dev.yaml
    └── application-prod.yaml
```

---

## 🛠️ 4. 기술 스택 (Tech Stack)

| 구분 | 기술 |
|:---|:---|
| 컨테이너 오케스트레이션 | AWS EKS (Kubernetes) |
| GitOps CD | ArgoCD |
| 구성 관리 | Kustomize |
| 로드밸런서 | AWS Load Balancer Controller (ALB) |
| 컨테이너 이미지 저장소 | Amazon ECR |
| CI/CD | GitHub Actions |

---

## ⚙️ 5. 환경별 설정 (Environment Configuration)

| 항목 | Dev | Prod |
|:---|:---|:---|
| 네임스페이스 | `diary-app-dev` | `diary-app-prod` |
| Replicas | 1 | 2 |
| CPU requests/limits | 250m / 500m | 250m / 500m |
| Memory requests/limits | 256Mi / 512Mi | 256Mi / 512Mi |
| ALB 이름 | `team4-diaryapp-dev-alb` | `team4-diaryapp-prod-alb` |

---

## 🚢 6. EKS 재생성 후 배포 순서 (Deployment Guide)

### 1) kubeconfig 업데이트
```bash
aws eks update-kubeconfig --name team4-cluster --region ap-northeast-2
```

### 2) ArgoCD 설치
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 설치 확인
kubectl get pods -n argocd -w
```

### 3) ArgoCD Application 등록
```bash
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/application-dev.yaml
kubectl apply -f argocd/application-prod.yaml

# 확인
kubectl get applications -n argocd
```

### 4) Secret 수동 생성
> ⚠️ secret.yaml은 보안상 gitignore 등록되어 있어 클러스터에 직접 생성해야 합니다.

```bash
# prod
kubectl create secret generic diary-app-secret \
  --from-literal=DB_HOST=<rds-endpoint> \
  --from-literal=DB_PASSWORD=<db-password> \
  '--from-literal=DB_URL=jdbc:mysql://<rds-endpoint>:3306/diarydb?serverTimezone=Asia/Seoul' \
  -n diary-app-prod

# dev
kubectl create secret generic diary-app-secret \
  --from-literal=DB_HOST=<rds-endpoint> \
  --from-literal=DB_PASSWORD=<db-password> \
  '--from-literal=DB_URL=jdbc:mysql://<rds-endpoint>:3306/diarydb?serverTimezone=Asia/Seoul' \
  -n diary-app-dev
```

### 5) AWS Load Balancer Controller 설치
```bash
# Service Account 생성
kubectl create serviceaccount aws-load-balancer-controller -n kube-system
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::194722398200:role/team4-alb-controller-role

# Helm으로 설치
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=team4-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=<vpc-id> \
  --set region=ap-northeast-2

# 설치 확인
kubectl get pods -n kube-system | grep aws-load-balancer
```

### 6) 배포 확인
```bash
# pod 상태 확인
kubectl get pods -A

# ALB 주소 확인
kubectl get ingress -A
```

---

## 🔒 7. ArgoCD UI 접속 방법

```bash
# 포트포워딩
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 초기 비밀번호 확인
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

브라우저에서 `https://localhost:8080` 접속
- ID: `admin`
- Password: 위 명령어 결과값

---

## 🔄 8. CI/CD 흐름 (Pipeline Flow)

```
개발자 코드 push (main 브랜치)
        │
        ▼
GitHub Actions CD 실행
        │
        ├── JAR 빌드
        ├── Docker 이미지 빌드
        ├── ECR 푸시 (${{ github.sha }} 태그)
        └── team4_config 이미지 태그 자동 업데이트
                │
                ▼
        ArgoCD 변경 감지
                │
                ▼
        Rolling Update 배포 (서비스 중단 없음)
```

---

## 💡 9. 유용한 명령어 (Useful Commands)

### Pod 관련
```bash
# 전체 pod 상태 확인
kubectl get pods -A

# prod pod 상태 실시간 확인
kubectl get pods -n diary-app-prod -w

# pod 로그 확인
kubectl logs -f <pod-name> -n diary-app-prod

# pod 내부 접속
kubectl exec -it <pod-name> -n diary-app-prod -- /bin/sh

# pod 내부에서 health 체크
kubectl exec -it <pod-name> -n diary-app-prod -- curl http://localhost:8080/actuator/health

# pod 재시작
kubectl rollout restart deployment/diary-app -n diary-app-prod
```

### Deployment 관련
```bash
# 이미지 태그 확인
kubectl get pods -n diary-app-prod -o jsonpath='{.items[*].spec.containers[0].image}'

# 롤백
kubectl rollout undo deployment/diary-app -n diary-app-prod

# 배포 히스토리 확인
kubectl rollout history deployment/diary-app -n diary-app-prod
```

### ArgoCD 관련
```bash
# Application 상태 확인
kubectl get applications -n argocd

# 강제 Sync
kubectl patch application diary-app-prod -n argocd --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {"revision": "HEAD"}}}'

# Hard Refresh
kubectl patch application diary-app-prod -n argocd --type merge \
  -p '{"metadata": {"annotations": {"argocd.argoproj.io/refresh": "hard"}}}'

# ArgoCD UI 포트포워딩
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Secret / ConfigMap 관련
```bash
# Secret 확인
kubectl get secret -n diary-app-prod

# Secret 삭제 후 재생성
kubectl delete secret diary-app-secret -n diary-app-prod

# ConfigMap 확인
kubectl get configmap -n diary-app-prod
```

### Ingress / Service 관련
```bash
# ALB 주소 확인
kubectl get ingress -A

# Service 확인
kubectl get svc -n diary-app-prod

# Ingress 상세 확인
kubectl describe ingress diary-app-ingress -n diary-app-prod
```

### 디버깅
```bash
# pod 이벤트 확인
kubectl describe pod <pod-name> -n diary-app-prod | grep -A20 "Events"

# probe 상태 확인
kubectl describe pod <pod-name> -n diary-app-prod | grep -A5 "Readiness\|Liveness"

# ALB Controller 로그
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```
