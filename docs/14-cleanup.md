# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
aws ec2 terminate-instances \
  --instance-ids \
    $(aws ec2 describe-instances \
      --filter "Name=tag:Name,Values=controller-0,controller-1,controller-2,worker-0,worker-1,worker-2" \
      --output text --query 'Reservations[].Instances[].InstanceId')
aws ec2 delete-key-pair \
  --key-name kubernetes
```

## Networking

Delete the external load balancer network resources:

```
LOAD_BALANCER_ARN=$(aws elbv2 describe-load-balancers \
  --names kubernetes \
  --output text \
  --query 'LoadBalancers[].LoadBalancerArn')

aws elbv2 delete-load-balancer \
  --load-balancer-arn "${LOAD_BALANCER_ARN}"

TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups \
  --name kubernetes \
  --output text \
  --query 'TargetGroups[].TargetGroupArn')

aws elbv2 delete-target-group \
  --target-group-arn "${TARGET_GROUP_ARN}"

SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'SecurityGroups[].GroupId')
aws ec2 delete-security-group \
  --group-id "${SECURITY_GROUP_ID}"
  
```

Delete the `kubernetes-the-hard-way` firewall rules:

```
gcloud -q compute firewall-rules delete \
  kubernetes-the-hard-way-allow-nginx-service \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external \
  kubernetes-the-hard-way-allow-health-check
```

Delete the `kubernetes-the-hard-way` network VPC:

```
{
  gcloud -q compute routes delete \
    kubernetes-route-10-200-0-0-24 \
    kubernetes-route-10-200-1-0-24 \
    kubernetes-route-10-200-2-0-24

  gcloud -q compute networks subnets delete kubernetes

  gcloud -q compute networks delete kubernetes-the-hard-way
}
```
