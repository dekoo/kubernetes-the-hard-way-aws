# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster within single [region](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

As a result of this lab we will have following infrastructure available:

<img alt="Compute infrastructure" style="border-width:0" src="https://github.com/dekoo/kubernetes-the-hard-way-aws/blob/master/schemas/kubernetes-the-hard-way-aws-compute.png?raw=true" />

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```
aws ec2 create-vpc --cidr-block 10.240.0.0/22  \
	--tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=kubernetes-the-hard-way-vpc},{Key=Project,Value=kubernetes-the-hard-way}]'  \
	--query Vpc.VpcId --output text
```

Three [subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html) must be provisioned in each of three availability zones, with an IP address ranges large enough to assign a private IP address to each node in the Kubernetes cluster. 

Create the `kubernetes` subnet in the `kubernetes-the-hard-way-vpc` VPC network:

```
for i in 0 1 2; do
  aws ec2 create-subnet \
    --availability-zone $(aws ec2 describe-availability-zones --query "AvailabilityZones[${i}].ZoneName" --output text) \
    --vpc-id $(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text) \
    --cidr-block 10.240.${i}.0/24  \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=kubernetes-subnet-${i}},{Key=Project,Value=kubernetes-the-hard-way}]" \
    --query Subnet.SubnetId --output text
done
```
> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

