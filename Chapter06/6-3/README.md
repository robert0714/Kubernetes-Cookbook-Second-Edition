# Using AWS as Kubernetes Cloud Provider
From Kubernetes 1.6, Cloud Controller Manager (CCM) was introduced, which defines a
set of interfaces so that different cloud providers could evolve their own implementations
out of the Kubernetes release cycle. Talking to the cloud providers, you can't ignore the
biggest player: Amazon Web Service. According to the Cloud Native Computing
Foundation, in 2017, 63% of Kubernetes workloads run on AWS. AWS CloudProvider
supports Service as ***Elastic Load Balancer (ELB)*** and ***Amazon Elastic Block Store (EBS)*** as
StorageClass.

At the time this book was written, Amazon Elastic Container Service for Kubernetes
(Amazon EKS) was under preview, which is a hosted Kubernetes service in AWS. Ideally,
it'll have better integration with Kubernetes, such as ***Application Load Balancer (ALB)*** for
Ingress, authorization, and networking. Currently in AWS, the limitation of routes per
route tables in VPC is 50; it could be up to 100 as requested. However, network
performance may be impacted if the routes exceed 50 according to the official
documentation of AWS. While kops uses kubenet networking by default, which allocates
a/24 CIDR to each node and configures the routes in route table in AWS VPC. This might
lead to the performance hit if the cluster has more than 50 nodes. Using a CNI network
could address this problem.

## Getting ready
For following along with the examples in this recipe, you'll need to create a Kubernetes
cluster in AWS. The following example is using kops to provision a Kubernetes cluster
named k8s-cookbook.net in AWS; as the preceding recipes show, set
$KOPS_STATE_STORE as a s3 bucket to store your kops configuration and metadata:

```bash
# kops create cluster --master-count 1 --node-count 2 --zones useast-1a,us-east-1b,us-east-1c --node-size t2.micro --master-size t2.small --topology private --networking calico --authorization=rbac --cloud-labels "Environment=dev" --state $KOPS_STATE_STORE --name k8s-cookbook.net 

I0408 16:10:12.212571 34744 create_cluster.go:1318] Using SSH public key: /Users/k8s/.ssh/id_rsa.pub I0408 16:10:13.959274 34744

create_cluster.go:472] Inferred --cloud=aws from zone "us-east-1a" I0408 16:10:14.418739 34744 subnets.go:184] Assigned CIDR 172.20.32.0/19 to subnet us-east-1a

I0408 16:10:14.418769 34744 subnets.go:184] Assigned CIDR 172.20.64.0/19 to subnet us-east-1b I0408 16:10:14.418777 34744 subnets.go:184] Assigned CIDR 172.20.96.0/19 to subnet us-east-1c
I0408 16:10:14.418785 34744 subnets.go:198] Assigned CIDR 172.20.0.0/22 to subnet utility-us-east-1a I0408 16:10:14.418793 34744 subnets.go:198] Assigned CIDR 172.20.4.0/22 to subnet utility-us-east-1b
I0408 16:10:14.418801 34744 subnets.go:198] Assigned CIDR 172.20.8.0/22 to subnet utility-us-east-1c ...

Finally configure your cluster with: kops update cluster k8s-cookbook.net --yes

```

Once we run the recommended kops update cluster <cluster_name> --yes command,
after a few minutes, the cluster is up and running. We can use the kops validate cluster to
check whether the cluster components are all up:

```bash
# kops validate cluster
Using cluster from kubectl context: k8s-cookbook.net
Validating cluster k8s-cookbook.net
INSTANCE GROUPS
NAME                ROLE      MACHINETYPE   MIN   MAX   SUBNETS
master-us-east-1a   Master    t2.small      1     1     us-east-1a
nodes               Node      t2.micro      2     2     us-east-1a,useast-1b,us-east-1c

NODE STATUS                       NAME ROLE     READY
ip-172-20-44-140.ec2.internal     node          True
ip-172-20-62-204.ec2.internal     master        True
ip-172-20-87-38.ec2.internal      node          True
Your cluster k8s-cookbook.net is ready
```
We're good to go!

## How to do it...
When running Kubernetes in AWS, there are two possible integrations we could use: ELB as Service with LoadBalancer Type and Amazon Elastic Block Store as StorageClass.

## Elastic load balancer as LoadBalancer service
Let's create a LoadBalancer Service with Pods underneath, which is what we learned in *Chapter 3, Playing with Containers*:
```bash
# cat aws-service.yaml 

```
