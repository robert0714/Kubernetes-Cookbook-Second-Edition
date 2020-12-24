# Chapter 6 Building Kubernetes on AWS

## How it works...
Let's explorer AWS to launch a typical infrastructure. Using awscli to build your own VPC, 
Subnet, Gateway, and Security group. Then, launch the EC2 instance to understand the 
basic usage of AWS.

### Creating VPC and Subnets
Virtual Private Cloud (VPC) is a Software-Defined Network. You can configure a virtual network on AWS. Subnets are inside of VPC that define network block (***Classless Inter Domain Routing (CIDR)***) such as 192.168.1.0/24.

Let's create one VPC and two subnets using the following steps:

1. Create a new VPC that has 192.168.0.0/16 CIDR block (IP range:192.168.0.0 – 192.168.255.255). Then, capture VpcId:

```bash
$ aws ec2 create-vpc --cidr-block 192.168.0.0/16  
---------------------------------------------------------
{
    "Vpc": {
        "CidrBlock": "192.168.0.0/16",
        "DhcpOptionsId": "dopt-aa9abacd",
        "State": "pending",
        "VpcId": "vpc-05ebce82711fe6a19",
        "OwnerId": "662260156742",
        "InstanceTenancy": "default",
        "Ipv6CidrBlockAssociationSet": [],
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-0e7ff01db2153c59c",
                "CidrBlock": "192.168.0.0/16",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false
    }
}


```

2. Create the first subnet under the VPC (***vpc-05ebce82711fe6a19***) that has 192.168.0.0/24 CIDR block (IP range: 192.168.0.0 – 192.168.0.255) and specify the availability zone as us-east-1a. Then, capture SubnetId:
```bash
$ aws ec2 create-subnet --vpc-id vpc-05ebce82711fe6a19 --cidr-block 192.168.0.0/24 --availability-zone us-east-1a
--------------------------------------------------------------------------------------------
{
    "Subnet": {
        "AvailabilityZone": "us-east-1a",
        "AvailabilityZoneId": "use1-az4",
        "AvailableIpAddressCount": 251,
        "CidrBlock": "192.168.0.0/24",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "available",
        "SubnetId": "subnet-03f6120aec2780686",
        "VpcId": "vpc-05ebce82711fe6a19",
        "OwnerId": "662260156742",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:us-east-1:662260156742:subnet/subnet-03f6120aec2780686"
    }
}

```

3. Create the second subnet on us-east-1b, which has 192.168.1.0/24 CIDR block (IP range: 192.168.1.0 – 192.168.1.255). Then, capture SubnetId:
```bash
$ aws ec2 create-subnet --vpc-id vpc-05ebce82711fe6a19 --cidr-block  192.168.1.0/24 --availability-zone us-east-1b
--------------------------------------------------------------------------------------------------------------------
{
    "Subnet": {
        "AvailabilityZone": "us-east-1b",
        "AvailabilityZoneId": "use1-az6",
        "AvailableIpAddressCount": 251,
        "CidrBlock": "192.168.1.0/24",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "available",
        "SubnetId": "subnet-0a94a1e951e6a151c",
        "VpcId": "vpc-05ebce82711fe6a19",
        "OwnerId": "662260156742",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:us-east-1:662260156742:subnet/subnet-0a94a1e951e6a151c"
    }
}

```
4. Check the subnet list under VPC (*vpc-05ebce82711fe6a19*) using the following command:
```
$ aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-05ebce82711fe6a19" --query "Subnets[*].{Vpc:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone,Id:SubnetId}" --output=table
---------------------------------------------------------------
---------------------------------------------------------------------------------------
|                                   DescribeSubnets                                   |
+------------+-----------------+----------------------------+-------------------------+
|     AZ     |      CIDR       |            Id              |           Vpc           |
+------------+-----------------+----------------------------+-------------------------+
|  us-east-1b|  192.168.1.0/24 |  subnet-0a94a1e951e6a151c  |  vpc-05ebce82711fe6a19  |
|  us-east-1a|  192.168.0.0/24 |  subnet-03f6120aec2780686  |  vpc-05ebce82711fe6a19  |
+------------+-----------------+----------------------------+-------------------------+
```
This looks good!

##　Internet gateway
To access your VPC network, you need to have a gateway that accesses it from the internet.
***Internet Gateway (IGW)*** is the one that connects the internet to your VPC.