A [security group](https://docs.aws.amazon.com/vpc/latest/userguide/security-groups.html) acts as a stateless firewall that controls the traffic allowed to and from the resources in your virtual private cloud (VPC). Associated with EC2 instances. You can choose the ports and protocols to allow for inbound traffic and for outbound traffic.

Security groups created by default with VPC allows inbound traffic only within given VPC. Add rules that would allow SSH, ICMP and HTTPS inbound traffic from outside world:

```
aws ec2 authorize-security-group-ingress \
  --group-id $(aws ec2 describe-security-groups \
    --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
       --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
    --query "SecurityGroups[*].GroupId" --output text) \
  --protocol tcp --port 22 --cidr 0.0.0.0/0 --output text
```
```
aws ec2 authorize-security-group-ingress \
  --group-id $(aws ec2 describe-security-groups \
    --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
       --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
    --query "SecurityGroups[*].GroupId" --output text) \
  --protocol tcp --port 6443 --cidr 0.0.0.0/0 --output text
```
```
aws ec2 authorize-security-group-ingress \
  --group-id $(aws ec2 describe-security-groups \
    --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
       --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" --query "Vpcs[*].VpcId" --output text)" \
    --query "SecurityGroups[*].GroupId" --output text) \
  --protocol icmp --port -1 --cidr 0.0.0.0/0 --output text
```

[Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html#nacl-basics) is another stateful way to control traffic to subnets. By default allow all traffic, hence we leave them as is. 

### Internet Gateway
In order to allow access to our compute resources within VPC from local machine via SSH we need to create Internet Gateway and attach it to our VPC:

```
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=kubernetes-the-hard-way-igw},{Key=Project,Value=kubernetes-the-hard-way}]' \
  --output text
```
```
aws ec2 attach-internet-gateway \
  --internet-gateway-id $(aws ec2 describe-internet-gateways \
    --filters "Name=tag:Name,Values=kubernetes-the-hard-way-igw" \
    --query "InternetGateways[*].InternetGatewayId" --output text) \
  --vpc-id $(aws ec2 describe-vpcs \
    --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" \
    --query "Vpcs[*].VpcId" --output text)
```

Then configure routing tables to allow outbound traffic to "0.0.0.0/0" via Internet Gateway.

```
aws ec2 create-route --route-table-id $(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs \
      --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" \
      --query "Vpcs[*].VpcId" --output text)" \
    --query "RouteTables[*].RouteTableId" \
    --output text) \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $(aws ec2 describe-internet-gateways \
    --filters "Name=tag:Name,Values=kubernetes-the-hard-way-igw" \
    --query "InternetGateways[*].InternetGatewayId" --output text) \
  --output text
```
## The Kubernetes Frontend Load Balancer

> An [elastic load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) will be used to expose the Kubernetes API Servers to remote clients.

### Provision a Network Load Balancer

Create empty target group for the port `6443` which is default for the Kubernetes API Server:

```
aws elbv2 create-target-group --name kubernetes-hard-way-tg \
  --protocol TCP --port 6443 --health-check-protocol HTTPS \
  --health-check-port 6443 --health-check-path '/healthz' \
  --target-type instance --vpc-id $(aws ec2 describe-vpcs \
    --filters "Name=tag:Name,Values=kubernetes-the-hard-way-vpc" \
    --query "Vpcs[*].VpcId" --output text) \
  --tags 'Key=Project,Value=kubernetes-the-hard-way' --output text --query TargetGroups[].TargetGroupName
```

Create Network Load Balancer:

```
aws elbv2 create-load-balancer --name kubernetes-hard-way-nlb \
  --subnets \
    $(aws ec2 describe-subnets --filters "Name=tag:Name,Values=kubernetes-subnet-0" \
      --query "Subnets[*].SubnetId" --output text) \
    $(aws ec2 describe-subnets --filters "Name=tag:Name,Values=kubernetes-subnet-1" \
      --query "Subnets[*].SubnetId" --output text) \
    $(aws ec2 describe-subnets --filters "Name=tag:Name,Values=kubernetes-subnet-2" \
      --query "Subnets[*].SubnetId" --output text) \
  --tags 'Key=Project,Value=kubernetes-the-hard-way' --type network \
  --output text --query LoadBalancers[].DNSName

aws elbv2 modify-load-balancer-attributes --load-balancer-arn $(aws elbv2 describe-load-balancers \
    --names kubernetes-hard-way-nlb --output text --query LoadBalancers[].LoadBalancerArn) \
    --attributes 'Key=load_balancing.cross_zone.enabled,Value=true' --output text
```

Associate Network Load Balancer with the Target Group:

```
aws elbv2 create-listener --load-balancer-arn $(aws elbv2 describe-load-balancers \
    --names kubernetes-hard-way-nlb --output text --query LoadBalancers[].LoadBalancerArn) \
  --protocol TCP --port 6443 --default-actions Type=forward,TargetGroupArn=$(aws elbv2 describe-target-groups \
    --names kubernetes-hard-way-tg --output text --query TargetGroups[].TargetGroupArn) --output text
```

> Configuration of the targets within the group will be completed in [Bootstrapping Kubernetes Controllers](08-bootstrapping-kubernetes-controllers.md) section, once control plane is ready. 

## Compute Instances

The compute instances in this lab will be provisioned using [Amazon Linux](https://aws.amazon.com/amazon-linux-2/?amazon-linux-whats-new.sort-by=item.additionalFields.postDateTime&amazon-linux-whats-new.sort-order=desc), which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address within respective subnet to simplify the Kubernetes bootstrapping process.

> We will be using [Spot Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html) in this lab to save on the budget

### Key Pair

Create key pair that will be used later to connect to the instances via SSH:

```
aws ec2 create-key-pair --key-name kubernetes-the-hard-way-key --output text
```
	
> Save key-material from the output in the `kubernetes-the-hard-way-key.pem` file, i.e. text between below mentioned "BEGIN", "END" sections inclusive those:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAkya1SjZZefmz3N9tY1eCktTCl5Tn9kTFH+0nulFddk1fyLzV
ePt4QRIWl1BpQThVT6hfxwhNjsyenHhEU0S4CzfzusU5ja/Ee6ha4I4JaUJmUQcs
1ZDb45gruBK+eh3tseOiIi5E EXAMPLE SKNtUhICkju2/y8tRNW01eKNdHDzzDq
52khXhJAfRrXu9n+5iqSk0W0 EXAMPLE SFUG86qF3Qq64XF5BUX1MaZgBgbxEys
AAZQkOYi486aJktbjJvFuhpZ EXAMPLE jLuywX1paD096BZxXhnYTOsjM7Z+UHk
fji6ouutxsvyfTXikBt2nAD4 EXAMPLE 4JaTwIDAQABAoIBACMC3bWHkuhzofjW
bCdrxdR7rMT2F+6/VAuRmJc7 EXAMPLE sDTTDxnOlrMNg7fgWTPkeJANnvYcZCX
COKrAgMhT+tLS7NLc7tcRisR EXAMPLE +AcdEUFirlkNE/H2SsvFv989Laxikhd
w9CH8SGFziYsTjCnRJrsvAbS EXAMPLE TXZTpRddOQbGmIFytXNx33HkQaKFPW4
iZH9CdqYVq2ijFqiIk4iMKnn EXAMPLE l6ssWJw83DLu9UFW0IGXnZ0wnPVDDzP
Kiy8aYis7FQIjq1rHzlZ5nTU EXAMPLE 9dint7f46YoK5LfuDF3mTd1T3CaW+L1
Tx8tFokCgYEAyQB7rlYAo4ow EXAMPLE 1RIvlUeOAHoTRg+HUzB8uxe18AAvUUp
GBrhOhmwunz9Y5Ai6a+IZs80 EXAMPLE HeekyZoFRisc1843n8PW7RyGVijAdVC
X6TGtZjoFxuXBLP117NHau3i EXAMPLE 9Rpqn/OVqy7hh5LyTKf66MCgYEAu2op
tFB8AUn5c3KYn67E/bgw152a EXAMPLE DTzmbvN6BQ3ja16sPdxYuSC7c5Iwix+
6hCs7qWY7T7EepMm6fGERhV9 EXAMPLE 4Vj7V4FAsBksiuMP29l7ieHl46acUDX
6hzogd8M+iii6MgSGJhEyhs0 EXAMPLE cdyQWUCgYEAkHC3cD0/MkZggTqt9F/X
eu+DYrsZ/xdnztbn8/gvu5ie EXAMPLE 305cp35gNnG4OA4JoPMWkz2O+FMLtqF
0Ds9ifLkgpx7eGDqJgFakQTn EXAMPLE vBHF0JtLgXWjTuhI8MiRDX0jgDTQuXZ
vhHXaP100pZIH4Xv4gJuJ08C EXAMPLE NJ73mbUkjLhhF2NZ81zvkXTGG/GDaGW
16IPJKMwmRuinv+Btoajh8VS EXAMPLE 4zgb0fXznRMtUWGGw60jkvgltNRcPtc
k/hQGV8/xC8X0A4TeX7W+ZWU EXAMPLE UIP8YXpMkwzWRpVhxqExP/SqlO9mVgb
QSDfSQKBgQCz4IUROwURKN2X EXAMPLE wF6D1bRIQCE5vT5L8QAMD1yCL93jWwg
LBuJesSE7YOym3AJ/Hu7gpYePzgF5qCgiP+j48N2HsBVbbltx2cNUJxZBH4zFZge
62kw1mp8lrEBE78JFQgz4HQDL/8Ox5E3lSTKoMOcabGhloxG6F1IXA==
-----END RSA PRIVATE KEY-----
```

Adjust permissions on the file to make it read-writable only for you as SSH demands: 

```
chmod 400 kubernetes-the-hard-way-key.pem
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

> Note that private IP address must belong to the selected subnet which in turn must correspond to one of the availability zones (AZ) in a way to ensure even spread of controllers instances across three AZ.

> Network Load Balancer DNS is passed into controller instances via [user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) as an environment variable `PUBLIC_API_DNS` to facilitate configuration of API server on [Bootstraping Kubernetes Controllers](https://github.com/dekoo/kubernetes-the-hard-way-aws/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) step. 

```
PUBLIC_API_DNS=$(aws elbv2 describe-load-balancers --names kubernetes-hard-way-nlb \
  --output text --query LoadBalancers[].DNSName)

for i in 0 1 2; do
  aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 \
    --instance-type t2.micro \
    --key-name kubernetes-the-hard-way-key \
    --subnet-id $(aws ec2 describe-subnets --filters "Name=tag:Name,Values=kubernetes-subnet-${i}" --query "Subnets[*].SubnetId" --output text) \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=controller-${i}},{Key=Project,Value=kubernetes-the-hard-way}]" \
    --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' \
    --associate-public-ip-address \
    --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' \
    --private-ip-address 10.240.${i}.11 \
    --output text --query "Instances[0].InstanceId" \
    --user-data "$(printf '#cloud-config\n\nruncmd:\n - echo export PUBLIC_API_DNS='${PUBLIC_API_DNS}' >> /etc/bashrc \n - export PUBLIC_API_DNS='${PUBLIC_API_DNS}')')"
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `POD_CIDR` environment variable set via [user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 \
    --instance-type t2.medium \
    --key-name kubernetes-the-hard-way-key \
    --subnet-id $(aws ec2 describe-subnets --filters "Name=tag:Name,Values=kubernetes-subnet-${i}" --query "Subnets[*].SubnetId" --output text) \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=worker-${i}},{Key=Project,Value=kubernetes-the-hard-way}]" \
    --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' \
    --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' \
    --private-ip-address 10.240.${i}.21 \
    --output text --query "Instances[0].InstanceId" \
    --user-data "$(printf '#cloud-config\n\nruncmd:\n - echo \"%s\" >> /etc/bashrc \n - export POD_CIDR=10.200.'${i}'.0/24' 'export POD_CIDR=10.200.'${i}'.0/24')"
done
```

### Verification

List the compute instances in your default region:

```
aws ec2 describe-instances --output table --query "Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key=='Name'].Value | [0],PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress,Status:State.Name}"
```

> output

```
-------------------------------------------------------------------------------------
|                                 DescribeInstances                                 |
+----------------------+---------------+--------------+-----------------+-----------+
|          ID          |     Name      |  PrivateIP   |    PublicIP     |  Status   |
+----------------------+---------------+--------------+-----------------+-----------+
|  i-0bbef0f8e80e8d5e1 |  worker-2     |  10.240.2.21 |  52.90.124.94   |  running  |
|  i-02324df4715e710a4 |  worker-1     |  10.240.1.21 |  3.91.234.180   |  running  |
|  i-04222701850127295 |  controller-1 |  10.240.1.11 |  52.201.245.168 |  running  |
|  i-0d610d55344cf7a09 |  worker-0     |  10.240.0.21 |  3.89.187.106   |  running  |
|  i-01e97df250fe43a10 |  controller-0 |  10.240.0.11 |  54.210.252.47  |  running  |
|  i-08eef4b1d65ae729e |  controller-2 |  10.240.2.11 |  18.206.191.182 |  running  |
+----------------------+---------------+--------------+-----------------+-----------+
```

## SSH Access

SSH will be used to configure the controller and worker instances. 

Test SSH access to the `worker-2` compute instances:

```
ssh -i "kubernetes-the-hard-way-key.pem" ec2-user@52.90.124.94
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
