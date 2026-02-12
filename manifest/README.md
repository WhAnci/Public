# Kubernetes Manifest 배포 가이드

## 필수 컨트롤러 설치

### 1. External Secrets Operator (ESO) 설치

External Secrets Operator를 설치하여 AWS Secrets Manager와 연동합니다.

```bash
# Helm 저장소 추가
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# External Secrets Operator 설치
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace \
  --set installCRDs=true

# 설치 확인
kubectl get pods -n external-secrets-system
```

### 2. AWS Load Balancer Controller 설치

AWS Load Balancer Controller를 설치하여 ELB/NLB를 자동 생성합니다.

#### 2.1 IAM Policy 생성

```bash
# IAM Policy 다운로드
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# IAM Policy 생성
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

#### 2.2 IRSA (IAM Role for Service Account) 생성

```bash
# EKS 클러스터 이름과 AWS 계정 ID 설정
export CLUSTER_NAME=<your-cluster-name>
export AWS_ACCOUNT_ID=586639730662
export AWS_REGION=ap-northeast-2

# IRSA 생성
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region=$AWS_REGION \
  --approve
```

#### 2.3 Helm으로 Controller 설치

```bash
# Helm 저장소 추가
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# AWS Load Balancer Controller 설치
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION \
  --set vpcId=<your-vpc-id>

# 설치 확인
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## 애플리케이션 배포

### 3. External Secrets용 IAM Role 생성

```bash
# External Secrets용 IRSA 생성
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=whanci \
  --name=external-secrets-sa \
  --attach-policy-arn=arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --override-existing-serviceaccounts \
  --region=$AWS_REGION \
  --approve
```

### 4. 매니페스트 배포 순서

```bash
# 1. Namespace 및 ServiceAccount 생성
kubectl apply -f namespace.yaml

# 2. SecretStore 생성
kubectl apply -f secretstore.yaml

# 3. ExternalSecret 생성 (AWS Secrets Manager와 동기화)
kubectl apply -f externalsecret.yaml

# 4. Secret 생성 확인
kubectl get secret wh-skills-secret -n whanci
kubectl describe externalsecret wh-skills-external-secret -n whanci

# 5. 애플리케이션 배포
kubectl apply -f customer-deployment.yaml
kubectl apply -f product-deployment.yaml

# 6. Load Balancer 생성
kubectl apply -f cus-lb.yaml
kubectl apply -f pro-lb.yaml

# 배포 확인
kubectl get all -n whanci
kubectl get svc -n whanci
```

## 트러블슈팅

### ExternalSecret 상태 확인
```bash
kubectl describe externalsecret wh-skills-external-secret -n whanci
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets
```

### Load Balancer 확인
```bash
kubectl get svc -n whanci
kubectl describe svc customer-svc -n whanci
kubectl describe svc product-svc -n whanci
```

### Pod 로그 확인
```bash
kubectl logs -n whanci -l app=customer
kubectl logs -n whanci -l app=product
```

## AWS Secrets Manager 설정

AWS Secrets Manager에 `wh/skills/secret` 시크릿이 다음 형식으로 저장되어 있어야 합니다:

```json
{
  "DB_HOST": "your-db-host",
  "DB_USER": "your-db-user",
  "DB_PASSWORD": "your-db-password",
  "DB_NAME": "your-db-name"
}
```

시크릿 생성 예시:
```bash
aws secretsmanager create-secret \
  --name wh/skills/secret \
  --secret-string '{"DB_HOST":"db.example.com","DB_USER":"admin","DB_PASSWORD":"password123","DB_NAME":"skillsdb"}' \
  --region ap-northeast-2
```
