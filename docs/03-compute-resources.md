# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster within single [region](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.
> Some of the commands posted on this page are using AWS identifiers generated as a result of the preceding commands, hence pay attention on these and replace identifiers where necessary with your ones

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```
aws ec2 create-vpc --region us-east-1 --cidr-block 10.240.0.0/24 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=kubernetes-the-hard-way},{Key=Project,Value=kubernetes-the-hard-way}]' --query Vpc.VpcId --output text
```

Three [subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html) must be provisioned in each of three availability zones, with an IP address ranges large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```
aws ec2 create-subnet --availability-zone us-east-1a --vpc-id vpc-0ae4a5ac3edf51d2d --cidr-block 10.240.0.0/26 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=kubernetes-1a},{Key=Project,Value=kubernetes-the-hard-way}]' --query Subnet.SubnetId --output text
```
```
aws ec2 create-subnet --availability-zone us-east-1b --vpc-id vpc-0ae4a5ac3edf51d2d --cidr-block 10.240.0.64/26 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=kubernetes-1b},{Key=Project,Value=kubernetes-the-hard-way}]' --query Subnet.SubnetId --output text
```
```	
aws ec2 create-subnet --availability-zone us-east-1c --vpc-id vpc-0ae4a5ac3edf51d2d --cidr-block 10.240.0.128/26 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=kubernetes-1c},{Key=Project,Value=kubernetes-the-hard-way}]' --query Subnet.SubnetId --output text
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Network ACL are created automatically together with VPC, therefore first we shell find the one associated with ours by listing them all and searching by VPC ID:

```
aws ec2 describe-network-acls --output text | grep vpc-0ae4a5ac3edf51d2d
```

Update tags on the NACL that was found on the previous step:

```
aws ec2 create-tags --resources acl-0edf6f4c8be481a2f --tags Key=Name,Value=kubernetes-the-hard-way-nacl
```
```
aws ec2 create-tags --resources acl-0edf6f4c8be481a2f --tags Key=Project,Value=kubernetes-the-hard-way
```

Delete default inbound rule that allows all traffic:

```
aws ec2 delete-network-acl-entry --network-acl-id acl-0edf6f4c8be481a2f --ingress --rule-number 100
```

Add rules that allow SSH, ICMP and HTTPS inbound traffic:

```
aws ec2 create-network-acl-entry --ingress --network-acl-id acl-0edf6f4c8be481a2f --rule-action allow --rule-number 100 --cidr-block 0.0.0.0/0 --protocol 6 --port-range 'From=22,To=22'
```
```
aws ec2 create-network-acl-entry --ingress --network-acl-id acl-0edf6f4c8be481a2f --rule-action allow --rule-number 200 --cidr-block 0.0.0.0/0 --protocol 6 --port-range 'From=6443,To=6443'
```
```
aws ec2 create-network-acl-entry --ingress --network-acl-id acl-0edf6f4c8be481a2f --rule-action allow --rule-number 300 --cidr-block 0.0.0.0/0 --protocol 1 --icmp-type-code 'Code=-1,Type=-1'
```

> An [elastic load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) will be used to expose the Kubernetes API Servers to remote clients.

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
aws ec2 allocate-address --region=us-east-1 --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=kubernetes-the-hard-way-ip},{Key=Project,Value=kubernetes-the-hard-way}]'
```

### Internet Gateway
In order to allow access via SSH we need to create Internet Gateway and attached it to our VPC:

```
aws ec2 create-internet-gateway --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=kubernetes-the-hard-way-igw},{Key=Project,Value=kubernetes-the-hard-way}]' --output text
```
```
aws ec2 attach-internet-gateway --internet-gateway-id igw-017fe81aa84a413ed --vpc-id vpc-0ae4a5ac3edf51d2d
```

Then configure routing tables to allow outbound SSH traffic to "0.0.0.0/0" via Internet Gateway.
First get routing table that was created by default together with VPC on the previous steps:

```
aws ec2 describe-route-tables --filters 'Name=vpc-id,Values=vpc-0ae4a5ac3edf51d2d' --output text
```
	
Now add the rule to the table:

```
aws ec2 create-route --route-table-id rtb-0958c5f1596132a09 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-017fe81aa84a413ed --output text
```

## Compute Instances

