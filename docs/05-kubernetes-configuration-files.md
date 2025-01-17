# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user.

### Kubernetes Public DNS

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the DNS address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the `kubernetes-the-hard-way-ip` static IP address:

```
PUBLIC_API_DNS=$(aws elbv2 describe-load-balancers \
  --names kubernetes-hard-way-nlb --output text --query LoadBalancers[].DNSName)
```

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/).

> The following commands must be run in the same directory used to generate the SSL certificates during the [Generating TLS Certificates](04-certificate-authority.md) lab.

Generate a kubeconfig file for each worker node:

```
for instance in worker-0 worker-1 worker-2; do
  INTERNAL_DNS=$(aws ec2 describe-instances --output text \
      --query "Reservations[*].Instances[*].{PrivateDnsName:PrivateDnsName}" \
      --filters "Name=instance-state-name,Values=running" "Name=tag:Name,Values=${instance}")

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${PUBLIC_API_DNS}:6443 \
    --kubeconfig=${INTERNAL_DNS}.kubeconfig

  kubectl config set-credentials system:node:${INTERNAL_DNS} \
    --client-certificate=${INTERNAL_DNS}.pem \
    --client-key=${INTERNAL_DNS}-key.pem \
    --embed-certs=true \
    --kubeconfig=${INTERNAL_DNS}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${INTERNAL_DNS} \
    --kubeconfig=${INTERNAL_DNS}.kubeconfig

  kubectl config use-context default --kubeconfig=${INTERNAL_DNS}.kubeconfig
done
```

Results:

```
ip-10-240-0-21.ec2.internal.kubeconfig
ip-10-240-1-21.ec2.internal.kubeconfig
ip-10-240-2-21.ec2.internal.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${PUBLIC_API_DNS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

```

Results:

```
kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

```

Results:

```
kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

```

Results:

```
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
```

> This one is to be used on controller instances, addressing API server by `127.0.0.1`. Similar `admin` kubeconfig to access API server from your local machine will be created in [Configuring kubectl for Remote Access](10-configuring-kubectl.md) lab

Results:

```
admin.kubeconfig
```


## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```
for instance in worker-0 worker-1 worker-2; do

  INTERNAL_DNS=$(aws ec2 describe-instances --output text \
      --query "Reservations[*].Instances[*].{PrivateDnsName:PrivateDnsName}" \
      --filters "Name=instance-state-name,Values=running" "Name=tag:Name,Values=${instance}")

  scp -i "kubernetes-the-hard-way-key.pem" ${INTERNAL_DNS}.kubeconfig kube-proxy.kubeconfig \
    ec2-user@$(aws ec2 describe-instances --output text \
      --query "Reservations[*].Instances[*].{PublicIP:PublicIpAddress}" \
      --filters "Name=instance-state-name,Values=running" "Name=tag:Name,Values=${instance}"):~/

done
```

Copy the appropriate `admin`, `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```
for instance in controller-0 controller-1 controller-2; do
  scp -i "kubernetes-the-hard-way-key.pem" admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig \
    ec2-user@$(aws ec2 describe-instances --output text \
      --query "Reservations[*].Instances[*].{PublicIP:PublicIpAddress}" \
      --filters "Name=instance-state-name,Values=running" "Name=tag:Name,Values=${instance}"):~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
