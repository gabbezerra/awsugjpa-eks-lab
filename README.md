# Workshop: EKS com Auto Mode e ALB Ingress Controller

## Visão Geral

Este workshop guia você na criação de um cluster EKS com Auto Mode habilitado e exposição de aplicações via AWS Load Balancer Controller (ALB Ingress).

https://aws.amazon.com/pt/blogs/containers/getting-started-with-amazon-eks-auto-mode/

## Pré-requisitos

### Ferramentas Necessárias

```bash
# AWS CLI v2
aws --version

# eksctl (v0.170.0+)
eksctl version

# kubectl
kubectl version --client

# Helm 3
helm version
```

### Instalação das Ferramentas (macOS)

```bash
# AWS CLI
brew install awscli

# eksctl
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# kubectl
brew install kubectl

# Helm
brew install helm
```

### Configuração AWS

```bash
# Configurar credenciais
aws configure

# Verificar identidade
aws sts get-caller-identity
```

---

## Parte 1: Criação do Cluster EKS com Auto Mode

### 1.1 Criar o arquivo de configuração do cluster

Crie o arquivo `cluster-config.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: workshop-eks-alb
  region: us-east-1
  version: "1.35"

# Habilita Auto Mode - gerenciamento automático de nodes
autoModeConfig:
  enabled: true

# Acesso público ao cluster
accessConfig:
  authenticationMode: API_AND_CONFIG_MAP

# VPC com subnets públicas e privadas
vpc:
  cidr: 10.42.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

# IAM OIDC Provider para service accounts
iam:
  withOIDC: true

# Addons gerenciados
addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: eks-pod-identity-agent
    version: latest

# CloudWatch Logging
cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler
```

### 1.2 Criar o cluster

```bash
# Criar cluster (leva ~15-20 minutos)
eksctl create cluster -f cluster-config.yaml

aws eks --region us-east-1 update-kubeconfig --name workshop-eks-automode
# Verificar contexto
kubectl config current-context

# Verificar nodes (Auto Mode provisiona conforme demanda)
kubectl get nodes
```

### 1.3 Verificar Auto Mode

```bash
# Verificar NodePools criados automaticamente
kubectl get nodepools

# Verificar NodeClasses
kubectl get ec2nodeclasses
```

---

## Parte 2: Instalação do AWS Load Balancer Controller

### 2.1 Criar IAM Policy para o Controller

```bash
# Baixar a policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json

# Criar a policy na AWS
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

### 2.2 Criar Service Account com IAM Role

#### 2.2.1 Se nunca tiver criado um oidc provider

oidc_id=$(aws eks describe-cluster --name workshop-eks-automode --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id

# check oidc
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

# if not create
eksctl utils associate-iam-oidc-provider --cluster workshop-eks-automode  --approve
```bash
# Obter Account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Criar IAM Service Account
eksctl create iamserviceaccount \
  --cluster=workshop-eks-automode \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS-ACCOUNT}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

### 2.3 Instalar o Controller via Helm

```bash
# Adicionar repo Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Instalar o AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=workshop-eks-automode \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-xxx-xxxx-xxxx
```

### 2.4 Verificar instalação

