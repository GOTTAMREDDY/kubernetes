# EKS: Service Accounts, OIDC & IRSA Explained

> A complete guide to pod-level AWS permissions in EKS — from core concepts to real-world implementation.

---

## Table of Contents

- [Without Service Account + IRSA](#without-service-account--irsa)
- [Core Concepts](#core-concepts)
  - [1. Service Account (SA)](#1-service-account-sa)
  - [2. OIDC](#2-oidc)
  - [3. IRSA](#3-irsa)
- [One-liner Definitions](#one-liner-definitions)
- [School Analogy](#school-analogy)
- [Interview Answers](#interview-answers)
- [The Easiest Memory Trick](#the-easiest-memory-trick)
- [Real-World EKS Setup](#real-world-eks-setup)
  - [Scenario](#scenario)
  - [Step 1: Create EKS Cluster with OIDC](#step-1-create-eks-cluster-with-oidc)
  - [Step 2: Create IAM Policies](#step-2-create-iam-policies)
  - [Step 3: Create Service Accounts + IAM Roles (IRSA)](#step-3-create-service-accounts--iam-roles-irsa)
  - [Step 4: Verify Service Accounts](#step-4-verify-service-accounts)
  - [Step 5: Deploy Pod A (S3)](#step-5-deploy-pod-a-s3)
  - [Step 6: Deploy Pod B (Route53)](#step-6-deploy-pod-b-route53)
  - [Step 7: Deploy Pod C (DynamoDB)](#step-7-deploy-pod-c-dynamodb)
- [Security Verification](#security-verification)
- [Architecture Diagram](#architecture-diagram)
- [Summary](#summary)

---

## Without Service Account + IRSA

Your pod can still access AWS services if:

- The worker node EC2 instance has an IAM role attached.
- The pod uses the node's IAM permissions.

**Example:**

```
Pod --> Node IAM Role --> S3
```

**Problem:**

- Every pod on that node gets the **same permissions**.
- Violates the **least-privilege** principle.

---

## Core Concepts

### 1. Service Account (SA)

**Definition:** A Service Account is a Kubernetes identity assigned to a pod.

Think of it like:

| Entity | Identity |
|--------|----------|
| Human User | Username |
| Pod | Service Account |

**Example:**

```yaml
serviceAccountName: sa-s3
```

This means: *"This pod's identity is `sa-s3`."*

> ⚠️ A Service Account does **not** contain AWS permissions. It is just an **identity**.

---

### 2. OIDC

**Definition:** OIDC (OpenID Connect) is a trust mechanism that allows AWS to verify Kubernetes identities.

**The problem it solves:**

```
Kubernetes says: "This pod is using Service Account sa-s3"
AWS says:        "How do I trust you?"
OIDC provides:   The proof.
```

**Without OIDC:**

```
Kubernetes ---> AWS
               AWS: I don't know who you are.
```

**With OIDC:**

```
Kubernetes ---> OIDC ---> AWS
                          AWS: Okay, I trust this identity.
```

> OIDC gives **authentication**.

---

### 3. IRSA

**Definition:** IRSA (IAM Roles for Service Accounts) is the mechanism that links a Kubernetes Service Account to an AWS IAM Role.

**Example mapping:**

```
Service Account: sa-s3
        ↓
IAM Role: S3ReadOnlyRole
```

This mapping **is** IRSA.

> IRSA answers: *"This Service Account gets which IAM Role?"*

---

## One-liner Definitions

| Concept | Definition |
|---------|-----------|
| **Service Account** | Kubernetes identity for a pod. |
| **OIDC** | Trust relationship that allows AWS to authenticate Kubernetes Service Accounts. |
| **IRSA** | Feature that maps a Kubernetes Service Account to an AWS IAM Role. |

---

## School Analogy

Imagine a school.

### Service Account → Student ID Card

```
Student: Ravi
ID Card: 123
```

The ID card identifies Ravi. That's the **Service Account**.

### OIDC → Security Guard

```
Ravi
 ↓
ID Card
 ↓
Guard verifies
```

The guard checks whether the ID card is genuine. That's **OIDC** — verification/trust.

### IRSA → Access Mapping

After verifying Ravi's ID card, the school decides:

```
ID Card 123  →  Library Access
ID Card 456  →  Lab Access
```

That **mapping** is **IRSA**.


---

## The Easiest Memory Trick

```
SA   = Who am I?
OIDC = How does AWS trust who I am?
IRSA = What IAM Role do I get after AWS trusts me?
```

That's the core idea. Everything else in EKS is built on top of those three concepts.

---

## Real-World EKS Setup

### Scenario

You have an EKS cluster and 3 applications:

| Pod | AWS Service |
|-----|------------|
| Pod A | S3 |
| Pod B | Route53 |
| Pod C | DynamoDB |

Each pod should have **only the permissions it needs**.

---

### Step 1: Create EKS Cluster with OIDC

```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: dev-cluster
  region: us-east-1

iam:
  withOIDC: true

managedNodeGroups:
  - name: workers
    instanceType: t3.medium
    desiredCapacity: 3
```

```bash
eksctl create cluster -f cluster.yaml
```

**Verify OIDC:**

```bash
aws eks describe-cluster \
  --name dev-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Expected output:

```
https://oidc.eks.us-east-1.amazonaws.com/id/ABC123XYZ
```

---

### Step 2: Create IAM Policies

#### S3 Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name EKS-S3-Policy \
  --policy-document file://s3-policy.json
```

#### Route53 Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:*"],
      "Resource": "*"
    }
  ]
}
```

#### DynamoDB Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:*"],
      "Resource": "*"
    }
  ]
}
```

---

### Step 3: Create Service Accounts + IAM Roles (IRSA)

**S3 Service Account:**

```bash
eksctl create iamserviceaccount \
  --cluster dev-cluster \
  --namespace default \
  --name sa-s3 \
  --attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/EKS-S3-Policy \
  --approve
```

**Route53 Service Account:**

```bash
eksctl create iamserviceaccount \
  --cluster dev-cluster \
  --namespace default \
  --name sa-route53 \
  --attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/EKS-Route53-Policy \
  --approve
```

**DynamoDB Service Account:**

```bash
eksctl create iamserviceaccount \
  --cluster dev-cluster \
  --namespace default \
  --name sa-dynamodb \
  --attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/EKS-DynamoDB-Policy \
  --approve
```

> What `eksctl` does under the hood:
> ```
> ServiceAccount → IAM Role → Trust Policy (OIDC)
> ```
> This is **IRSA**.

---

### Step 4: Verify Service Accounts

```bash
kubectl get sa
```

Expected output:

```
sa-s3
sa-route53
sa-dynamodb
```

---

### Step 5: Deploy Pod A (S3)

```yaml
# pod-s3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-s3
spec:
  serviceAccountName: sa-s3
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-s3.yaml
kubectl exec -it pod-s3 -- sh

# Test S3 access
aws s3 ls
```

✅ Works — the IAM Role grants S3 permissions.

---

### Step 6: Deploy Pod B (Route53)

```yaml
# pod-route53.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-route53
spec:
  serviceAccountName: sa-route53
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep", "3600"]
```

```bash
# Test Route53 access
aws route53 list-hosted-zones
```

✅ Works.

---

### Step 7: Deploy Pod C (DynamoDB)

```yaml
# pod-dynamodb.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dynamodb
spec:
  serviceAccountName: sa-dynamodb
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep", "3600"]
```

```bash
# Test DynamoDB access
aws dynamodb list-tables
```

✅ Works.

---

## Security Verification

Inside the S3 pod, try accessing other services:

```bash
aws route53 list-hosted-zones
# Expected: AccessDenied ❌

aws dynamodb list-tables
# Expected: AccessDenied ❌
```

> This proves **least-privilege access** — each pod can only access what it's permitted to.

---

## Architecture Diagram

```
                    OIDC Provider
                          │
                          ▼
                  AWS IAM Trust
                          │
         ┌────────────────┼────────────────┐
         │                │                │
      Pod-S3          Pod-Route53      Pod-DynamoDB
         │                │                │
         ▼                ▼                ▼
     SA: sa-s3       SA: sa-route53   SA: sa-dynamodb
         │                │                │
         ▼                ▼                ▼
    IAM Role S3     IAM Role Route53  IAM Role DynamoDB
         │                │                │
         ▼                ▼                ▼
     Amazon S3         Route53          DynamoDB
```

---

## Summary

| Component | Role |
|-----------|------|
| **Service Account** | Pod identity |
| **OIDC** | AWS trusts that identity |
| **IRSA** | Connects that identity to an IAM Role |
| **IAM Role** | Defines AWS permissions |

> 🔑 The pod gets AWS access **without storing any access keys**.
