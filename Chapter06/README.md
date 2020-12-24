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

NAT-GW
What happens if the subnet default route is not pointing to IGW? The subnet is classified as 
a private subnet with no connectivity to the internet. However, some of situation, your VM 
in private subnet needs to access to the Internet. For example, download some security 
patch.
In this case, you can setup NAT-GW. It allows you access to the internet from the private 
subnet. However, it allows outgoing traffic only, so you cannot assign public IP address for 
a private subnet. Therefore, it is suitable for backend instances, such as the database.
Let's create NAT-GW and configure a second subnet (192.168.1.0/24) as a private 
subnet that routes to NAT-GW using the following steps:   
