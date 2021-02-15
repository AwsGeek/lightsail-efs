# How to Use Amazon EFS with Amazon Lightsail

To get started, you'll need an [AWS account](https://portal.aws.amazon.com/billing/signup). To complete this guide you must install the [AWS Command Line Interface (CLI) tool](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [jq](https://stedolan.github.io/jq/)) on your system. Follow the provided links if you don't have some of those.

## 1. Peer the Lightsail VPC with the default VPC

   1. Peer the Lightsail and default VPCs using the  [peer-vpc](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/peer-vpc.html) command. Use jq to extract the VPC IDs for later use. 


   ```
   $ aws lightsail peer-vpc | jq -r '.operation.resourceName, .operation.operationDetails'

   <Lightsail VPC ID>
   <Default VPC ID>
   ```


## 2. Create an EFS file system

   Create a new file system with the [create-file-system](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-file-system.html) command. Use jq to extract the file system ID for later use. 

   ```
   $ aws efs create-file-system | jq -r '.FileSystemId'
   
   <EFS File System ID>
   ```

   This command creates a new general purpose file system with bursting throughput mode. Additional options are available if you want to use a difference file system performance model, encryption strategy, or throughput mode.

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

   ```
   $ aws efs create-mount-target --file-system-id <EFS File System ID> --subnet-id <Subnet ID> | jq -r '.IpAddress'
   
   <EFS Mount Point IP Address>
   ```

   Do this for each ```<Subnet ID>``` reported in section #3.

   Note: If you don't have Lightsail instances in some availability zones (AZ), you don't need to craete a mount point in the subnet for that AZ. 

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

   This rule allows traffic from the Lightsail VPC on port 2049 (the default NFS port).


## 5. Connect a Lightsail instance to the EFS file system

   Connect to your Linux based Lightsail instance using your own compatible SSH client or connect using your browser from your instances [management page](https://lightsail.aws.amazon.com/ls/webapp/home/instances). For more information on connecting to your instance with SSH, visit the [SSH and connecting to your Lightsail instance](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/understanding-ssh-in-amazon-lightsail) page. 

   1. Install the NFS client and mount the EFS file system. These commands must be executed using ```sudo```. Replace ```<EFS Mount Point IP Address>``` with the IP address obtained in step #3 for the availability zone your Lightsail instance is located in. 

   ```
   apt install nfs-common
   mkdir /mnt/efs
   mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <EFS Mount Point IP Address>:/ /mnt/efs
   ```

   2. Write a file to the shared file system
   ```
   touch /mnt/efs/sharedfile.txt
   ```

   3. Repeat step 1 for a different Linux based Lightsail instance in the same region. Be sure to use the EFS mount point for the availability zone the new instance is in. Verify you can read the file written by the first Lightsail instance

   Congratulations. You have successfully connected your Lightsail instances to a shared EFS file system. 


## Cleanup

Complete the following steps to cleanup resources you created in this guide.

Unpeer the Lightsail and default VPCs using the  [unpeer-vpc](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/unpeer-vpc.html) command.


```
$ aws lightsail unpeer-vpc
```

Remove the mount tarets for the EFS file system by first using the [describe-mount-targets](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/describe-mount-targets.html) command to get the mount target IDs  
   
```
$ aws efs describe-mount-targets --file-system-id <EFS File System ID>'

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
