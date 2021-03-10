# How to Use Amazon EFS with Amazon Lightsail

Amazon Lightsail is a virtual private server (VPS) and is the easiest way to get started with AWS for developers, small businesses, students, and other users who need a solution to build and host their applications on cloud.

Amazon EFS is a simple, serverless, set-and-forget, elastic file system that makes it easy to set up, scale, and cost-optimize file storage in the AWS Cloud.

In this guide I'll show how create and connect to an EFS file system from one or more Lightsail instances. Multiple Lightsail instances can access the same shared EFS file system, enabling you to build more highly available and scalable applications on Lightsail.

## Getting started

To get started, you'll need an [AWS account](https://portal.aws.amazon.com/billing/signup). To complete this guide you must install the [AWS Command Line Interface (CLI) tool](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and have [jq](https://stedolan.github.io/jq/) on your system. In this guide I'll be using jq to extract information returned by AWS CLI comments in JSON format. If you don't have an AWS account, the AWS CLI, or ```jq```, follow the provided links to get them before proceding further.

## 1. Peer the Lightsail VPC with the default VPC

Lightsail can use VPC peering to connect to other AWS services. VPC peering creates a network connection between the Lightsail VPC and the region's default VPC, providing a secure communication link that allows Lightsail instances to access other AWS services.

Peer the Lightsail and default VPCs using the  [peer-vpc](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/peer-vpc.html) command. Use ```jq``` to extract the ```<Lightsail VPC ID>``` and ```<Default VPC ID>``` for later use. 

```
$ aws lightsail peer-vpc | jq -r '.operation.resourceName, .operation.operationDetails'

<Lightsail VPC ID>
<Default VPC ID>
```

Note: I use ```<Identifier>``` notation throughout this guide to indicate the results returned by the ```jq``` command. 

Note: You can complete this guide without the use of ```jq``` but will need to extract the requried information from the JSON returned by AWS CLI commands yourself.

## 2. Create an EFS file system

Create a new file system with the [create-file-system](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-file-system.html) command. Use jq to extract the ```<EFS File System ID>``` for later use. 

```
$ aws efs create-file-system | jq -r '.FileSystemId'
   
<EFS File System ID>
```

This command creates a new general purpose file system with bursting throughput mode. Additional options are available if you want to use a difference file system performance model, encryption strategy, or throughput mode. For more information see the [EFS documentation](https://docs.aws.amazon.com/efs/index.html). 

## 3. Create EFS mount targets in each availability zone (AZ)

EFS mount targets allow you to mount an Amazon EFS file system from your Lightsail instance. You should create a mount target for each AZ that your Lightsail instances are lcoated in . Lightsail instances should connect to the EFS mount target in the same AZ. 

Obtain subnet information for the default VPC using the [describe-subnets](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-subnets.html) command. Provide the ```<Default VPC ID>``` obtained previously. Use '''jq''' to extract the ```<subnet X ID>```s for later use. 

```
$ aws ec2 describe-subnets --filters Name=vpc-id,Values=<Default VPC ID> Name=default-for-az,Values=true | jq -r '.Subnets[].SubnetId'

<subnet 1 ID>
<subnet 2 ID>
...
<subnet X ID>
```

Create an EFS mount target in each of the default subnets using the [create-mount-target](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-mount-target.html) command.  Use '''jq''' to extract the ```<EFS Mount Point IP Address>``` for later use.

```
$ aws efs create-mount-target --file-system-id <EFS File System ID> --subnet-id <Subnet ID> | jq -r '.IpAddress'

<EFS Mount Point IP Address>
```

Do this for each ```<Subnet ID>``` obtained in the previous section.

Note: If you don't have Lightsail instances in some AZs, you don't need to create a mount targets in those AZs. 

## 4. Create a rule allowing Lightsail to connect to EFS

Identify the VPC CIDR block for the Lightsail VPC using the [describe-vpc-peering-connections](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-vpc-peering-connections.html) command. Use '''jq''' to extract the ```<Lightsail VPC CIDR>``` block for later use.

```
$ aws ec2 describe-vpc-peering-connections --filters Name=requester-vpc-info.vpc-id,Values=<Lightsail VPC ID> | jq -r '.VpcPeeringConnections[0].RequesterVpcInfo.CidrBlock'

<Lightsail VPC CIDR>
```

Identify the default security group for the default VPC using the [describe-security-groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-security-groups.html) command. Use '''jq''' to extract the ```<Default Security Group ID>``` for later use.

```
$ aws ec2 describe-security-groups --filters Name=vpc-id,Values=<Default VPC ID> --group-names default | jq -r '.SecurityGroups[].GroupId'

<Default Security Group ID>
```

Create the security group rule with the [authorize-security-group-ingress](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-ingress.html) command. 

```
$ aws ec2 authorize-security-group-ingress --group-id <Default Security Group ID> --protocol tcp --port 2049 --cidr <Lightsail VPC CIDR>
```

This rule allows TCP traffic from the Lightsail VPC on port 2049 (the default NFS port).


## 5. Connect a Lightsail instance to the EFS file system

Connect to your Linux based Lightsail instance using your own compatible SSH client or connect using your browser from your instances [management page](https://lightsail.aws.amazon.com/ls/webapp/home/instances). For more information on connecting to your instance with SSH, visit the [SSH and connecting to your Lightsail instance](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/understanding-ssh-in-amazon-lightsail) page. 

Note: The instructions below are for Ubuntu based systems. Similar commands may be used with other supported Linux systems.

Install the NFS client and mount the EFS file system. These commands must be executed using ```sudo```. Replace ```<EFS Mount Point IP Address>``` with the IP address obtained in the previous step for the AZ your Lightsail instance is located in. 

```
apt install nfs-common
mkdir /mnt/efs
mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <EFS Mount Point IP Address>:/ /mnt/efs
```

Write a file to the shared file system

```
touch /mnt/efs/sharedfile.txt
```

Repeat step 1 for a different Linux based Lightsail instance in the same region. Be sure to use the EFS mount point for the AZ the different instance is in. Verify you can read the file written by the first Lightsail instance.

Congratulations. You have successfully connected your Lightsail instances to a shared EFS file system.


## Cleanup

Complete the following steps to cleanup resources you created in this guide.

Remove the security group rule using the [revoke-security-group-ingress](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/revoke-security-group-ingress.html) command.

```
$ aws ec2 revoke-security-group-ingress --group-id <Default Security Group ID> --protocol tcp --port 2049 --cidr <Lightsail VPC CIDR>
```

Unpeer the Lightsail and default VPCs using the [unpeer-vpc](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/unpeer-vpc.html) command.


```
$ aws lightsail unpeer-vpc
```

Remove the mount targets for the EFS file system by first using the [describe-mount-targets](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/describe-mount-targets.html) command to get the mount target IDs  
   
```
$ aws efs describe-mount-targets --file-system-id <EFS File System ID>

<Mount Target 1 ID>
<Mount Target 2 ID>
...
<Mount Target X ID>
```

Then use the [delete-mount-target](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/delete-mount-target.html) command to delete the individual mount targets.

```
$ aws efs delete-mount-target --mount-target-id <Mount Target ID>
```

Finally, delete the EFS file system using the [delete-file-system](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/delete-file-system.html)

```
$ aws efs delete-file-system --file-system-id <EFS File System ID>
```

