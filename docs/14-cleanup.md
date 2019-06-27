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

Delete the network resources:

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
  
ROUTE_TABLE_ASSOCIATION_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'RouteTables[].Associations[].RouteTableAssociationId')

aws ec2 disassociate-route-table \
  --association-id "${ROUTE_TABLE_ASSOCIATION_ID}"

ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'RouteTables[].RouteTableId')

aws ec2 delete-route-table \
  --route-table-id "${ROUTE_TABLE_ID}"
  
INTERNET_GATEWAY_ID=$(aws ec2 describe-internet-gateways \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'InternetGateways[].InternetGatewayId')

VPC_ID=$(aws ec2 describe-internet-gateways \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'InternetGateways[].Attachments[].VpcId')
aws ec2 detach-internet-gateway \
  --internet-gateway-id "${INTERNET_GATEWAY_ID}" \
  --vpc-id "${VPC_ID}"
  
aws ec2 delete-internet-gateway \
  --internet-gateway-id "${INTERNET_GATEWAY_ID}"

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'Subnets[].SubnetId')

aws ec2 delete-subnet \
  --subnet-id "${SUBNET_ID}"

aws ec2 delete-vpc \
  --vpc-id "${VPC_ID}"
  
ELASTIC_IP_ALLOCATION_ID=$(aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text \
  --query 'Addresses[].AllocationId')
  
aws ec2 release-address \
  --allocation-id "${ELASTIC_IP_ALLOCATION_ID}"
```
