Deploy a 3 TIER web app built using JS, java python, go etc
 EKS with EC2 instance
### Install EKS
```
eksctl create cluster --name demo-cluster-three-tier-1 --region us-east-1
```

### Configure IAM OIDC Provider
```
export cluster_name=<CLUSTER-NAME>
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```
### ALB Configuration
Download IAM Policy
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
Create IAM Policy
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

Create IAM Role
```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### Deploy ALB Controller
Add helm repo
```
helm repo add eks https://aws.github.io/eks-charts
```
Update Helm Repo
```
helm repo add eks https://aws.github.io/eks-charts
```
Install
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

Verify Deployments are running
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
Helm chart
Redis as in memory DB, persisted usinf EBS
### Configure EBS CSI Plugin add it as add on
Create an IAM Role and attach a policy. Command deploys an AWS Cloud Formation stack that creates an IAM Role and attaches IAM policy to it
```
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve

eksctl create addon --name aws-ebs-csi-driver --cluster <YOUR-CLUSTER-NAME> --service-account-role-arn arn:aws:iam::<AWS-ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```
Why?- Redis is created as stateful set. Redis required persistent volume. When we create persistent vol claim, storage class loos at claim and has to manage to crete EBS vol. There needs to be a mechanism -- EBS csi PLUGIN

Deploy HELM Chart

