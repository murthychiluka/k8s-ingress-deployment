# Deploy HTTPS Application on Amazon EKS using AWS Load Balancer Controller and ACM

This guide explains how to expose an application running on Amazon EKS using an AWS Application Load Balancer (ALB) with HTTPS enabled through AWS Certificate Manager (ACM).

---

# Architecture

```
                 Internet
                     |
              HTTPS (443)
                     |
            AWS Application Load Balancer
                     |
        AWS Load Balancer Controller
                     |
              Kubernetes Ingress
                     |
              ClusterIP Service
                     |
              NGINX Deployment
                     |
                 Amazon EKS
```

---

# Prerequisites

Before starting, make sure you have:

- An Amazon EKS cluster
- kubectl configured
- eksctl installed
- Helm installed
- AWS CLI configured
- A registered domain
- An SSL/TLS certificate in AWS Certificate Manager (ACM)

---

# Step 1: Request or Import an SSL Certificate

Create or import your SSL/TLS certificate in **AWS Certificate Manager (ACM)**.

After the certificate is issued, copy the **Certificate ARN**.

Example:

```
arn:aws:acm:us-east-1:123456789012:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

# Step 2: Associate IAM OIDC Provider

Replace `<cluster-name>` with your EKS cluster name.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster <cluster-name> \
  --region us-east-1 \
  --approve
```

---

# Step 3: Create IAM Policy

Download the IAM policy.

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create the policy.

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

---

# Step 4: Create IAM Service Account

Replace:

- `<cluster-name>`
- `<ACCOUNT_ID>`

```bash
eksctl create iamserviceaccount \
  --cluster <cluster-name> \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

# Step 5: Install AWS Load Balancer Controller

Add the Helm repository.

```bash
helm repo add eks https://aws.github.io/eks-charts

```

Update repositories.

```bash
helm repo update
```

Install the controller.

Replace:

- `<cluster-name>`
- `<vpc-id>`

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<vpc-id>
```

Verify installation.

```bash
kubectl get pods -n kube-system
```

Expected output:

```
aws-load-balancer-controller-xxxxxxxxx   Running
```

---

# Step 6: Deploy NGINX Application

Create a file named **nginx.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Deploy it.

```bash
kubectl apply -f nginx.yaml
```

Verify.

```bash
kubectl get pods
kubectl get svc
```

---

# Step 7: Create ALB Ingress

Create a file named **ingress.yaml**

Replace the following values before applying:

- `ACCOUNT_ID`
- `CERTIFICATE_ID`
- `app.example.com`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERTIFICATE_ID
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  ingressClassName: alb
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

Apply the ingress.

```bash
kubectl apply -f ingress.yaml
```

---

# Step 8: Verify Ingress

Check the ingress resource.

```bash
kubectl get ingress
```

Example output:

```
NAME        CLASS   HOSTS             ADDRESS
nginx-alb   alb     app.example.com   k8s-xxxxxxxx.us-east-1.elb.amazonaws.com
```

---

# Step 9: Configure DNS

Create a DNS record pointing your domain to the ALB.

Example:

| Record Type | Name | Value |
|-------------|------|-------|
| CNAME | app.example.com | k8s-xxxxxxxx.us-east-1.elb.amazonaws.com |

or use an Alias record if you're using Amazon Route 53.

---

# Step 10: Test HTTPS

Open:

```
https://app.example.com
```

You should see the default NGINX welcome page over HTTPS.

---

# Useful Commands

Check deployments

```bash
kubectl get deployments
```

Check pods

```bash
kubectl get pods
```

Check services

```bash
kubectl get svc
```

Check ingress

```bash
kubectl get ingress
```

Describe ingress

```bash
kubectl describe ingress nginx-alb
```

View controller logs

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

# Troubleshooting

### Ingress ADDRESS is empty

Verify the AWS Load Balancer Controller is running.

```bash
kubectl get pods -n kube-system
```

---

### ALB is not created

Check controller logs.

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

### HTTPS is not working

Verify:

- ACM certificate is issued
- Correct certificate ARN is used
- DNS points to the ALB
- Port 443 listener exists

---

### DNS is not resolving

Check your DNS record.

```bash
nslookup app.example.com
```

---

# Project Structure

```
.
├── README.md
├── nginx.yaml
└── ingress.yaml
```

---

# Summary

This guide covered:

- Requesting an ACM certificate
- Associating the EKS OIDC provider
- Creating IAM policies and service accounts
- Installing the AWS Load Balancer Controller
- Deploying an NGINX application
- Creating an ALB Ingress
- Enabling HTTPS using ACM
- Configuring DNS
- Verifying the application over HTTPS
```
