# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## Routes

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

create network routes for each worker instance:

```
for instance in worker-0 worker-1 worker-2; do
  INSTANCE_ID="$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" \
    --output text --query 'Reservations[].Instances[].[InstanceId]')"
  POD_CIDR="$(aws ec2 describe-instance-attribute \
    --instance-id "${INSTANCE_ID}" \
    --attribute userData \
    --output text --query 'UserData.Value' \
    | base64 --decode | tr "|" "\n" | grep "^pod-cidr" | cut -d'=' -f2)"
  ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text --query 'RouteTables[].Associations[].RouteTableId')
  aws ec2 create-route \
    --route-table-id "${ROUTE_TABLE_ID}" \
    --destination-cidr-block "${POD_CIDR}" \
    --instance-id "${INSTANCE_ID}"
done
```

> output

```
{
    "Return": true
}
{
    "Return": true
}
{
    "Return": true
}
```

List the routes in the `kubernetes` route table:

```
aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes" \
  --output text
```

> output

```
ROUTETABLES	317510512299	rtb-06e061bbeddaab333	vpc-08e16139fa44c7375
ASSOCIATIONS	False	rtbassoc-0a33e138a48005a64	rtb-06e061bbeddaab333	subnet-073dd26b66c51aac6
ROUTES	10.200.0.0/24		i-0826f658ad677da14	317510512299	eni-0059ab79faaa025d2	CreateRoute	active
ROUTES	10.200.1.0/24		i-0d105a3345e267628	317510512299	eni-01c09ab8bdd2e440c	CreateRoute	active
ROUTES	10.200.2.0/24		i-0e3de9e40402ad363	317510512299	eni-059b177d94be2acb4	CreateRoute	active
ROUTES	10.240.0.0/24	local				CreateRouteTable	active
ROUTES	0.0.0.0/0	igw-0074cd27c3f44cc0c				CreateRoute	active
TAGS	Name	kubernetes

```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
