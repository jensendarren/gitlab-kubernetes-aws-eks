# Deploy an app to Kubernetes running on AWS EKS via GitLab AutoDevOps

In order to achive this we need:

* A K8S cluster setup in AWS
* A Gitlab account
* Integratiom with Gitlab and the K8S cluster in AWS
* Helm Tiller, Ingress and GitLab Runner running in the K8S cluster (installed via Gitlab)
* A domian to test out the application (e.g. example.com)
* A wildcard DNS entry for the domain pointing to the ELB created during the Ingress installation.

### Project in Gitlab

Here is the [deploy-to-eks project in Gitlab](https://gitlab.com/jensendarren/deploy-to-eks). Note that the K8S cluster may have been removed to save money. The below instructions can be used to recreate a new cluster to try this out anytime.

## Setup a Kubernetes Cluster in AWS EKS

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

# Now, in the same terminal session, this will use the rotati profile by default:

aws ec2 describe-instances
```

### AWS CLI Query Param

You can query / filter results from the AWS CLI tool using `query` parameter, like so:

```
# Fetch only AvailabilityZone, Name and InstanceId from EC2 Describe Instances
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId]'

# Fetch only the cluster endpoint from EKS Descript Cluster
aws eks describe-cluster --name=prod --query 'cluster.endpoint'
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

A reminder, make sure that you have set your AWS_PROFILE to the account where the k8s cluster is deoployed using `export AWS_PROFILE=rotati` if you choose to run this command in a new terminal window (like I did). If you do not set that, then you are not effectivly running this command against the correct account and you will receive a `401 Error` when attempting to open the Dashboard in your browser!

```
kubectl proxy
```

**Log into the Kubernetes Dashboard**

Navigate to this location for the [running K8S Dashboard](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login) and copy past the token from the previous step to authenticate and gain access!

For more information about using the dashboard, see the [project documentation on GitHub](https://github.com/kubernetes/dashboard).


## Setup GitLab: Connecting and deploying to an Amazon EKS cluster

Create a new GitLab account (if you need to) and then follow these instructions for [connecting and deploying to an Amazon EKS cluster](https://docs.gitlab.com/ee/user/project/clusters/eks_and_gitlab/).

This uses a templated Rails application to test the auto devops process to the running K8S cluster created previously in AWS.

Once you have created the project you can create the integration with the K8S cluster. You will need:

* **Kubernetes cluster name**: Provide a name for the cluster to identify it within GitLab. For now just call it **eks**.
* **Environment scope**: Leave this as * for now, since we are only connecting a single cluster.
* **API URL**: Paste in the API server endpoint. You can fetch this with the following command `aws eks describe-cluster --name=prod --query 'cluster.endpoint'`
* **CA Certificate**: Paste the certificate data from the earlier step (see gitlab doc linked above)
* Paste the **admin token** value (again, see gitlab doc linked above)
* **Project namespace**: This can be left blank to accept the default namespace, based on the project name.
* Make sure you **DISABLE** _RBAC-enabled cluster_ option for now.
* MAke sure you **ENABLE** _GitLab-managed cluster_ option for now.
* Set the **base domain** to the domain that you want to use to test this app (e.g. example.com)

In order to ensure that RBAC being disabled will not cause issues run this command:

```
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```

## Deploy services to the cluster

Now we need to deploy some Gitlab Managed Services to the cluster to enable deployment and auto devops to work. We need to install:

* Helm Tiller
* Ingress
* GitLab Runner

### Instlaling Helm Tiller

Just click the 'install' button next to Helm Tiller.

### Instlaling Ingress

Just click the 'install' button next to Ingress. Once the installation is complete then make a note of the 'Ingress Endpoint' for later. This is essentially the public ELB created in AWS that points to all the nodes in our cluster!

### Instlaling Gitlab Runner

Just click the 'install' button next to Gitlab Runner.

Note that the Storage Class should already be applied.

### Setting the DNS for the application domain

Basically, we need to set a wildcard DNS entry that points to the ELB endpoint.

For example, if your domain is `example.com` and your ELB DNS Name is `a05338436cbd711e98e8a027800492e8-2140553012.ap-southeast-1.elb.amazonaws.com` then you need to add a DNS entry (using Route 53 for example) as follows:

*.example.com.  A   ALIAS dualstack.a05338436cbd711e98e8a027800492e8-2140553012.ap-southeast-1.elb.amazonaws.com.

**NOTE**: its a wildcard subdomain A record entry pointing to an ALIAS of the ELB with a 'dualstack' prefix!

Read the details on [how to add this entry in Route 53](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/using-domain-names-with-elb.html?icmpid=docs_elb_console#dns-associate-custom-elb).

### Troubleshooting

Sometimes the install may freeze or not be successful. Best approach here is to resort to the `kubectl` tool to see what might be causeing this and if necessary delete the entire **deployment** then delete the K8S integration in Gitlab and start again!

Remember also, if you open a new terminal window, remember to set your AWS_PROFILE correctly `export AWS_PROFILE=rotati` otherwise you will be interacting with a different AWS account where your cluster is deployed to. Of course, if you are using your 'default' AWS account or you have set AWS_PROFILE as part of your `bash_profile` then this will note be an issue!

### Deploy to EKS

Deployment should now be as simple as kicking off a build either via a new commit or manually via the Gitlab UI under CI/CD. As long as the project is enabled to use AutoDevOps this should just work!

Check out these details for further information to [deploy your app to EKS](https://docs.gitlab.com/ee/user/project/clusters/eks_and_gitlab/#deploy-the-app-to-eks).

## Scaling

If you want to manually scale the number of nodes in the cluster, use the following command:

```
# This will scale the number of desired nodes to 1 in the cluster 'prod' for 'standard-workers' nodegroup
eksctl scale nodegroup --nodes=1 --cluster=prod --name=standard-workers
```

## Deleting

To delete a cluster run the following command:

```
# delete the nodegroup first
eksctl delete nodegroup standard-workers --cluster prod

# then delete the entire cluster
eksctl delete cluster prod
```