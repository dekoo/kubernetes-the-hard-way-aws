# Prerequisites
## Amazon Web Services
This tutorial leverages the [Amazone Web Services](https://aws.amazon.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Set up](aws.amazon.com/free) an account, if you do not have it yet. 

[Estimated cost](https://calculator.aws/#/estimate?id=d571b2eb284d41ed814781289ea9a4fcf992779c) to run this tutorial: $0.2 per hour ($4.8 per day)
> The compute resources required for this tutorial exceed the Amazone Web Services free tier

## AWS Command Line Utility
### Install 
Follow the [Get started with the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) to install and configure `aws` command line utility.

Verify the AWS CLI version is 2.7.9 or higher:

```
aws --version
```

### Configure 

This example is for the long-term credentials from AWS Identity and Access Management. For more options, see [Configuring using AWS CLIE commands](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html#getting-started-quickstart-new-command)

```
aws configure
```

> output

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```

Next: [Installing the Client Tools](02-client-tools.md)
