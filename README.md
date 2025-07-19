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

#### In the terminal window, run the following command. In the command, replace <CafeInstance Availability Zone> with the CafeInstanceAZ value that you recorded at the beginning of the lab, and replace <CafeDatabaseSG Group ID> with the value 

```
aws rds create-db-instance \
--db-instance-identifier CafeDBInstance \
--engine mariadb \
--engine-version 10.6.14 \
--db-instance-class db.t3.micro \
--allocated-storage 20 \
--availability-zone <CafeInstance Availability Zone> \
--db-subnet-group-name "CafeDB Subnet Group" \
--vpc-security-group-ids <CafeDatabaseSG Group ID> \
--no-publicly-accessible \
--master-username root --master-user-password 'Re:Start!9'

```
<img width="679" height="446" alt="image" src="https://github.com/user-attachments/assets/8cf1cf9f-e4af-4b9e-a4ab-60e21d49460e" />

#### To check the status of the database, run the following command:

```
aws rds describe-db-instances \
--db-instance-identifier CafeDBInstance \
--query "DBInstances[*].[Endpoint.Address,AvailabilityZone,PreferredBackupWindow,BackupRetentionPeriod,DBInstanceStatus]"
```

This command shows the following information for the database, including the status of the database as the last returned value:

  - Endpoint address

  - Availability Zone

  - Preferred backup window

  - Backup retention period

  - Status of the database

<img width="681" height="448" alt="Screenshot 2025-07-19 235907" src="https://github.com/user-attachments/assets/68fcd855-2ea3-49bd-a622-2ebe5f3d09b9" />

# Migrating application data to the Amazon RDS instance

We migrate the data from the existing local database to the newly created Amazon RDS database. Specifically, you do the following:

- Connect to the CafeInstance by using EC2 Instance Connect.

- Use the mysqldump utility to create a backup of the local database.

- Restore the backup to the Amazon RDS database.

- Test the data migration.

#### Connect to the CafeInstance by using EC2 Instance Connect. Then, use the mysqldump utility to create a backup of the local cafe_db database.
run the following command: 
```
mysqldump --user=root --password='Re:Start!9' \
--databases cafe_db --add-drop-database > cafedb-backup.sql
```
This command generates SQL statements in a file named cafedb-backup.sql, which can be run to reproduce the schema and data of the original cafe_db database.

<img width="678" height="504" alt="Screenshot 2025-07-20 001446" src="https://github.com/user-attachments/assets/c6db6b06-ab4c-4c78-a44b-7a13ca8641eb" />

#### To review the contents of the backup file,enter the following command in the terminal window:
``less cafedb-backup.sql``

When you use the less command, use the up and down arrow keys to move one line up or one line down, respectively. You can also use the Page up and Page down keys to navigate one page up or one page down, respectively.


#### Next, you restore the backup to the Amazon RDS database by using the mysql command. You must specify the endpoint address of the Amazon RDS instance in the command. 

```
mysql --user=root --password='Re:Start!9' \
--host=<RDS Instance Database Endpoint Address> \
< cafedb-backup.sql

```
This command creates a MySQL connection to the Amazon RDS instance and runs the SQL statements in the cafedb-backup.sql file.

#### Finally, you verify that the cafe_db was successfully created and populated in the Amazon RDS instance. You open an interactive MySQL session to the instance and retrieve the data in the product table of the cafe_db database. 