Then, in the subnets under VPC, you can set the default route to go to IGW or not. If  
it routes to IGW, the subnet is classified as the public subnet. Then, you can assign the  
global IP address on the public subnet.

Let's configure the first subnet (*192.168.0.0/24*) as the public subnet that routes to IGW 
using the following steps:  
1. Create IGW and capture InternetGatewayId:  
```bash
$ aws ec2 create-internet-gateway
-------------------------------------------
{
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-09e40d0fd3e3dd82f",
        "OwnerId": "662260156742",
        "Tags": []
    }
}
```

2. Attach IGW (igw-09e40d0fd3e3dd82f) to your VPC (vpc-05ebce82711fe6a19):
```bash
$ aws ec2 attach-internet-gateway --vpc-id vpc-05ebce82711fe6a19 --internet-gateway-id igw-09e40d0fd3e3dd82f
```
3. Create a routing table on  VPC (vpc-05ebce82711fe6a19) and then capture RouteTableId:
```bash
$ aws ec2 create-route-table --vpc-id vpc-05ebce82711fe6a19
-------------------------------------------------------------
{
    "RouteTable": {
        "Associations": [],
        "PropagatingVgws": [],
        "RouteTableId": "rtb-0a652ba4b877d559b",
        "Routes": [
            {
                "DestinationCidrBlock": "192.168.0.0/16",
                "GatewayId": "local",
                "Origin": "CreateRouteTable",
                "State": "active"
            }
        ],
        "Tags": [],
        "VpcId": "vpc-05ebce82711fe6a19",
        "OwnerId": "662260156742"
    }
}

```
4. Set the default route (0.0.0.0/0) for route table (rtb-0a652ba4b877d559b) as IGW (igw-09e40d0fd3e3dd82f):

```bash
$  aws ec2 create-route --route-table-id rtb-0a652ba4b877d559b --gateway-id  igw-09e40d0fd3e3dd82f --destination-cidr-block 0.0.0.0/0
-------------------------------
{
    "Return": true
}
```

5. Associate route table (rtb-0a652ba4b877d559b) to public subnet (subnet-0a94a1e951e6a151c ):
```bash
$ aws ec2 associate-route-table --route-table-id rtb-0a652ba4b877d559b --subnet-id subnet-0a94a1e951e6a151c 
-----------------------------------------
{
    "AssociationId": "rtbassoc-0d0c168cef9705f95",
    "AssociationState": {
        "State": "associated"
    }
}
```
6. Enable autoassign public IP on the public subnet ( subnet-0a94a1e951e6a151c):
```bash
$ aws ec2 modify-subnet-attribute --subnet-id  subnet-0a94a1e951e6a151c --map-public-ip-on-launch
```

## NAT-GW
What happens if the subnet default route is not pointing to IGW? The subnet is classified as 
a private subnet with no connectivity to the internet. However, some of situation, your VM 
in private subnet needs to access to the Internet. For example, download some security 
patch.

In this case, you can setup NAT-GW. It allows you access to the internet from the private 
subnet. However, it allows outgoing traffic only, so you cannot assign public IP address for 
a private subnet. Therefore, it is suitable for backend instances, such as the database.

Let's create NAT-GW and configure a second subnet (192.168.1.0/24) as a private 
subnet that routes to NAT-GW using the following steps:   

1. NAT-GW needs a Global IP address, so create Elastic IP (EIP):
```bash
$ aws ec2 allocate-address
---------------------------
{
    "PublicIp": "54.146.72.113",
    "AllocationId": "eipalloc-0717c5dd1d5d04e74",
    "PublicIpv4Pool": "amazon",
    "NetworkBorderGroup": "us-east-1",
    "Domain": "vpc"
}

```
2. Create NAT-GW on the public subnet ( subnet-0a94a1e951e6a151c) and assign EIP (eipalloc-0717c5dd1d5d04e74). Then, capture NatGatewayId.

ps. Since NAT-GW needs to access the internet, it must be located on the public subnet instead of the private subnet.

