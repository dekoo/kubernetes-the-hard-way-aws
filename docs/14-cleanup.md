# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
for instance in worker-0 worker-1 worker-2 controller-0 controller-1 controller-2; do
  aws ec2 terminate-instances --output text --instance-ids \
    $(aws ec2 describe-instances --output text --query \
      "Reservations[*].Instances[*].{ID:InstanceId}" --filters "Name=instance-state-name,Values=running" "Name=tag:Name,Values=${instance}")
done
```

Delete keys pairs:

```
aws ec2 delete-key-pair --key-name kubernetes-the-hard-way-key --output text
```

## Networking

Delete the external load balancer network resources:

```
{
  aws elbv2 delete-load-balancer --load-balancer-arn $(aws elbv2 describe-load-balancers \
    --names kubernetes-hard-way-nlb --output text --query LoadBalancers[].LoadBalancerArn) \
    --output text

  aws elbv2 delete-target-group --target-group-arn \
    $(aws elbv2 describe-target-groups --names kubernetes-hard-way-tg --output text --query TargetGroups[].TargetGroupArn) --output text
}
```

Delete the `kubernetes-the-hard-way` firewall rules:

```
{
  aws ec2 revoke-security-group-ingress \
    --group-id $(aws ec2 describe-security-groups \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
      --query "SecurityGroups[*].GroupId" --output text) \
    --protocol tcp --port 22 --cidr 0.0.0.0/0 --output text
  
  aws ec2 revoke-security-group-ingress \
    --group-id $(aws ec2 describe-security-groups \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
      --query "SecurityGroups[*].GroupId" --output text) \
    --protocol tcp --port 6443 --cidr 0.0.0.0/0 --output text
  
  aws ec2 revoke-security-group-ingress \
    --group-id $(aws ec2 describe-security-groups \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
      --query "SecurityGroups[*].GroupId" --output text) \
    --protocol icmp --port -1 --cidr 0.0.0.0/0 --output text
	
  aws ec2 revoke-security-group-ingress \
    --group-id $(aws ec2 describe-security-groups \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
      --query "SecurityGroups[*].GroupId" --output text) \
    --protocol tcp --port ${NODE_PORT} --cidr 0.0.0.0/0 --output text
}
```
>>> `NODE_PORT` is the one we set during [smoke test](https://github.com/dekoo/kubernetes-the-hard-way-aws/blob/master/docs/13-smoke-test.md#services)

Delete the `kubernetes-the-hard-way` network VPC and related services:

```
{
  aws ec2 detach-internet-gateway \
    --internet-gateway-id $(aws ec2 describe-internet-gateways \
      --filters "Name=tag:Name,Values=kubernetes-the-hard-way-igw" \
      --query "InternetGateways[*].InternetGatewayId" --output text) \
	  --vpc-id $(aws ec2 describe-vpcs \
      --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" \
      --query "Vpcs[*].VpcId" --output text)

  aws ec2 delete-internet-gateway \
    --internet-gateway-id $(aws ec2 describe-internet-gateways \
      --filters "Name=tag:Name,Values=kubernetes-the-hard-way-igw" \
      --query "InternetGateways[*].InternetGatewayId" --output text)
	
  for i in 0 1 2; do
    aws ec2 delete-subnet --subnet-id $(aws ec2 describe-subnets --filters "Name=tag:Name,Values=kubernetes-subnet-${i}" \
        --query "Subnets[*].SubnetId" --output text) \
      --output text
  done

  aws ec2 delete-vpc --vpc-id $(aws ec2 describe-vpcs \
    --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text) --output text
}
```
