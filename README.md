# Overview
This project demonstrates how to migrate a web application's backend database from a locally hosted MySQL instance to a fully managed Amazon RDS MySQL database.

#### The migration process involves:
- Generating sample data on the existing local database.

- Migrating the data to a newly provisioned Amazon RDS database instance.

- Setting up the required AWS infrastructure, including two private subnets across different Availability Zones and a dedicated security group for the RDS instance.

- Deploying the Amazon RDS instance within a secure network configuration.

- Reconfiguring the café web application to connect to the Amazon RDS database instead of the local instance.

This setup enhances the application's scalability, availability, and operational management by leveraging Amazon RDS as a fully managed database service.


## Starting architecture

The following diagram illustrates the topology of the café web application runtime environment before the migration. The application database runs in an Amazon Elastic Compute Cloud (Amazon EC2) Linux, Apache, MySQL, and PHP (LAMP) instance along with the application code. The instance has a T3 small instance type and runs in a public subnet so that internet clients can access the website. A CLI Host instance resides in the same subnet to facilitate the administration of the instance by using the AWS Command Line Interface (AWS CLI).


<img width="816" height="904" alt="image" src="https://github.com/user-attachments/assets/306fe0f5-fd55-4bcd-9b10-8fa3210151b8" />


Final architecture

The following diagram illustrates the topology of the café web application runtime environment after the migration.

You migrate the local café database to an Amazon RDS database that resides outside the instance. The Amazon RDS database is deployed in the same virtual private cloud (VPC) as the instance.

<img width="1516" height="878" alt="image" src="https://github.com/user-attachments/assets/65d28a91-edc0-4536-85c1-61e031bb8047" />

# Objectives
- Provision an Amazon RDS MariaDB instance via AWS CLI for managed database hosting.

- Perform data migration from an EC2-hosted MariaDB instance to the Amazon RDS instance.

- Implement monitoring of the RDS instance using Amazon CloudWatch metrics to track performance and availability.


## Generating order data on the café website

We'll browse the café website and place a few orders that are stored in the existing database. Placing orders creates data for the application before the application is migrated to new Amazon RDS instance.

<img width="1357" height="539" alt="Screenshot 2025-07-19 203336" src="https://github.com/user-attachments/assets/0c2aea64-ada9-4b24-bc2d-aa32355f8d7a" /> <img width="1332" height="523" alt="Screenshot 2025-07-19 203600" src="https://github.com/user-attachments/assets/3ab0fe6c-4c46-41f6-a188-47501fa31826" /> <img width="1364" height="542" alt="Screenshot 2025-07-19 203638" src="https://github.com/user-attachments/assets/5e4233e2-b404-4950-bca6-c9402e0e6e77" />


# Provisioning the Amazon RDS Instance
As part of this project, you use EC2 Instance Connect to securely access a pre-provisioned CLI host with AWS CLI installed. From this host, you execute AWS CLI commands to:

- Configure the AWS CLI with appropriate credentials and region settings.

- Create the required networking components for the RDS deployment, including:

     - A security group to control access to the Amazon RDS instance.

     - Two private subnets in different Availability Zones, along with a database subnet group for multi-AZ support.

- Provision the Amazon RDS MariaDB instance within the configured network and security boundaries.

This setup establishes a secure, scalable database environment in line with AWS best practices.

#### Connecting to the CLI Host instance
To execute AWS CLI commands, you first connect to the pre-configured CLI Host EC2 instance using EC2 Instance Connect from the AWS Management Console:

- Navigate to EC2 > Instances and select the CLI Host instance.

- Choose Connect, then use the EC2 Instance Connect tab to establish a secure session.

Alternatively, you may connect using an SSH client following the official AWS guidance.

Once connected, the CLI Host allows configuring and using the AWS CLI for provisioning AWS resources and managing services.

#### Configuring the AWS CLI
To set up the AWS CLI profile with credentials, in the EC2 Instance Connect terminal, run the following command:
  `` aws configure ``
  ### then input necessary credentials
      AWS Access Key ID: 

      AWS Secret Access Key: 

      Default region name: 

      Default output format

# Creating prerequisite components
Next, we create the prerequisite infrastructure components for the Amazon RDS instance. Specifically, you create the following components that are shown in the final architecture diagram:

- CafeDatabaseSG (Security group for the Amazon RDS database)

- CafeDB Private Subnet 1

- CafeDB Private Subnet 2

- CafeDB Subnet Group (Database subnet group)

#### First, we create the CafeDatabaseSG security group. This security group is used to protect the Amazon RDS instance. It will have an inbound rule that allows only MySQL requests (using the default TCP protocol and port 3306) from instances that are associated with the CafeSecurityGroup. This rule ensures that only the CafeInstance is able to access the database
  - run the following command:
```
aws ec2 create-security-group \
--group-name CafeDatabaseSG \
--description "Security group for Cafe database" \
--vpc-id <CafeInstance VPC ID>
```
<img width="639" height="154" alt="Screenshot 2025-07-19 221327" src="https://github.com/user-attachments/assets/3f1386cc-8116-4aba-b0ac-8d83477f0045" />


