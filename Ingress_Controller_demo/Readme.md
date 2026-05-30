# Kubernetes Ingress & Deploying Tic-Tac-Toe and RockPaperScissor Games on EKS with AWS Load Balancer Controller

---

## Table of Contents

- [Why is Ingress Widely Used?](#why-is-ingress-widely-used)
- [Components of Ingress in Kubernetes](#components-of-ingress-in-kubernetes)
- [Ingress Controller Modes in AWS EKS](#ingress-controller-modes-in-aws-eks)
- [Instance Mode vs. IP Mode](#instance-mode-vs-ip-mode)
- [Prerequisites](#prerequisites)
- [Step 1: Connect kubectl to Your EKS Cluster](#step-1-connect-kubectl-to-your-eks-cluster)
- [Step 2: Create an IAM OIDC Provider](#step-2-create-an-iam-oidc-provider-for-your-cluster)
- [Step 3: Install AWS Load Balancer Controller with Helm](#step-3-install-aws-load-balancer-controller-with-helm)
- [Step 4: Deploy Games Application](#step-4-deploy-games-application)
- [Step 5: Verify the Deployment](#step-5-verify-the-deployment)
- [Optional: Route53 + ACM Certificate Setup](#optional-route53--acm-certificate-setup)

---

## Why is Ingress Widely Used?

Ingress is widely used in Kubernetes because it provides a **centralized way to manage external access to services**. Instead of creating multiple LoadBalancers or NodePort services, Ingress offers a scalable and efficient way to expose applications.

### Key Advantages of Ingress

| Advantage | Description |
|---|---|
| **Single Entry Point** | Manages access to multiple services from a single external URL |
| **Path-Based Routing** | Hosts different applications under different paths (`/app1`, `/app2`) |
| **Host-Based Routing** | Supports different domains/subdomains (`app1.example.com`, `app2.example.com`) |
| **TLS/SSL Termination** | Secures applications with HTTPS, offloading SSL/TLS termination at the Ingress |
| **Load Balancing** | Distributes incoming traffic across multiple backend pods |
| **Reduces Costs** | Eliminates the need to create separate AWS ELBs for each service |

---

## Components of Ingress in Kubernetes

An Ingress setup consists of the following components:

### 1. Ingress Resource *(Kubernetes Object)*
- Defines routing rules specifying how external traffic reaches services.
- Examples: path-based routing, host-based routing, SSL termination.

### 2. Ingress Controller
- The actual implementation that processes Ingress rules and provisions the underlying AWS ALB or NLB.
- Examples: **AWS Load Balancer Controller**, NGINX Ingress Controller, Traefik.

### 3. Load Balancer
- The external AWS **Application Load Balancer (ALB)** or **Network Load Balancer (NLB)** that routes traffic.
- The ALB Controller automatically creates and manages this.

### 4. Target Groups
- AWS resources that ALB forwards traffic to.
- In **Instance Mode**, the target is EC2 instances.
- In **IP Mode**, the target is pod IPs.

---

## Ingress Controller Modes in AWS EKS

In AWS EKS, the AWS Load Balancer Controller supports two modes for managing traffic:

1. **Instance Mode**
2. **IP Mode**

---

## Instance Mode vs. IP Mode

| Feature | Instance Mode | IP Mode |
|---|---|---|
| **Target Type** | Routes traffic to EC2 instances | Routes traffic directly to pod IPs |
| **Networking** | Uses AWS VPC routing; requires Worker Nodes in a Public Subnet | Uses AWS VPC CNI for direct pod communication |
| **Target Group** | Instance Target Group | IP Target Group |
| **Traffic Flow** | ALB → NodePort → Pods | ALB → Pods (bypasses NodePort) |
| **Use Case** | EC2 worker nodes in public subnets with simpler setup | EKS Fargate or private nodes needing low-latency networking |
| **Security Group** | Worker nodes must allow inbound traffic from ALB | Pods must allow inbound traffic from ALB |

### When to Use Instance Mode
- Your cluster runs EC2 instances and you need a simpler setup.
- When using traditional Kubernetes `NodePort` services.

### When to Use IP Mode
- Running on **EKS Fargate** or using private subnets.
- You want **better performance and lower latency** (direct traffic to pods).
- You want to avoid additional hops through `NodePort`.

---

## Prerequisites

Before you begin, ensure the following tools are installed and configured:

- [`eksctl`](https://eksctl.io/)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/)
- [AWS CLI](https://aws.amazon.com/cli/)

> You must have already created an EKS cluster before proceeding.

---

## Step 1: Connect kubectl to Your EKS Cluster

Create or update your kubeconfig file so that `kubectl` can connect to your EKS cluster:

```bash
aws eks update-kubeconfig --region ap-south-1 --name ekswithavinash
```

---

## Step 2: Create an IAM OIDC Provider for Your Cluster

Set your cluster name in an environment variable:

```bash
cluster_name=ekswithavinash
```

Extract the OIDC ID from your cluster:

```bash
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Check if an IAM OIDC provider already exists:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

- **If output is returned:** An IAM OIDC provider already exists — skip the next command.
- **If no output is returned:** Create one with:

```bash
eksctl utils associate-iam-oidc-provider --cluster ekswithavinash --approve
```

---

## Step 3: Install AWS Load Balancer Controller with Helm

### 3.1 — Download and Create the IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 3.2 — Create the IAM Service Account

```bash
eksctl create iamserviceaccount \
    --cluster=ekswithavinash \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<<account-id>>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-south-1 \
    --approve
```

> Replace `<<account-id>>` with your actual AWS account ID.

### 3.3 — Install the AWS Load Balancer Controller

Add and update the Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ekswithavinash \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 3.4 — Verify the Controller Deployment

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Step 4: Deploy Games Application

### 4.1 — Namespace (`namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: games
```

```bash
kubectl apply -f namespace.yaml
```

---

### 4.2 — Deployment: Tic-Tac-Toe (`deployment_ttt.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-ttt-deployment
  namespace: games
  labels:
    app: game-ttt
spec:
  replicas: 2
  selector:
    matchLabels:
      app: game-ttt
  template:
    metadata:
      labels:
        app: game-ttt
    spec:
      containers:
      - name: game-ttt
        image: sadashivareddy/gamettt:1.0
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment_ttt.yaml
```

---

### 4.3 — Service: Tic-Tac-Toe (`service_ttt.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-ttt-service
  namespace: games
  labels:
    app: game-ttt
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: game-ttt
```

```bash
kubectl apply -f service_ttt.yaml
```

---

### 4.4 — Deployment: Rock-Paper-Scissors (`deployment_rps.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rps-deployment
  namespace: default
  labels:
    app: gamerps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gamerps
  template:
    metadata:
      labels:
        app: gamerps
    spec:
      containers:
      - name: gamerps
        image: sadashivareddy/gamerps:1.0
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment_rps.yaml
```

---

### 4.5 — Service: Rock-Paper-Scissors (`service_rps.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gamerpsservice
  namespace: default
  labels:
    app: gamerps
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: gamerps
```

```bash
kubectl apply -f service_rps.yaml
```

---

### 4.6 — NodePort Services (Optional)

If you prefer to expose services via `NodePort` instead of `ClusterIP`:

**RPS NodePort Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-rps-service
  namespace: games
  labels:
    app: game-rps
spec:
  type: NodePort
  selector:
    app: game-rps
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30081
```

**TTT NodePort Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-ttt-service
  namespace: games
  labels:
    app: game-ttt
spec:
  type: NodePort
  selector:
    app: game-ttt
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

### 4.7 — Ingress (`ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: games-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
  - http:
      paths:
      - path: /rps
        pathType: Prefix
        backend:
          service:
            name: gamerpsservice
            port:
              number: 80
      - path: /ttt
        pathType: Prefix
        backend:
          service:
            name: game-ttt-service
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
```

---

## Step 5: Verify the Deployment

Check the ingress status to confirm the ALB has been provisioned:

```bash
kubectl describe ingress games-ingress -n default
```

Once the ALB is ready, the **Address** field will show the ALB DNS name. Open it in your browser to access the games.

---

## Optional: Route53 + ACM Certificate Setup

If you have a **Route53 Hosted Zone** and an **ACM Certificate**, use the following Ingress manifest to enable HTTPS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-ingress
  namespace: games
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:1234567890:certificate/Your-Cert-ARN
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
spec:
  rules:
  - host: rps.ssrg.online
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: game-rps-service
            port:
              number: 80
  - host: ttt.ssrg.online
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: game-ttt-service
            port:
              number: 80
```

> After applying, create a **CNAME or Alias record** in Route53 pointing `ssrg.online` to the ALB DNS name.

```bash
kubectl apply -f ingress.yaml
```