Input the following command:
```bash
$ aws ec2 create-nat-gateway --subnet-id  subnet-0a94a1e951e6a151c --allocation-id eipalloc-0717c5dd1d5d04e74
---------------------------------------------------------------
{
    "ClientToken": "f45bbfd6-5b55-4cd0-965e-ae2111949958",
    "NatGateway": {
        "CreateTime": "2020-12-24T08:24:59+00:00",
        "NatGatewayAddresses": [
            {
                "AllocationId": "eipalloc-0717c5dd1d5d04e74"
            }
        ],
        "NatGatewayId": "nat-0de64c21a344b6b10",
        "State": "pending",
        "SubnetId": "subnet-0a94a1e951e6a151c",
        "VpcId": "vpc-05ebce82711fe6a19"
    }
}
```
3. Create the route table and capture RouteTableId:
```bash
$ aws ec2 create-route-table --vpc-id vpc-05ebce82711fe6a19
--------------------------------------------------------------------
{
    "RouteTable": {
        "Associations": [],
        "PropagatingVgws": [],
        "RouteTableId": "rtb-05120bc24a41e57a1",
        "Routes": [
            {
                "DestinationCidrBlock": "192.168.0.0/16",
                "GatewayId": "local",
                "Origin": "CreateRouteTable",
                "State": "active"
            }
        ],
        "Tags": [],
        "VpcId": "vpc-05ebce82711fe6a19",
        "OwnerId": "662260156742"
    }
}
```
4. Set the default route (0.0.0.0/0) of the route table (rtb-05120bc24a41e57a1) to NATGW (nat-0de64c21a344b6b10):
```bash
$ aws ec2 create-route --route-table-id rtb-05120bc24a41e57a1 --nat-gateway-id nat-0de64c21a344b6b10 --destination-cidr-block 0.0.0.0/0
{
    "Return": true
}

```
5. Associate route table (rtb-05120bc24a41e57a1) to private subnet (subnet-03f6120aec2780686):
```bash
$ aws ec2 associate-route-table --route-table-id rtb-05120bc24a41e57a1 --subnet-id subnet-03f6120aec2780686
---------------------------------------------------------------------------------------------------------
{
    "AssociationId": "rtbassoc-0f7d3808e89d2a46d",
    "AssociationState": {
        "State": "associated"
    }
}

```
## Security group
Before launching your Virtual Server (EC2), you need to create a Security Group that has an 
appropriate security rule. Now, we have two subnets, public and private. Let's set public 
subnet such that it allows ssh (22/tcp) and http (80/tcp) from the internet. Then, set the 
private subnet such that it allows ssh from the public subnet: 

1. Create one security group for the public subnet on VPC (vpc-05ebce82711fe6a19):
```bash
$ aws ec2 create-security-group --vpc-id vpc-05ebce82711fe6a19 --group-name public --description "public facing host"
---------------------------------------------
{
    "GroupId": "sg-0d3891ab3d2e782ce"
}
```
2. Add the ssh allow rule to the public security group (sg-0d3891ab3d2e782ce):
```bash
$ aws ec2 authorize-security-group-ingress --group-id sg-0d3891ab3d2e782ce  --protocol tcp --port 22 --cidr 0.0.0.0/0
```
3. Add the http allow rule to the public security group ( sg-0d3891ab3d2e782ce ):
```bash
$ aws ec2 authorize-security-group-ingress --group-id sg-0d3891ab3d2e782ce  --protocol tcp --port 80 --cidr 0.0.0.0/0
```
4. Create a second security group for the private subnet on VPC (vpc-05ebce82711fe6a19):
```bash
$ aws ec2 create-security-group --vpc-id vpc-05ebce82711fe6a19 --group-name private --description "private subnet host"
---------------------------------------------
{
    "GroupId": "sg-046fe855b60b9bd49"
}
```
5. Add an ssh allow rule to the private security group (sg-046fe855b60b9bd49):
```bash
$ aws ec2 authorize-security-group-ingress --group-id sg-046fe855b60b9bd49   --protocol tcp --port 22 --source-group sg-0d3891ab3d2e782ce
```
6. Check the Security Group list using the following command:
```bash
$ aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-05ebce82711fe6a19" --query "SecurityGroups[*].{id:GroupId,name:GroupName}" --output table
---------------------------------------------------------
-------------------------------------
|      DescribeSecurityGroups       |
+-----------------------+-----------+
|          id           |   name    |
+-----------------------+-----------+
|  sg-046fe855b60b9bd49 |  private  |
|  sg-0d3891ab3d2e782ce |  public   |
|  sg-0e746f70d741c4113 |  default  |
+-----------------------+-----------+

```