##### To create the inbound rule, run the following command. In the command, replace <CafeDatabaseSG Group ID> with the GroupId value that you recorded in the previous step :

```
aws ec2 authorize-security-group-ingress \
--group-id <CafeDatabaseSG Group ID> \
--protocol tcp --port 3306 \
--source-group <CafeSecurityGroup Group ID>
```

<img width="665" height="82" alt="Screenshot 2025-07-19 222048" src="https://github.com/user-attachments/assets/610612ff-5a0b-4174-9417-e9cc7667297e" />


#### To confirm that the inbound rule was applied appropriately, run the following command: 
```
aws ec2 describe-security-groups \
--query "SecurityGroups[*].[GroupName,GroupId,IpPermissions]" \
--filters "Name=group-name,Values='CafeDatabaseSG'"
```

<img width="530" height="367" alt="Screenshot 2025-07-19 222311" src="https://github.com/user-attachments/assets/23a2c480-e8f7-4c9b-abb1-af787adb4d37" />
###### inbound rule applied !!!

#### Next, you create two private subnets and a database subnet group. First, you create CafeDB Private Subnet 1. This subnet hosts the RDS DB instance. It is a private subnet that is defined in the same Availability Zone as the CafeInstance.

We assign the subnet a Classless Inter-Domain Routing (CIDR) address block that is within the address range of the VPC but that does not overlap with the address range of any other subnet in the VPC. This reason is why you collected the information about the VPC and existing subnet CIDR blocks:

``
Cafe VPC IPv4 CIDR block: 10.200.0.0/20

Cafe Public Subnet 1 IPv4 CIDR block: 10.200.0.0/24
``
considering the CIDR of current subnet we will use the address range " 10.200.2.0/23 "

##### To create the subnet, run the following command. In the command, replace <CafeInstance VPC ID> and <CafeInstance Availability Zone> with the values of CafeVpcID and CafeInstanceAZ, respectively. 

```
aws ec2 create-subnet \
--vpc-id <CafeInstance VPC ID> \
--cidr-block 10.200.2.0/23 \
--availability-zone <CafeInstance Availability Zone>

```

<img width="663" height="323" alt="Screenshot 2025-07-19 224303" src="https://github.com/user-attachments/assets/feb44f23-9ea9-4fb3-bbf1-58e72d07a76b" />

From the output of the command, note the value for SubnetId. You use this information later for CafeDB Private Subnet 1.


##### Next, you create CafeDB Private Subnet 2. This is the extra subnet that is required to form the database subnet group. It is an empty private subnet that is defined in a different Availability Zone than the CafeInstance.

Similar to what you did in the previous steps, you must assign a CIDR address block to the subnet that is within the address range of the VPC but does not overlap with the address range of any other subnet in the VPC. So far, you have used the following address ranges:
    - Cafe VPC IPv4 CIDR block: 10.200.0.0/20

    - Cafe Public Subnet 1 IPv4 CIDR block: 10.200.0.0/24

    - Cafe Private Subnet 1 IPv4 CIDR block: 10.200.2.0/23
we will use the "10.200.10.0/23" address range for this second private subnet.

##### To create the second subnet, run the following command. In the command, replace <CafeInstance VPC ID> 
```
aws ec2 create-subnet \
--vpc-id <CafeInstance VPC ID> \
--cidr-block 10.200.10.0/23 \
--availability-zone <availability-zone>
```

<img width="665" height="321" alt="Screenshot 2025-07-19 230958" src="https://github.com/user-attachments/assets/d417b0af-fac2-4eb9-b1b7-512e64cfc7aa" />
From the output of the command, note the value for SubnetId. You use this information later for CafeDB Private Subnet 2.

#### Next, you create CafeDB Subnet Group.
For the Amazon RDS instance for the café, the DB subnet group consists of the two private subnets that we created in the previous steps: CafeDB Private Subnet 1 and CafeDB Private Subnet 2.

#### In the terminal window, run the following command. In the command, replace <Cafe Private Subnet 1 ID> and <Cafe Private Subnet 2 ID> 
```
aws rds create-db-subnet-group \
--db-subnet-group-name "CafeDB Subnet Group" \
--db-subnet-group-description "DB subnet group for Cafe" \
--subnet-ids <Cafe Private Subnet 1 ID> <Cafe Private Subnet 2 ID> \
--tags "Key=Name,Value= CafeDatabaseSubnetGroup"
```
<img width="665" height="377" alt="Screenshot 2025-07-19 232124" src="https://github.com/user-attachments/assets/5bbf730f-1fae-4ccf-9a3d-68adfc139c17" />

## Creating the Amazon RDS MariaDB instance
now create the CafeDBInstance that is shown in the final architecture. Using the AWS CLI, you create an Amazon RDS MariaDB instance with the following configuration settings:
- DB instance identifier: CafeDBInstance

- Engine option: MariaDB

- DB engine version: 10.5.13

- DB instance class: db.t3.micro

- Allocated storage: 20 GB

- Availability Zone: CafeInstanceAZ

- DB Subnet group: CafeDB Subnet Group

- VPC security groups: CafeDatabaseSG

- Public accessibility: No

- Username: root

- Password: Re:Start!9




















