# How to Use Amazon EFS with Amazon Lightsail

In this guide I will describe how to mount an EFS drive on a Lightsail instamce.

```
vpcs=($(aws lightsail peer-vpc | jq -r '.operation.resourceName, .operation.operationDetails'))
```

To get started, you'll need an [AWS account](https://portal.aws.amazon.com/billing/signup) and must install the [AWS Command Line Interface (CLI) tool](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [jq](https://stedolan.github.io/jq/)) on your system. Follow the provided links if you don't have some of those.

Let's get started. 

## 1. Creating a peering connection between the Lightsail VPC and the default VPC
   Peer the Lightsail VPC with the Default VPC in the region with the  [peer-vpc](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/peer-vpc.html) command. Use jq to extract the VPC IDs for later use. 


   ```
   $ aws lightsail peer-vpc | jq -r '.operation.resourceName, .operation.operationDetails'

   <Lightsail VPC ID>
   <Default VPC ID>
   ```
   A peering connection is required for a Lightsail instance to connect to an EFS file system. 

## 2. Create an EFS file system

   Create a new file system with the [create-file-system](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-file-system.html) command. Use jq to extract the file system ID for later use. 

   ```shell
   $ aws efs create-file-system | jq -r '.FileSystemId'
   
   <EFS File System ID>
   ```

   This command creates a new empty general purpose file system with bursting throughput mode. Additional options are available if you want to use a difference file system performance model, encryption strategy, or throughput mode.

## 3. Create EFS mount points in each availability zone

   1. Obtain subnet information for the default VPC using the [describe-subnets](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-subnets.html) command. Provide the ```<Default VPC ID>``` obtained in section #1. Use jq to extract the default subnet IDs for later use. 

   ```
   $ aws ec2 describe-subnets --filters Name=vpc-id,Values=<Default VPC ID> Name=default-for-az,Values=true | jq -r '.Subnets[].SubnetId'

   <subnet 1 ID>
   <subnet 2 ID>
   ...
   <subnet X ID>
   ```

   2. Create an EFS mount point in each of the default subnets using the [create-mount-target](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-mount-target.html) command.  Use jq to extract the IP address of the mount point for later use.
    subnet IDs for later use. 
   ```shell
   $ aws efs create-mount-target --file-system-id <EFS File System ID> --subnet-id <Subnet ID> | jq -r '.IpAddress'
   
   <EFS Mount Point IP Address>
   ```

   Do this for each ```<Subnet ID>``` reported in section #3.

   Note: If you do not have Lightsail instances in some availability zones, you may omit creation of a mounting point in the subnet for that AZ. 

## 4. Create a rule allowing Lightsail to connect to EFS

   1. Identify the VPC CIDR block for the Lightsail VPC using the [describe-vpc-peering-connections](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-vpc-peering-connections.html) command. Use jq to extract the VPC CIDR block for later use.
   ```
   $ aws ec2 describe-vpc-peering-connections --filters Name=requester-vpc-info.vpc-id,Values=<Lightsail VPC ID> | jq -r '.VpcPeeringConnections[0].RequesterVpcInfo.CidrBlock'

   <Lightsail VPC CIDR>
   ```

   2. Identify the default security group for the default VPC using the [describe-security-groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-security-groups.html) command. Use jq to extract the security group ID for later use.
   ```
   $ aws ec2 describe-security-groups --filters Name=vpc-id,Values=<Default VPC ID> --group-names default | jq -r '.SecurityGroups[].GroupId'

   <Default Security Group ID>
   ```

   3. Create the security group rule with the [describe-security-groups](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-security-groups.html) command. 

   ```
   $ aws ec2 authorize-security-group-ingress --group-id <Default Security Groupo ID> --protocol tcp --port 2049 --cidr <Lightsail VPC CIDR>
   ```

## 5. Connect a Lightsail instance to the EFS file system



   Congratulations. You have successfully connected your Lightsail instances to a shared EFS file system. 


## Cleanup

Complete the following steps to the Lightsail container service that you created as part of this tutorial.

To cleanup and delete Lightsail resources, use the [delete-container-service](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/delete-container-service.html) command.
```
aws lightsail delete-container-service --service-name sample-service
```
The ```delete-container-service``` removes the container service, any associated container deployments, and container images.

## Additional Resources
The source code for this guide and this documentation is located in this [GitHub repository](https://github.com/AwsGeek/lightsail-containers-nginxq)
