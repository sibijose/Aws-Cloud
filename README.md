# PROJECT VPC:  Provisioning Instances via AWS CLI
## What you’ll need

- IAM permissions to access EC2.
- Your Access Key ID
- Your Secret Access Key ID
- Linux Instance

## Purpose: 

To set up the AWS CLI on an AWS EC2 instance running Linux. Then provision an EC2 instance, install NGINX, create an AMI of the instance, provision a new instance from the AMI, and finally terminate our instance, all from the command line.

### Installing AWS CLI :

```sh
# sudo yum install curl -y
# curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# unzip awscliv2.zip
# sudo ./aws/install
# aws --version
```
####  Run the command aws configure you will be prompted to enter the next four things, press enter after each one.

- Your Access Key ID
- Your Secret Access Key ID
- Region of choice
- Default output format: json for today.

### Provisioning an EC2 instance through AWS:
```sh
aws ec2 create-vpc --cidr-block 172.16.0.0/16 --query Vpc.VpcId --output text > VPCid.txt // writing it to a file  VPCid.txt

 cat VPCid.txt

vpc-099f0e5b4d86eb930  //VPC ID
```
### Creating 2 Subnets 2 public 
```sh
aws ec2 create-subnet --vpc-id vpc-099f0e5b4d86eb930 --cidr-block 172.16.0.0/18
aws ec2 create-subnet --vpc-id vpc-099f0e5b4d86eb930 --cidr-block 172.16.64.0/18
aws ec2 create-subnet --vpc-id vpc-099f0e5b4d86eb930 --cidr-block 172.16.128.0/18  //private subnet
```
### Create Internet Gateway : To make our subnet publicly accessible by creating an Internet Gateway and route table:



```sh
aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text
igw-0eab14c344cbea994 //This is the internet gateway ID
```
### Attach our IGW to the VPC
```sh
aws ec2 attach-internet-gateway --vpc-id vpc-099f0e5b4d86eb930 --internet-gateway-id igw-0eab14c344cbea994
```
### Creating a Route Table

```sh
aws ec2 create-route-table --vpc-id vpc-099f0e5b4d86eb930 --query RouteTable.RouteTableId --output text
rtb-0fc308688b6bc0a41
```
### creating a Route on the Route Table to route traffic to the internet. 
```sh
 aws ec2 create-route --route-table-id rtb-0fc308688b6bc0a41 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0eab14c344cbea994
{
    "Return": true
}
```
### The route table is currently not associated with any subnet. We need to associate it with a subnet in your VPC to route traffic from that subnet to the internet. For this, we need our subnet IDs.
```sh
>>aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-099f0e5b4d86eb930" --query "Subnets[*].{ID:SubnetId,CIDR:CidrBlock}" //Querying subnet-IDs

[
    {
        "ID": "subnet-08f9e010122dc3813",
        "CIDR": "172.16.0.0/18"
    },
    {
        "ID": "subnet-07ae36ccab3ab6549",
        "CIDR": "172.16.64.0/18"
    },
    {
        "ID": "subnet-0f3c5999b2e5c4c68",
        "CIDR": "172.16.128.0/18"
    }
]
```
### We need to associate our subnet with the Route Table that is looking at the IGW this way we can have a public subnet. 

### Making the 2 subnets as public:

```sh
aws ec2 associate-route-table  --subnet-id subnet-08f9e010122dc3813 --route-table-id rtb-0fc308688b6bc0a41  // Route entry for First subnet

aws ec2 associate-route-table  --subnet-id subnet-07ae36ccab3ab6549 --route-table-id rtb-0fc308688b6bc0a41  //  Route entry for second subnet
```
###  Modify your subnet’s public IP addressing behaviorso that an instance launched into the subnet automatically receives a public IP address using the following modify-subnet-attribute command. 

### Otherwise, associate an Elastic IP address with your instance after launch so that the instance is reachable from the internet.(Applies to Private Zone)
```sh
 aws ec2 modify-subnet-attribute --subnet-id subnet-08f9e010122dc3813 --map-public-ip-on-launch

aws ec2 modify-subnet-attribute --subnet-id subnet-07ae36ccab3ab6549 --map-public-ip-on-launch
```

### Creating-Security-Group for allow port 22, 80, 443 

```sh
aws ec2 create-security-group --group-name vpc-traffic --description "Allow 22,80,443 access" --vpc-id vpc-099f0e5b4d86eb930

: To view the security group rules:


>>  aws ec2 describe-security-groups --group-ids sg-0b137a452205353df
```
### Creating a Key Pair
```sh
aws ec2 create-key-pair --key-name MyKeyNew --query 'KeyMaterial' --output text > MyKeyNew.pem 
```
### To Launch an instance we need to first choose our Amazon Machine Image(AMI). We can find this in the CLI by entering the following commands. Remember that not all AMIs are available in every region.

```sh
 aws ec2 describe-images --owners self amazon  --region ap-south-1 
```
### To launch our EC2 instance run the following command, with  AMI, Instance type, key pair name, Security Group ID, and Subnet ID.

```sh
aws ec2 run-instances --image-id ami-074dc0a6f6c764218  --count 1 --instance-type t2.micro --key-name MyKeyNew --security-group-ids 0b137a452205353df --subnet-id subnet-08f9e010122dc3813
```

