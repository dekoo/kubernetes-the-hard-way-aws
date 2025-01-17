# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the DNS name assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
{
  PUBLIC_API_DNS=$(aws elbv2 describe-load-balancers \
  --names kubernetes-hard-way-nlb --output text --query LoadBalancers[].DNSName)

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${PUBLIC_API_DNS}:6443 \

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
	--embed-certs=true \

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin 

  kubectl config use-context kubernetes-the-hard-way 
}
```

## Verification

Check the version of the remote Kubernetes cluster:

```
kubectl version
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes  
```

> output

```
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-240-0-21.ec2.internal   Ready    <none>   33m   v1.28.1
ip-10-240-1-21.ec2.internal   Ready    <none>   33m   v1.28.1
ip-10-240-2-21.ec2.internal   Ready    <none>   33m   v1.28.1
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
