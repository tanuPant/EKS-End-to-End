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
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
### Check address is displayed in ingress
```
kubectl get ingress -n game-2048
```
<img width="958" height="113" alt="image" src="https://github.com/user-attachments/assets/7eb247ef-6171-42f9-8316-2629731c08aa" />

****************
Troubleshooting commands

Get deployment
```
kubectl get deployment -n kube-system
```
<img width="722" height="70" alt="image" src="https://github.com/user-attachments/assets/6c162b17-7937-4d46-b60f-348a642e1efb" />
Both deployment should be in ready state

Troubleshoot ingress address issue
```
kubectl describe ingress ingress-2048 -n game-2048
```
<img width="959" height="96" alt="image" src="https://github.com/user-attachments/assets/4241b0a2-49d1-4d7f-a6c0-3a58c2a75428" />
In the AmazonEKSLoadBalancerControllerRole Role, EC2: DescribeRouteTables is not allowed.
Add it to the AWSLoadBalancerControllerIAMPolicy and reattach it to the role AmazonEKSLoadBalancerControllerRole
<img width="433" height="386" alt="image" src="https://github.com/user-attachments/assets/15c8218b-87cc-4ca4-b21c-ddee5ca762a8" />

<img width="715" height="49" alt="image" src="https://github.com/user-attachments/assets/42374d24-702a-4afb-b37d-aca5cdd0ef7c" />

### Restart Loadbalancer
```
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
```
<img width="749" height="33" alt="image" src="https://github.com/user-attachments/assets/90d40167-9842-4486-ae1f-e3f247f1bc55" />

### Check ingress again
<img width="905" height="101" alt="image" src="https://github.com/user-attachments/assets/ec8f0633-9afd-4623-9cfd-791becd17812" />

Give the address in the address bar
<img width="811" height="533" alt="image" src="https://github.com/user-attachments/assets/f341dacd-bdaa-4fca-9c83-88adc9d845a6" />
Game is running!!






  