## EC2
Now you need to upload your ssh public key and then launch the EC2 instance on both the public subnet and the private subnet:  
1. Upload your ssh public key (assume you have a public key that is located at *~/.ssh/id_rsa.pub* ):
```bash
$aws ec2 import-key-pair --key-name=chap6-key --public-key-material "`cat ~/.ssh/id_rsa.pub`"
```

2. Launch the first EC2 instance with the following parameters:

*  Use Amazon Linux image: ami-1853ac65 (Amazon Linux)
*  T2.nano instance type: t2.nano
*  Ssh key: chap6-key
*  Public Subnet: subnet-0a94a1e951e6a151c
*  Public Security Group: sg-0e746f70d741c4113
```bash
$ aws ec2 run-instances --image-id ami-1853ac65 --instance-type t2.nano --key-name chap6-key --security-group-ids sg-0e746f70d741c4113 --subnet-id subnet-0a94a1e951e6a151c
--------------------------------------------------
{
    "Groups": [],
    "Instances": [
        {
            "AmiLaunchIndex": 0,
            "ImageId": "ami-1853ac65",
            "InstanceId": "i-0e9f5c78a1df96534",
            "InstanceType": "t2.nano",
            "KeyName": "chap6-key",
            "LaunchTime": "2020-12-24T08:56:59+00:00",
            "Monitoring": {
                "State": "disabled"
            },
            "Placement": {
                "AvailabilityZone": "us-east-1b",
                "GroupName": "",
                "Tenancy": "default"
            },
            "PrivateDnsName": "ip-192-168-1-213.ec2.internal",
            "PrivateIpAddress": "192.168.1.213",
            "ProductCodes": [],
            "PublicDnsName": "",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "StateTransitionReason": "",
            "SubnetId": "subnet-0a94a1e951e6a151c",
            "VpcId": "vpc-05ebce82711fe6a19",
            "Architecture": "x86_64",
            "BlockDeviceMappings": [],
            "ClientToken": "4c39da43-bd9c-4b63-8fca-d56cc4317a9c",
            "EbsOptimized": false,
            "EnaSupport": true,
            "Hypervisor": "xen",
            "NetworkInterfaces": [
                {
                    "Attachment": {
                        "AttachTime": "2020-12-24T08:56:59+00:00",
                        "AttachmentId": "eni-attach-0b184c004d77ae807",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attaching",
                        "NetworkCardIndex": 0
                    },
                    "Description": "",
                    "Groups": [
                        {
                            "GroupName": "default",
                            "GroupId": "sg-0e746f70d741c4113"
                        }
                    ],
                    "Ipv6Addresses": [],
                    "MacAddress": "0e:90:4d:05:03:f5",
                    "NetworkInterfaceId": "eni-02b4910f076245c15",
                    "OwnerId": "662260156742",
                    "PrivateIpAddress": "192.168.1.213",
                    "PrivateIpAddresses": [
                        {
                            "Primary": true,
                            "PrivateIpAddress": "192.168.1.213"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Status": "in-use",
                    "SubnetId": "subnet-0a94a1e951e6a151c",
                    "VpcId": "vpc-05ebce82711fe6a19",
                    "InterfaceType": "interface"
                }
            ],
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SecurityGroups": [
                {
                    "GroupName": "default",
                    "GroupId": "sg-0e746f70d741c4113"
                }
            ],
            "SourceDestCheck": true,
            "StateReason": {
                "Code": "pending",
                "Message": "pending"
            },
            "VirtualizationType": "hvm",
            "CpuOptions": {
                "CoreCount": 1,
                "ThreadsPerCore": 1
            },
            "CapacityReservationSpecification": {
                "CapacityReservationPreference": "open"
            },
            "MetadataOptions": {
                "State": "pending",
                "HttpTokens": "optional",
                "HttpPutResponseHopLimit": 1,
                "HttpEndpoint": "enabled"
            },
            "EnclaveOptions": {
                "Enabled": false
            }
        }
    ],
    "OwnerId": "662260156742",
    "ReservationId": "r-0f99b4af64fb7402f"
}
```
3. Launch the second EC2 instance with the following parameters:
*  Use Amazon Linux image: ami-1853ac65
*  T2.nano instance type: t2.nano
*  Ssh key: chap6-key
*  Private subnet: subnet-03f6120aec2780686
*  Private Secuity Group: sg-046fe855b60b9bd49
```
$ aws ec2 run-instances --image-id ami-1853ac65 --instance-type t2.nano --key-name chap6-key --security-group-ids sg-046fe855b60b9bd49 --subnet-id  subnet-03f6120aec2780686
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
{
    "Groups": [],
    "Instances": [
        {
            "AmiLaunchIndex": 0,
            "ImageId": "ami-1853ac65",
            "InstanceId": "i-0b029d431795edb40",
            "InstanceType": "t2.nano",
            "KeyName": "chap6-key",
            "LaunchTime": "2020-12-24T09:00:39+00:00",
            "Monitoring": {
                "State": "disabled"
            },
            "Placement": {
                "AvailabilityZone": "us-east-1a",
                "GroupName": "",
                "Tenancy": "default"
            },
            "PrivateDnsName": "ip-192-168-0-163.ec2.internal",
            "PrivateIpAddress": "192.168.0.163",
            "ProductCodes": [],
            "PublicDnsName": "",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "StateTransitionReason": "",
            "SubnetId": "subnet-03f6120aec2780686",
            "VpcId": "vpc-05ebce82711fe6a19",
            "Architecture": "x86_64",
            "BlockDeviceMappings": [],
            "ClientToken": "4e955d97-0aca-4eaa-9d4e-65f6a386db73",
            "EbsOptimized": false,
            "EnaSupport": true,
            "Hypervisor": "xen",
            "NetworkInterfaces": [
                {
                    "Attachment": {
                        "AttachTime": "2020-12-24T09:00:39+00:00",
                        "AttachmentId": "eni-attach-01f7eef2d9206b1ba",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attaching",
                        "NetworkCardIndex": 0
                    },
                    "Description": "",
                    "Groups": [
                        {
                            "GroupName": "private",
                            "GroupId": "sg-046fe855b60b9bd49"
                        }
                    ],
                    "Ipv6Addresses": [],
                    "MacAddress": "0a:94:c1:02:11:f7",
                    "NetworkInterfaceId": "eni-0d619b2ab7e7efb71",
                    "OwnerId": "662260156742",
                    "PrivateIpAddress": "192.168.0.163",
                    "PrivateIpAddresses": [
                        {
                            "Primary": true,
                            "PrivateIpAddress": "192.168.0.163"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Status": "in-use",
                    "SubnetId": "subnet-03f6120aec2780686",
                    "VpcId": "vpc-05ebce82711fe6a19",
                    "InterfaceType": "interface"
                }
            ],
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SecurityGroups": [
                {
                    "GroupName": "private",
                    "GroupId": "sg-046fe855b60b9bd49"
                }
            ],
            "SourceDestCheck": true,
            "StateReason": {
                "Code": "pending",
                "Message": "pending"
            },
            "VirtualizationType": "hvm",
            "CpuOptions": {
                "CoreCount": 1,
                "ThreadsPerCore": 1
            },
            "CapacityReservationSpecification": {
                "CapacityReservationPreference": "open"
            },
            "MetadataOptions": {
                "State": "pending",
                "HttpTokens": "optional",
                "HttpPutResponseHopLimit": 1,
                "HttpEndpoint": "enabled"
            },
            "EnclaveOptions": {
                "Enabled": false
            }
        }
    ],
    "OwnerId": "662260156742",
    "ReservationId": "r-000a73bffe02f8407"
}

```
4. Check the status of the EC2 instances:
```bash
$ aws ec2 describe-instances --filters "Name=vpc-id,Values=vpc-05ebce82711fe6a19" --query "Reservations[*].Instances[*].{id:InstanceId,PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress,Subnet:SubnetId}" --output=table
--------------------------------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
|                                  DescribeInstances                                  |
+---------------+----------------+----------------------------+-----------------------+
|   PrivateIP   |   PublicIP     |          Subnet            |          id           |
+---------------+----------------+----------------------------+-----------------------+
|  192.168.0.163|  None          |  subnet-03f6120aec2780686  |  i-0b029d431795edb40  |
|  192.168.1.213|  54.160.57.105 |  subnet-0a94a1e951e6a151c  |  i-0e9f5c78a1df96534  |
+---------------+----------------+----------------------------+-----------------------+

```
SSH (use the -A option to forward your authentication info) to the public EC2 host from your computer:
```bash
$ ssh -A ec2-user@54.160.57.105
```
