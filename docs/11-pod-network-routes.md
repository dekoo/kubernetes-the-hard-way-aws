# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```
for i in 0 1 2; do
  aws ec2 describe-instances --output text --query \
    "Reservations[*].Instances[*].{Name:Tags[?Key=='Name'].Value | [0],PublicIP:PrivateIpAddress}" \
	 --filters "Name=tag:Name,Values=worker-${i}"

  echo "POD_CIDR=10.200."${i}".0/24"
done
```

> output

```
worker-0        10.240.0.21
POD_CIDR=10.200.0.0/24
worker-1        10.240.1.21
POD_CIDR=10.200.1.0/24
worker-2        10.240.2.21
POD_CIDR=10.200.2.0/24
```

## Routes

Create network routes for each worker instance:

```
for i in 0 1 2; do
  aws ec2 create-route \
    --route-table-id $(aws ec2 describe-route-tables \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" \
        --query "Vpcs[*].VpcId" --output text)" \
      --query "RouteTables[*].RouteTableId" \
      --output text) \
    --destination-cidr-block "10.200."${i}".0/24" \
    --instance-id $(aws ec2 describe-instances \
      --output text --query "Reservations[*].Instances[*].{ID:InstanceId}"  \
	  --filters "Name=tag:Name,Values=worker-"${i}) \
    --output text
done 
```

List the routes in the `kubernetes-the-hard-way` VPC network:

```
aws ec2 describe-route-tables --route-table-ids $(aws ec2 describe-route-tables \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" \
        --query "Vpcs[*].VpcId" --output text)" \
      --query "RouteTables[*].RouteTableId" \
      --output text) \
	--query "RouteTables[*].Routes[*].{State:State,Destination:DestinationCidrBlock,Instance:InstanceId,Gateway:GatewayId}" \
	--output table
```

> output

```
-----------------------------------------------------------------------------
|                            DescribeRouteTables                            |
+---------------+-------------------------+-----------------------+---------+
|  Destination  |         Gateway         |       Instance        |  State  |
+---------------+-------------------------+-----------------------+---------+
|  10.200.0.0/24|  None                   |  i-08263e279971d4442  |  active |
|  10.200.1.0/24|  None                   |  i-0794a71e63c46ee24  |  active |
|  10.200.2.0/24|  None                   |  i-01e503d1bdaf06b6a  |  active |
|  10.240.0.0/22|  local                  |  None                 |  active |
|  0.0.0.0/0    |  igw-051f2c6434be15629  |  None                 |  active |
+---------------+-------------------------+-----------------------+---------+
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
