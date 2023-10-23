# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl) and [helm](https://www.cncf.io/cncf-helm-project-journey/).


## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

### Linux

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64 \
  https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64
```

```
mv cfssl_1.6.4_linux_amd64 cfssl
mv cfssljson_1.6.4_linux_amd64 cfssljson

chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

### Other

Other installation options can be found [here](https://github.com/cloudflare/cfssl#installation)

### Verification

Verify `cfssl` and `cfssljson` version 1.6.4 or higher is installed:

```
cfssl version
```

> output

```
Version: 1.6.4
Runtime: go1.18
```

```
cfssljson --version
```

> output

```
Version: 1.6.4
Runtime: go1.18
```

## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

### Linux

```
wget https://dl.k8s.io/v1.28.1/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Other

Other installation options can be found [here](https://kubernetes.io/docs/tasks/tools/)

### Verification

Verify `kubectl` version 1.28.1 or higher is installed:

```
kubectl version --client
```

> output

```
Client Version: v1.28.1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

## Install HELM

Helm is the package manager for Kubernetes, and you can read detailed background information in the [CNCF Helm Project Journey report](https://www.cncf.io/cncf-helm-project-journey/).

### Linux

Helm now has an installer script that will automatically grab the latest version of Helm and install it locally.

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Other

Other installation options can be found [here](https://helm.sh/docs/intro/install/).

### Verification

```
helm version
```

> output

```
version.BuildInfo{Version:"v3.12.3", GitCommit:"3a31588ad33fe3b89af5a2a54ee1d25bfe6eaa5e", GitTreeState:"clean", GoVersion:"go1.20.7"}
```

Next: [Provisioning Compute Resources](03-compute-resources.md)
