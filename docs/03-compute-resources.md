# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [availability zone](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).

> Ensure a default availability zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.amazonaws.cn/en_us/vpc/latest/userguide/what-is-amazon-vpc.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.240.0.0/24 \
  --output text --query 'Vpc.VpcId')

aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=kubernetes-the-hard-way
 
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-support '{"Value": true}'
  
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-hostnames '{"Value": true}'
```


A [subnet](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.240.0.0/24 \
  --output text --query 'Subnet.SubnetId')

aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=kubernetes
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

For exposing the instances in your VPC to the internet you will create an [internet gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html):

```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text --query 'InternetGateway.InternetGatewayId')

aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=kubernetes

aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
```

Create a [route table](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) to route outgoing traffic through the internet gateway:

```
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} \
  --output text --query 'RouteTable.RouteTableId')

aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=kubernetes

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}

aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
```

### Security Groups

Create a security group that allows internal communication across all protocols:

```
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')

aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=kubernetes

aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol all \
  --cidr 10.240.0.0/24

aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol all \
  --cidr 10.200.0.0/16

aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol icmp \
  --port all \
  --cidr 0.0.0.0/0
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
ELASTIC_IP_ARN=$(aws ec2 allocate-address \
  --domain vpc \
  --output text --query 'AllocationId')
  
aws ec2 create-tags \
  --resources ${ELASTIC_IP_ARN} \
  --tags Key=Name,Value=kubernetes

KUBERNETES_PUBLIC_IP=$(aws ec2 describe-addresses \
  --allocation-ids $ELASTIC_IP_ARN \
  --output text | awk '{print $4}')
```

### Elastic Load Balancer

An [elastic load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) will be used to expose the Kubernetes API Servers to remote clients:

```
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
  --name kubernetes \
  --scheme internet-facing \
  --type network \
  --subnet-mappings SubnetId=${SUBNET_ID},AllocationId=$ELASTIC_IP_ARN \
  --output text --query 'LoadBalancers[].LoadBalancerArn')
```

We need to specify a [target group](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) For the load balancer which point to the EC2 instances by their IP:

```
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name kubernetes \
  --protocol TCP \
  --port 6443 \
  --vpc-id ${VPC_ID} \
  --target-type ip \
  --output text --query 'TargetGroups[].TargetGroupArn')
```
Register the targets to the load balancer:

```
aws elbv2 register-targets \
  --target-group-arn ${TARGET_GROUP_ARN} \
  --targets Id=10.240.0.1{0,1,2}
```

After registering the target group to the load balancer, it required to add a [listener](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html), who checks for connection requests and tells the load balancer how to route them into the target group:

```
aws elbv2 create-listener \
  --load-balancer-arn ${LOAD_BALANCER_ARN} \
  --protocol TCP \
  --port 6443 \
  --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
  --output text --query 'Listeners[].ListenerArn'
```


## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

Getting the newest Ubuntu 18.04 image ID:

```
IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
  --filters \
  'Name=root-device-type,Values=ebs' \
  'Name=architecture,Values=x86_64' \
  'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-*' \
  | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
```

Creating SSH key pair to authorize with for gaining access to the instances:

```
mkdir -p KubernetesPractice

cd KubernetesPractice

mkdir -p ssh

aws ec2 create-key-pair \
  --key-name kubernetes \
  --output text --query 'KeyMaterial' \
  > ssh/kubernetes.id_rsa
chmod 600 ssh/kubernetes.id_rsa
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 10.240.0.1${i} \
    --user-data "name=controller-${i}" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=controller-${i}"
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name kubernetes \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 10.240.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute \
    --instance-id ${instance_id} \
    --no-source-dest-check
  aws ec2 create-tags \
    --resources ${instance_id} \
    --tags "Key=Name,Value=worker-${i}"
done
```

### Verification

List the running compute instances in your default compute zone:

```
aws ec2 describe-instances \
--query 'Reservations[*].Instances[*].[Tags]' \
--filters Name=instance-state-name,Values=running Name=subnet-id,Values=$SUBNET_ID \
--output text
```

> output

```
Name	controller-1
Name	controller-2
Name	controller-0
Name	worker-0
Name	worker-1
Name	worker-2
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
