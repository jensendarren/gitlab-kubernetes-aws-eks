# Deploy an app to Kubernetets running on AWS EKS via GitLab AutoDevops

In order to achived this we need:

* A K8S cluster setup in AWS
* A Gitlab account
* Integratiom with Gitlab and the K8S cluster in AWS
* Helm Tiller, Ingress and GitLab Runner running in the K8S cluster (installed via Gitlab)
* A domian to test out the application (e.g. example.com)
* A wildcard DNS entry for the domain pointing to the ELB created during the Ingress installation.

### Setup a Kubernetes Cluster in AWS EKS

In order to easily setup a Kubernetes Cluster in [AWS EKS](https://aws.amazon.com/eks/) it's best to use command line tools. For this we will need to install:

* awscli
* eksctl
* kubectl

Follow the detailed instructions for [Getting Started with Amazon EKS with eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)

If you have multiple AWS profiles then you will need to pass the profile name to each command line, like so:

```
aws ec2 describe-instances --profile=rotati
```

Note if you want to avoid having to pass the profile name each time then set a local environment variable like so first:

```
export AWS_PROFILE=rotati

# Now this will use the rotati profile by default (notice this is also an example of using the query switch to filter the response)

aws ec2 describe-instances --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId]'
```

Read the following article for more details on [controlling AWS CLI output](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-output.html).

### Create the K8S Cluster

Once all the tools are installed and the versions are checked, run the following to create new K8S cluster in your current AWS account (the one where the `AWS_PROFILE` is set as per above).

```
eksctl create cluster \
--name prod \
--region ap-southeast-1 \
--version 1.13 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto
```

**NOTE: This may take about 10 minutes to complete!**. You can examine the progress by checking the CloudFormation execution.

For help on the options available when creating a cluster, check out the following:

```
eksctl create cluster --help
```

### Deploy the Kubernetes Web UI (Dashboard)

While its possible to use `kubectl` to monitor and manage the K8S cluster, sometimes it can be easier to get a quick overview of what is available via the Kubernetes Web UI (Dashboard).

Follow these instructions to [deploy and run the Kubernetes Web UI](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html) (Dashboard) to your cluster.

Once the dashbord is running, you need to get an auth token from the service, startup the proxy server and connect to the dashboard:

**Get the auth token**

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

**Start the proxy server**

```
kubectl proxy
```

**Log into the Kubernetes Dashboard**

Navigate to this location for the [running K8S Dashboard](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login) and copy past the token from the previous step to authenticate and gain access!

For more information about using the dashboard, see the [project documentation on GitHub](https://github.com/kubernetes/dashboard).