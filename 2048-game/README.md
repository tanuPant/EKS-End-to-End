This project demonstrates deploying 2048 game on AWS EKS.
## Prerequisites
1. Install AWS CLI
2. Configure AWS CLI Credentials aws configure
3. Configure kubectl , update kubeconfig
   aws eks update-kubeconfig --name your-cluster-name
   Verify the configuration
   kubectl get nodes
 ---  
## 2048 App
Create Fargate profile
```
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```
##  Deploy the deployment, service and Ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
---
## Configure IAM OIDC Connector
```
export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 
```
### Configure IAM Provider
```
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```
---
## Setup ALB add on
### Download IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
### Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
### Create IAM Role
```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
   --override-existing-serviceaccounts  \
  --approve
```
### Deploy ALB controller
```
helm repo add eks https://aws.github.io/eks-charts
```
### Update the repo
```
helm repo update eks
```
### Install
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<your-region> \
  --set vpcId=<your-vpc-id>
```
### Verify that the deployments are running.
kubectl get deployment -n kube-system aws-load-balancer-controller



  