In the SSH window, enter the following command. 
```
mysql --user=root --password='Re:Start!9' \
--host="cafedbinstance.ceh5ojdrdrep.us-west-2.rds.amazonaws.com" \
cafe_db
````
# Configuring the website to use the Amazon RDS instance
On the AWS Management Console, in the Search bar, enter and choose Systems Manager to go to AWS Systems Manager.

In the left navigation pane, choose Parameter Store.

From the My parameters list, choose /cafe/dbUrl. The current value of the parameter is displayed, along with its description and other metadata information.

Choose Edit.

In the Parameter details page, replace the text in the Value box with the RDS Instance Database Endpoint Address value that you recorded earlier.

Choose Save changes.

The dbUrl parameter now references the RDS DB instance instead of the local database.

Next, you test the website to confirm that it is able to access the new database correctly.

In a new browser window, paste the URL for CafeInstanceURL that you copied to a text editor at the beginning of the lab.

The website's home page should load correctly. 

Choose the Order History tab. and observe the number of orders in the database. Compare this number with the number of orders that you recorded before the database migration. Both numbers should be the same.


# Monitoring the Amazon RDS database

One of the benefits of using Amazon RDS is the ability to monitor the performance of a database instance. Amazon RDS automatically sends metrics to CloudWatch every minute for each active database. In this task, you identify some of these performance metrics and learn how to monitor a metric in the Amazon RDS console.

On the AWS Management Console, in the Search bar, enter and choose RDS to open the RDS Management Console.

In the left navigation pane, choose Databases.

From the Databases list, choose cafedbinstance. Detailed information about the database is displayed.

Choose the Monitoring tab. By default, this tab displays a number of key database instance metrics that are available from CloudWatch. Each metric includes a graph that shows the metric as it is monitored over a specific time span.

The list of displayed metrics includes the following:

CPUUtilization: The percent of CPU utilization

DatabaseConnections: The number of database connections in use

FreeStorageSpace: The amount of available storage space

FreeableMemory: The amount of memory (RAM) available on the Amazon RDS instance

WriteIOPS: The average number of disk write I/O operations per second

ReadIOPS: The average number of disk read I/O operations per second

Tip: Some of the metrics listed might appear on the second or third pages of charts. To see additional metrics, choose 2 or 3 to the right of the CloudWatch search box.

Next, you monitor the DatabaseConnections metric as you create a connection to the database from the CafeInstance.

To open an interactive SQL session to the RDS cafe_db instance, in the CafeInstance terminal window, enter the following command. In the command, replace <RDS Instance Database Endpoint Address> with the value that you recorded earlier.

mysql --user=root --password='Re:Start!9' \
--host=<RDS Instance Database Endpoint Address> \
cafe_db
To retrieve the data in the product table, enter the following SQL statement:

select * from product;
   The query should return rows from the table.

In the Amazon RDS console, choose the DatabaseConnections graph to open it in a larger view. The graph shows a line that indicates that 1 connection is in use. This connection was established by the interactive SQL session from the CafeInstance.

Tip: If the graph does not show any connections in use, wait 1 minute (which is the sampling interval), and then choose Refresh. It should now show one open connection.

To close the connection from the interactive SQL session, in the CafeInstance terminal window, enter the following command:

exit
Wait 1 minute, and then in the DatabaseConnections graph in Amazon RDS console, choose Refresh. The graph now shows that the number of connections in use is 0.

If you have time, explore the other graphs.

# Use Case
This project showcases the migration of a web application's backend database from a self-managed MySQL instance on Amazon EC2 to a fully managed Amazon RDS MariaDB service. It reflects a typical real-world scenario where organizations move from managing their own databases to using cloud-native managed services for improved reliability, scalability, and maintenance overhead reduction.

It is particularly relevant for:

- Developers and DevOps Engineers looking to practice AWS infrastructure provisioning and database migration.

- Organizations aiming to decouple their application from tightly coupled EC2-hosted databases.

- Cloud Architects designing high-availability, production-grade backend systems on AWS.

# Key Benefits
- Managed Database Service
By migrating to Amazon RDS, you offload routine database management tasks such as backups, patching, monitoring, and scaling, allowing teams to focus on application development.

- Improved Scalability and Availability
Deployment within multiple Availability Zones using private subnets ensures high availability and fault tolerance.

- Enhanced Security
With VPC security groups and private networking, database access is tightly controlled and isolated from public internet exposure.

- Automated Monitoring
Amazon CloudWatch integration provides real-time visibility into database performance, enabling proactive performance tuning and troubleshooting.

- Disaster Recovery and Backup
Built-in automated backups and snapshots via RDS enhance data protection and recovery options.

- Simplified Operations via AWS CLI and Systems Manager
Demonstrates how to manage infrastructure and application parameters programmatically, enabling Infrastructure as Code (IaC) and streamlined operations.

# Real-World Application Example
A growing e-commerce business wants to improve the reliability of its ordering system. By migrating its MySQL backend to Amazon RDS and implementing proper monitoring, the business reduces downtime risks, ensures better performance during peak shopping periods, and lowers the operational burden on its small IT team.





















