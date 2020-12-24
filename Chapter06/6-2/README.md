# Setting up Kubernetes with kops
What is kops? It is the abbreviated term of Kubernetes Operation (https://github.com/kubernetes/kops). Similar to kubeadm, minikube, and kubespray, kops reduces the heavy
duty of building up a Kubernetes cluster by ourselves. It helps in creation, and provides an 
interface to users for managing the clusters. Furthermore, kops achieves a more automatic 
installing procedure and delivers a production-level system. It targets to support dominate 
cloud platforms, such as AWS, GCE, and VMware vSphere. In this recipe, we will talk 
about how to run a Kubernetes cluster with kops.

## Getting ready
Before our major tutorial, we will need to install kops on to your local host. It is a 
straightforward step for downloading the binary file and moving it to the system directory 
of the execution file:
```bash
// download the latest stable kops binary
$ curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
$ chmod +x kops-linux-amd64
$ sudo mv kops-linux-amd64 /usr/local/bin/kops
// verify the command is workable
$ kops version
Version 1.9.0 (git-cccd71e67)
```


Next, we have to prepare some AWS configuration on your host and required services for 
cluster. Refer to the following items and make sure that they are ready:

-  IAM user: Since kops would create and build several AWS service components
together for you, you must have an IAM user with kops required permissions.
We've created an IAM user named chap6 in the previous section that has the
following policies with the necessary permissions for kops:
   - AmazonEC2FullAccess
   - AmazonRoute53FullAccess
   - AmazonS3FullAccess
   - IAMFullAccess
   - AmazonVPCFullAccess  
   
Then, exposing the AWS access key ID and secret key as environment variables
can make this role applied on host while firing kops commands:
```bash
$ export AWS_ACCESS_KEY_ID=${string of 20 capital character combination}
$ export AWS_SECRET_ACCESS_KEY=${string of 40 character and number combination}
```

-  ***Prepare an S3 bucket for storing cluster configuration***: In our demonstration later, the S3 bucket name will be *kubernetes-cookbook*.
-  ***Prepare a Route53 DNS domain for accessing points of cluster***: In our demonstration later, the domain name we use will be *k8s-cookbook.net*.

## How to do it...
We can easily run up a Kubernetes cluster using a single command with parameters containing complete configurations. These parameters are described in the following table:  
| Parameter        | Description                                                                                                                                                                                                                    | Value in example            |
|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------|
| --name           | This is the name of the cluster. It will also be the domain name of the cluster's entry point. So you can utilize your Route53 DNS domain with a customized name, for example, {your cluster name}.{your Route53 domain name}. | my-cluster.k8s-cookbook.net |
| --state          | This indicates the S3 bucket that stores the status of the cluster in the format s3://{bucket name}.                                                                                                                           | s3://kubernetes-cookbook    |
| --zones          | This is the availability zone where you need to build your cluster.                                                                                                                                                            | us-east-1a                  |
| --cloud          | This is the cloud provider.                                                                                                                                                                                                    | aws                         |
| --network-cidr   | Here, kops helps to create independent CIDR range for the new VPC.                                                                                                                                                             | 10.0.0.0/16                 |
| --master-size    | This is the instance size of Kubernetes master.                                                                                                                                                                                | t2.large                    |
| --node-size      | This is the instance size of Kubernetes nodes.                                                                                                                                                                                 | t2.medium                   |
| --node-count     | This is the number of nodes in the cluster.                                                                                                                                                                                    | 2                           |
| --network        | This is the overlay network used in this cluster.                                                                                                                                                                              | calico                      |
| --topology       | This helps you decide whether the cluster is public facing.                                                                                                                                                                    | private                     |
| --ssh-public-key | This helps you assign an SSH public key for bastion server, then we may log in through the private key.                                                                                                                        | ~/.ssh/id_rsa.pub           |
| --bastion        | This gives you an indication to create the bastion server.                                                                                                                                                                     | N/A                         |
| --yes            | This gives you the confirmation for executing immediately.                                                                                                                                                                     | N/A                         |

Now we are ready to compose the configurations into a command and fire it:

```bash
$ kops create cluster --name my-cluster.k8s-cookbook.net --state=s3://kubernetes-cookbook --zones us-east-1a --cloud aws --networkcidr
10.0.0.0/16 --master-size t2.large --node-size t2.medium --node-count 2 --networking calico --topology private --ssh-public-key ~/.ssh/id_rsa.pub --bastion --yes

...
I0408 15:19:21.794035 13144 executor.go:91] Tasks: 105 done / 105 total;
0 can run
I0408 15:19:21.794111 13144 dns.go:153] Pre-creating DNS records
I0408 15:19:22.420077 13144 update_cluster.go:248] Exporting kubecfg for
cluster
kops has set your kubectl context to my-cluster.k8s-cookbook.net
Cluster is starting. It should be ready in a few minutes.
...
```

After a few minutes, the command takes out the preceding logs showing what AWS services have been created and served for you kops-built Kubernetes cluster. You can even check your AWS console to verify their relationships, which will look similar to the following diagram:

## How it works...
From localhost, users can interact with the cluster on AWS using the kops command:
```bash
//check the cluster
$ kops get cluster --state s3://kubernetes-cookbook

NAME                        CLOUD   ZONES
my-cluster.k8s-cookbook.net aws     us-east-1a
```

## Working with kops-built AWS cluster
Furthermore, as you can see in the previous section, the last few logs of kops cluster 
creation shows that the environment of the client is also ready. It means that kops helps to 
bind the API server to our host securely as well. We may use the kubectl command like 
we were in Kubernetes master. What we need to do is install kubectl manually. It would be 
as simple as installing kops; just download the binary file:
```bash
// install kubectl on local
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
$ chmod +x kubectl
$ sudo mv kubectl /usr/local/bin/
// check the nodes in cluster on AWS
$ kubectl get nodes
NAME                          STATUS  ROLES   AGE     VERSION
ip-10-0-39-216.ec2.internal   Ready   master  2m      v1.8.7
ip-10-0-40-26.ec2.internal    Ready   node    31s     v1.8.7
ip-10-0-50-147.ec2.internal   Ready   node    33s     v1.8.7
```
However, you can still access the nodes in the cluster. Since the cluster is set down in a private network, we will require to login to the bastion server first, and jump to the nodes
for the next:

```bash
//add private key to ssh authentication agent
$ ssh-add ~/.ssh/id_rsa

//use your private key with flag “-i”
//we avoid it since the private key is in default location, ~/.ssh/id_rsa
//also use -A option to forward an authentication agent
$ ssh -A admin@bastion.my-cluster.k8s-cookbook.net

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Apr 8 19:37:31 2018 from 10.0.2.167
// access the master node with its private IP
admin@ip-10-0-0-70:~$ ssh 10.0.39.216

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Apr 8 19:36:22 2018 from 10.0.0.70
admin@ip-10-0-39-216:~$
```

## Deleting kops-built AWS cluster
We can simply remove our cluster using the kops command as follows:
```bash
$ kops delete cluster --name my-cluster.k8s-cookbook.net --state s3://kubernetes-cookbook --yes

Deleted cluster: "my-cluster.k8s-cookbook.net"
```

It will clean the AWS services for you. But some other services created by yourself: S3 bucket, IAM role with powerful authorization, and Route53 domain name; kops will not
remove them on user's behavior. Remember to delete the no used AWS services on your side.
