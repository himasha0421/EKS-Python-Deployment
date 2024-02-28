# EKS-Python-Deployment

## system design architecture

![]("eks-microservices.drawio.png")

scalable python solution deployment using EKS managed service.

step1. setup eks cluster

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

step2. update the kube config to work with eks cluster

```bash
aws eks update-kubeconfig --name bot-cluster --region us-east-1
```

step3. create fargate profile to create namespace deployments

```bash
eksctl create fargateprofile \
    --cluster bot-cluster \
    --region us-east-1 \
    --name alb-python-app \
    --namespace game-2048
```

step4. deploy the application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

configure IAM OIDC provider

```bash

eksctl utils associate-iam-oidc-provider --cluster bot-cluster --approve --region us-east-1
```

## setup alb addon

ALB controller is nothing but a kubernetes pod , so it has to create a application load balancer inside AWS , for that it need to communicate with AWS ALB service , which needs service role with permissions.

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

step1. create IAM policy using the json file

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

step2. create a service role with the policy attached

```bash
eksctl create iamserviceaccount \
  --cluster=bot-cluster \
  --region us-east-1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::630210676530:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## Deploy ALB controller

step1. add helm repo

```bash
helm repo add eks https://aws.github.io/eks-charts
```

step2. update the helm repo

```bash
helm repo update eks
```

step3. install aws-load-balancer controller

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
  --set clusterName=bot-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0f6e5e88266ad0a7b
```
