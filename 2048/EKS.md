# Setting up AWS Load Balancer Controller on EKS with Fargate

This guide walks through the process of setting up an AWS Load Balancer Controller on Amazon EKS with Fargate, including the deployment of a sample 2048 game application. It also covers common challenges and their solutions.

## Prerequisites

- AWS CLI configured with appropriate permissions
- kubectl installed
- helm installed
- eksctl installed

## Step 1: Create EKS Cluster with Fargate

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

## Step 2: Configure IAM OIDC Provider

The OIDC provider is necessary for IAM roles for service accounts (IRSA).

```bash
# Export cluster name
export cluster_name=demo-cluster

# Get OIDC ID
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Check for existing OIDC provider
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

# Associate IAM OIDC provider if not exists
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## Step 3: Set Up AWS Load Balancer Controller

### 3.1 Download and Create IAM Policy

```bash
# Download policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 3.2 Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 3.3 Install ALB Controller using Helm

```bash
# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

## Step 4: Deploy Sample Application (2048 Game)

### 4.1 Create Fargate Profile

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### 4.2 Deploy Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

## Challenges Faced and Solutions

### 1. IAM Permission Issues

**Challenge:**
- Load Balancer Controller encountered 403 AccessDenied errors
- Could not perform actions like `elasticloadbalancing:DescribeListenerAttributes` and `ec2:GetSecurityGroupsForVpc`

**Root Cause:**
- Insufficient IAM permissions in the controller's role
- Missing critical permissions for ALB management

**Solution:**
- Created comprehensive IAM policy with necessary permissions
- Attached additional permissions for ELB and EC2 operations
- Properly configured IRSA (IAM Roles for Service Accounts)

### 2. Target Group Registration Issues

**Challenge:**
- ALB created successfully but no targets were registered
- Empty target group despite running pods

**Root Cause:**
- Incomplete security group configuration
- Missing ingress annotations for target type

**Solution:**
- Added proper security group rules for port 80
- Configured correct ingress annotations:
  ```yaml
  alb.ingress.kubernetes.io/target-type: ip
  alb.ingress.kubernetes.io/scheme: internet-facing
  ```

### 3. Load Balancer Webhook Service Issues

**Challenge:**
- Error: "no endpoints available for service aws-load-balancer-webhook-service"
- Controller deployment issues

**Root Cause:**
- Incomplete controller deployment
- Service account configuration issues

**Solution:**
- Properly reinstalled the controller with correct Helm values
- Ensured service account had correct annotations
- Verified OIDC provider configuration

## Best Practices and Learnings

1. **IAM Configuration:**
   - Always verify IAM permissions first
   - Use principle of least privilege
   - Properly configure IRSA

2. **Networking:**
   - Ensure proper security group configuration
   - Verify VPC and subnet settings
   - Configure correct ingress annotations

3. **Troubleshooting:**
   - Check controller logs
   - Verify pod and service status
   - Monitor ingress events

## Cleanup

To delete all resources:

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

## References
- iamveeramalla
- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)
  
