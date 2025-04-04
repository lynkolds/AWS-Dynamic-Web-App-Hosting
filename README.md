![Alt text](/Host_a_Dynamic_Web_App_on_AWS_architecture.png)


# Hosting a Dynamic Website on AWS

This repository contains the resources and scripts used to deploy a dynamic website on Amazon Web Services (AWS). The project leverages various AWS services to ensure high availability, scalability, and security for the dynamic web application.

## Architecture Overview

The dynamic website is hosted on EC2 instances within a highly available and secure architecture that includes:

- **VPC (Virtual Private Cloud):** Contains public and private subnets across two Availability Zones (AZs) for fault tolerance and high availability.
- **Internet Gateway:** Allows communication between instances in the VPC and the internet.
- **Security Groups:** Acts as a virtual firewall to control inbound and outbound traffic.
- **Public Subnets:** Used for the NAT Gateway and Application Load Balancer, facilitating external access and load balancing.
- **Private Subnets:** Hosts web servers to enhance security.
- **EC2 Instance Connect Endpoint:** Provides secure SSH access to instances.
- **Application Load Balancer (ALB):** Distributes incoming web traffic across multiple EC2 instances.
- **Auto Scaling Group (ASG):** Automatically adjusts the number of EC2 instances based on traffic, ensuring scalability and resilience.
- **Amazon RDS (Relational Database Service):** Provides a managed relational database service for the dynamic website.
- **Amazon S3:** A scalable storage service used for hosting application files.
- **AWS Certificate Manager (ACM):** Manages SSL/TLS certificates for secure communication.
- **Amazon SNS (Simple Notification Service):** Sends notifications related to Auto Scaling Group activities.
- **Amazon Route 53:** Manages domain names and DNS for routing traffic to the dynamic website.
- **Amazon Machine Image (AMI):** Used to create consistent EC2 instances for deployment.

## Deployment Scripts

### Database Migration Script

This script is used for migrating the SQL script to the RDS database through an EC2 instance with an S3 IAM role. It includes steps for downloading and extracting Flyway, downloading the migration SQL script from AWS S3, and using Flyway to apply the SQL script to the RDS database with the appropriate RDS details.

```bash
#!/bin/bash

S3_URI=s3://forthesql/V1__shopwise.sql
RDS_ENDPOINT=database-1.cvc2q660ws86.us-east-2.rds.amazonaws.com
RDS_DB_NAME=rds
RDS_DB_USERNAME=rds
RDS_DB_PASSWORD=rds156.

# Update all packages
sudo yum update -y

# Download and extract Flyway
sudo wget -qO- https://download.red-gate.com/maven/release/com/redgate/flyway/flyway-commandline/10.9.1/flyway-commandline-10.9.1-linux-x64.tar.gz | tar -xvz

# Create a symbolic link to make Flyway accessible globally
sudo ln -s $(pwd)/flyway-10.9.1/flyway /usr/local/bin

# Create the SQL directory for migrations
sudo mkdir sql

# Download the migration SQL script from AWS S3
sudo aws s3 cp "$S3_URI" sql/

# Run Flyway migration
flyway -url="jdbc:mysql://$RDS_ENDPOINT:3306/$RDS_DB_NAME?allowPublicKeyRetrieval=true" \
  -user="$RDS_DB_USERNAME" \
  -password="$RDS_DB_PASSWORD" \
  -locations=filesystem:sql \
  migrate
```

### Dynamic Website Installation Script

This script is used for the installation of the dynamic website on an EC2 instance. It includes steps for installing Apache, PHP, MySQL, downloading the dynamic website code files from S3 buckets, and configuring the database credentials with RDS details.

```bash
#!/bin/bash

# Update all packages
sudo yum update -y

# Install Apache and enable it
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP and required extensions
sudo dnf install -y php php-pdo php-mysqlnd php-gd php-zip

# Install MySQL Server
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install -y mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Enable mod_rewrite in Apache
sudo sed -i '/<Directory "/var/www/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

# Download website files from S3
S3_BUCKET_NAME=forthecodes
sudo aws s3 sync s3://"$S3_BUCKET_NAME" /var/www/html

# Extract and configure the application
cd /var/www/html
sudo unzip shopwise.zip
sudo cp -R shopwise/. /var/www/html/
sudo rm -rf shopwise shopwise.zip
sudo chmod -R 777 /var/www/html

# Edit .env file for database credentials
sudo vi .env

# Restart Apache
sudo service httpd restart
```

### SSL Configuration and AMI Preparation Script

This script is used after securing an SSL certificate for the dynamic website. It updates the API URI and modifies `AppServiceProvider.php` before creating an Amazon Machine Image (AMI) for use in an Auto Scaling Group launch template.

```bash
#!/bin/bash

# Move to the website directory
cd /var/www/html

# Update API URI in .env file
sudo vi .env

# Restart Apache
sudo service httpd restart

# Navigate to the providers directory
cd app/Providers

# Edit AppServiceProvider.php to force HTTPS in production
sudo vi AppServiceProvider.php

# To ensure consistent secure connection for users, add the following code inside the "public function boot()" method:
# if (env('APP_ENV') === 'production') {
#   \Illuminate\Support\Facades\URL::forceScheme('https');
# }

# Restart Apache
sudo service httpd restart
```

## How to Use

1. Clone this repository to your local machine.
2. Follow the AWS documentation to create the required resources (VPC, subnets, Internet Gateway, etc.) as outlined in the architecture overview.
3. Use the provided scripts to set up the dynamic website application on EC2 instances within the VPC.
4. Configure the Auto Scaling Group, Load Balancer, and other services as per the architecture.
5. Access the dynamic website through its HTTPS domain name.

---

This setup ensures that your dynamic website is hosted in a highly available and scalable environment, leveraging AWS services like EC2, RDS, AMI, S3, and more. Make sure to adjust configurations as needed for your specific requirements.