```bash
# Verificar deployment
kubectl get deployment -n kube-system aws-load-balancer-controller

# Verificar pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Verificar logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

### 2.5 Taggear Subnets (Importante!)

O ALB Controller precisa identificar as subnets. Verifique se as tags estão corretas:

```bash
# Obter VPC ID
VPC_ID=$(aws eks describe-cluster --name workshop-eks-automode --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Listar subnets da VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query "Subnets[*].[SubnetId,Tags[?Key=='Name'].Value|[0],MapPublicIpOnLaunch]" --output table

# Para subnets PÚBLICAS (internet-facing ALB), adicionar tag:
# kubernetes.io/role/elb = 1

# Para subnets PRIVADAS (internal ALB), adicionar tag:
# kubernetes.io/role/internal-elb = 1
```

---

## Parte 3: Aplicação de Teste

### 3.1 Criar namespace para a aplicação

```bash
kubectl create namespace workshop-app
```

### 3.2 Deploy da aplicação de teste

Crie o arquivo `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-api
  namespace: workshop-app
  labels:
    app: workshop-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-api
  template:
    metadata:
      labels:
        app: workshop-api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args:
          - "-text=Workshop EKS ALB - Node: $(NODE_NAME)"
          - "-listen=:8080"
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: workshop-api
  namespace: workshop-app
spec:
  selector:
    app: workshop-api
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

```bash
kubectl apply -f app-deployment.yaml
```

### 3.3 Configurar Ingress com ALB

Crie o arquivo `ingress-alb.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: workshop-ingress
  namespace: workshop-app
  annotations:
    # Tipo de Load Balancer
    alb.ingress.kubernetes.io/scheme: internet-facing
    
    # Tipo de target (ip = direto no pod, instance = via NodePort)
    alb.ingress.kubernetes.io/target-type: ip
    
    # Health check
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
    
    # Grupo do ALB (permite compartilhar ALB entre múltiplos Ingress)
    alb.ingress.kubernetes.io/group.name: workshop-alb
    
    # Tags para o ALB
    alb.ingress.kubernetes.io/tags: Environment=workshop,Project=eks-lab
    
    # Listener ports
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: workshop-api
            port:
              number: 80
```

```bash
kubectl apply -f ingress-alb.yaml

# Verificar Ingress
kubectl get ingress -n workshop-app

# Aguardar ALB ser provisionado (pode levar 2-3 minutos)
kubectl get ingress workshop-ingress -n workshop-app -w
```

### 3.4 Obter URL do ALB

```bash
# Obter DNS do ALB
ALB_URL=$(kubectl get ingress workshop-ingress -n workshop-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB URL: http://${ALB_URL}"

# Testar (aguarde DNS propagar ~1-2 min)
curl -v http://${ALB_URL}
```

---

### 3.5 Monitorar Auto Mode

```bash
# Monitorar nodes sendo criados
watch kubectl get nodes

# Verificar pods
kubectl get pods -n workshop-app -w

# Verificar eventos
kubectl get events -n workshop-app --sort-by='.lastTimestamp'
```

---

## Parte 4: Limpeza

### 4.1 Remover recursos

```bash
# Remover aplicações (isso também remove o ALB)
kubectl delete namespace workshop-app

# Aguardar ALB ser deletado
aws elbv2 describe-load-balancers --query "LoadBalancers[?contains(LoadBalancerName, 'workshop')]" --output table

# Remover AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system

# Remover Service Account
eksctl delete iamserviceaccount \
  --cluster=workshop-eks-alb \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --region us-east-1
```

### 5 Deletar o cluster

```bash
# Deletar cluster completo
eksctl delete cluster --name workshop-eks-alb --region us-east-1

# Remover IAM Policy (opcional)
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam delete-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy
```

---

## Troubleshooting

### Problemas Comuns

**ALB não está sendo criado:**
```bash
# Verificar logs do controller
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Verificar eventos do Ingress
kubectl describe ingress workshop-ingress -n workshop-app

# Verificar se as subnets têm as tags corretas
aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" --query "Subnets[*].[SubnetId,Tags]" --output json
```

**Target Group com targets unhealthy:**
```bash
# Verificar se os pods estão rodando
kubectl get pods -n workshop-app

# Verificar se o service está correto
kubectl get endpoints workshop-api -n workshop-app

# Verificar security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=${VPC_ID}" --query "SecurityGroups[*].[GroupId,GroupName]" --output table
```

**Erro de permissão IAM:**
```bash
# Verificar se o service account foi criado
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml

# Verificar role ARN
kubectl describe sa aws-load-balancer-controller -n kube-system | grep -i arn
```

---

## Annotations Úteis do ALB

| Annotation | Descrição | Exemplo |
|------------|-----------|---------|
| `alb.ingress.kubernetes.io/scheme` | internet-facing ou internal | `internet-facing` |
| `alb.ingress.kubernetes.io/target-type` | ip ou instance | `ip` |
| `alb.ingress.kubernetes.io/certificate-arn` | ARN do certificado ACM | `arn:aws:acm:...` |
| `alb.ingress.kubernetes.io/ssl-redirect` | Redirect HTTP→HTTPS | `443` |
| `alb.ingress.kubernetes.io/group.name` | Compartilhar ALB | `my-group` |
| `alb.ingress.kubernetes.io/load-balancer-name` | Nome customizado | `my-alb` |
| `alb.ingress.kubernetes.io/subnets` | Subnets específicas | `subnet-xxx,subnet-yyy` |
| `alb.ingress.kubernetes.io/security-groups` | Security groups | `sg-xxx,sg-yyy` |
| `alb.ingress.kubernetes.io/wafv2-acl-arn` | WAF Web ACL | `arn:aws:wafv2:...` |
| `alb.ingress.kubernetes.io/actions.*` | Actions customizadas | Ver docs |

---

## Referências

- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [EKS Auto Mode Documentation](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [ALB Ingress Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)
- [eksctl Documentation](https://eksctl.io/)