The compute instances in this lab will be provisioned using [Amazon Linux](https://aws.amazon.com/amazon-linux-2/?amazon-linux-whats-new.sort-by=item.additionalFields.postDateTime&amazon-linux-whats-new.sort-order=desc), which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address within respective subnet to simplify the Kubernetes bootstrapping process.

### Key Pair

Create key pair that will be used later to connect to the instances via SSH:

```
aws ec2 create-key-pair --key-name kubernetes-the-hard-way-key --output text
```
	
> Save key-material from the output in the `kubernetes-the-hard-way-key.pem` file, i.e. text between below mentioned "BEGIN", "END" sections inclusive those:
```
	-----BEGIN RSA PRIVATE KEY-----
				…
	-----END RSA PRIVATE KEY-----
```

Adjust permissions on the file: 

```
chmod 400 kubernetes-the-hard-way-key.pem
```

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

> Note that private IP address must belong to the selected subnet which in turn must correspond to one of the availability zones (AZ) in a way to ensure even spread of controllers instances across three AZ.

```
aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 --instance-type t2.micro --key-name kubernetes-the-hard-way-key --subnet-id subnet-014744e367f4ac4f1 --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=controller-1},{Key=Project,Value=kubernetes-the-hard-way}]' --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' --region us-east-1 --private-ip-address 10.240.0.11 --output text --query "Instances[0].InstanceId"
```
```
aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 --instance-type t2.micro --key-name kubernetes-the-hard-way-key --subnet-id subnet-097e7f895f6ff520c --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=controller-2},{Key=Project,Value=kubernetes-the-hard-way}]' --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' --region us-east-1 --private-ip-address 10.240.0.75 --output text --query "Instances[0].InstanceId"
```
```
aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 --instance-type t2.micro --key-name kubernetes-the-hard-way-key --subnet-id subnet-0bbf466abf833ad3b --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=controller-3},{Key=Project,Value=kubernetes-the-hard-way}]' --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' --region us-east-1 --private-ip-address 10.240.0.139 --output text --query "Instances[0].InstanceId"
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod_cidr` environment variable set via [user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 --instance-type t2.micro --key-name kubernetes-the-hard-way-key --subnet-id subnet-014744e367f4ac4f1 --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-1},{Key=Project,Value=kubernetes-the-hard-way}]' --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' --region us-east-1 --private-ip-address 10.240.0.21 --output text --query "Instances[0].InstanceId" --user-data "$(printf '#cloud-config\n\nruncmd:\n - echo \"%s\" >> /etc/bashrc' 'export pod_cidr=10.200.1.0/24')"
```
```
aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 --instance-type t2.micro --key-name kubernetes-the-hard-way-key --subnet-id subnet-097e7f895f6ff520c --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-2},{Key=Project,Value=kubernetes-the-hard-way}]' --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' --region us-east-1 --private-ip-address 10.240.0.85 --output text --query "Instances[0].InstanceId" --user-data "$(printf '#cloud-config\n\nruncmd:\n - echo \"%s\" >> /etc/bashrc' 'export pod_cidr=10.200.2.0/24')"
```
```
aws ec2 run-instances --image-id ami-0f34c5ae932e6f0e4 --instance-type t2.micro --key-name kubernetes-the-hard-way-key --subnet-id subnet-0bbf466abf833ad3b --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-3},{Key=Project,Value=kubernetes-the-hard-way}]' --instance-market-options 'MarketType=spot,SpotOptions={MaxPrice=0.3,SpotInstanceType=one-time,InstanceInterruptionBehavior=terminate}' --associate-public-ip-address --block-device-mapping 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,VolumeSize=30,VolumeType=gp3}' --region us-east-1 --private-ip-address 10.240.0.149 --output text --query "Instances[0].InstanceId" --user-data "$(printf '#cloud-config\n\nruncmd:\n - echo \"%s\" >> /etc/bashrc' 'export pod_cidr=10.200.3.0/24')"
```

### Verification

List the compute instances in your default region:

```
aws ec2 describe-instances --output text  --query "Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key=='Name'].Value | [0],PublicIP:PublicIpAddress,Status:State.Name}"
```

> output

```
i-09412985db7c72158		controller-2		54.90.196.141		running
i-0381ccbd1b41ba93f		worker-2			23.20.192.99		running
i-0f9b26279f0a9ce11		worker-1			44.212.25.155		running
i-095ad38d5362b6273		controller-1		54.224.51.211		running
i-05de50d771f987162		worker-3			3.87.115.200		running
i-01b852f450b8ff73e		controller-3		3.93.0.159			running
```

## SSH Access

SSH will be used to configure the controller and worker instances. 

Test SSH access to the `worker-2` compute instances:

```
ssh -i "kubernetes-the-hard-way-key.pem" ec2-user@23.20.192.99
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
