# Enabling Pod Network

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) and NAT blocked by EC2 default [settings](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_NAT_Instance.html#EIP_Disable_SrcDestCheck)

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address and disable source\destination ip address check on workers EC2 nodes.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

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

List the routes in the `kubernetes-the-hard-way-vpc` VPC network:

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


## Source\destination check on Workers

Since `kube-proxy` configures lot of network address translation rules as explained in this [article](https://medium.com/@seifeddinerajhi/kube-proxy-and-cni-the-hidden-components-of-kubernetes-networking-eb30000bf87a) we need to disable source/destination check (enabled by default) on our worker EC2 instances as it wont always be the instance the source or destination of any traffic it sends or receives. 

```
for instance in worker-0 worker-1 worker-2; do
  aws ec2 modify-instance-attribute --instance-id \
    $(aws ec2 describe-instances --output text --query "Reservations[*].Instances[*].{ID:InstanceId}" \
	  --filters "Name=tag:Name,Values=${instance}") \
	    --no-source-dest-check
done
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
