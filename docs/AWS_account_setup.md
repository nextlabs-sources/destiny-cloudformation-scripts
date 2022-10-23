<!--
Author        : Duan Shiqiang <shiqiang.duan@nextlabs.com>
-->

# AWS Account Setup for using CloudFormation templates of CC/JPC

## Summary

This document describes the procedure for configure an AWS account for creating a Control Center and Java Policy Controller CloudFormation Stack.

## VPC and Subnets

The Stack must be created inside a VPC and at least 2 subnets from different AWS Availability Zones. And your VPC's **Network ACLs** should allow network connections between the subnets you choose for the stack. (by default, the Network ACLs allows anything)

For example, the user can create a VPC called "**Platform-VPC**" and have 2 subnets inside the VPC falls into different Availability Zones called "**Platform-Subnet-a**" and "**Platform-Subnet-b**".

> The VPC should have "**DNS resolution**" and "**DNS hostnames**" set to **yes**. The **DHCP Options Sets** attached to the VPC should set **domain-name-servers** option to value "**AmazonProvidedDNS**".

> Instances created inside the subnets chosen in the VPC should have Internet Access, an **Internet Gateway** or **NAT Gateway** or **NAT Instance** should be attached to the VPC and the **Route Table** associated with VPC (and subnets chosen) should have route **0.0.0.0/0 to the gateway**.

> Parameters **VpcId** and **EC2SubnetIDs** are required to input to the stack when creating it.

### Walkthrough of creating a new VPC and Configure it

1. In the AWS VPC console, **Create VPC** with name tag as "Platform-VPC" and CIDR block to any valid IP range (for example: 10.10.0.0/16).
2. Select the newly created VPC, right click to change "**DNS Hostnames**" to **yes**.
3. In the AWS VPC console's left panel, select **Subnets**, Create 2 subnets under different availability zones for the newly created VPC. Choose 2 non-overlapping subnets CIDR such as "10.10.1.0/24" and "10.10.2.0/24". Name the 2 subnets as "Platform-Subnet-a" and "Platform-Subnet-b".
4. In the AWS VPC console's left panel, select **Internet Gateways**, **Create Internet Gateway** for the VPC, give it a meaningful name such as "Platform-IGW".
5. Select the newly created Internet Gateway, right click to **attach** it to the newly created VPC.
6. In the AWS VPC console's left panel, select **Route Tables**, select the route table AWS created along with the newly created VPC. In the **Routes** panel below, **Add another route** with destination set to "0.0.0.0/0" and target set to the newly created Internet Gateway.

## RDS Subnet Group

The stack requires you have a **RDS Subnet Group** created with the subnets you have chosen. The RDS Subnet Group must be created with **exactly same VPC and subnets** you have chosen in "VPC and Subnets" section.

> Parameter **DBSubnetGroupName** is required to input to the stack when creating it.

### Walkthrough of creating a RDS Subnet Group

In the AWS RDS console's left panel, select **Subnet Groups**, **Create DB Subnet Group** with a meaningful name such as "Platform-Subnetgroup", select the VPC created in "VPC and Subnets" section and the 2 subnets created along with the VPC.

## Domain Name and Route53 Hosted Zone

### Domain Name

The Control Center Server requires to have a fully qualified domain name (FQDN), so the stack need to have a domain name specified. The domain name will be used to compose it's full hostname.

For specific, the **FQDN for Control Center** will be:

    {ControlCenterHostname}.{DNSDomain}

For example, the user may give parameter **ControlCenterHostname** value as **cc** and **DNSDomain** value as **example.com**. Then the FQDN of Control Center will be **cc.example.com**.

And the **FQDN for Java Policy Controller**'s load balancer will be:

    {JPCHostname}.{DNSDomain}

For example, the user may give parameter **JPCHostname** value as **jpc** and **DNSDomain** value as **example.com**. Then the FQDN of Java Policy Controller's load balancer will be **jpc.example.com**.

> The **ControlCenterHostname**, **JPCHostname** and **DNSDomain** are all parameters users need to input to the stack when creating it.

### Route53 Hosted Zone

A Route53 private hosted zone and a Route53 public hosted zone are required for the domain you specified. You need to **specify the VPC** you choose for the stack when the private hosted zone is created.

> Parameter **Route53PrivateHostedZone** and **Route53PublicHostedZone** are required to input to the stack when creating it.

### Walkthrough of creating the Route53 hosted zones

1. In the AWS Route 53 console's left panel, select **Hosted zones**, click **Create Hosted Zone**, choose type as **Private Hosted Zone for Amazon VPC**, input the domain name then choose the VPC created in "VPC and Subnets" section.
2. Create another hosted zone with same domain but choose type as **Public Hosted Zone**.

> To make the public hosted zone working (DNS populated to Internet), modify your domain's NS records to the public hosted zone's NS values.

## SSL Certificates for Load Balancers

You need to specify the SSL Certificates for the stack to use on the load balancers it creates. The certificate should contain the **FQDN of Control Center** and the **FQDN of Java Policy Controller's load balancer**.

> Parameter **SSLCertificateId** is required to input to the stack when creating it.

There are 2 ways to supply a SSL Certificate. You can refer to relevant AWS documentation at <http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/ssl-server-cert.htm>.

The first is to use **Amazon Certificate Manager** (ACM). ACM is recommended since it's free and users don't need to handle certificate renewal work.

To use ACM to generate SSL certificates which can be used by AWS Elastic Load Balancer, you can refer to AWS documentation at <http://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request.html>.

The second way to supply a SSL Certificate is to upload your own certificates to AWS IAM. Using this way you can purchase your certificates from third-party SSL vendor or use your self-signed certificates (that's should only happen for development purpose).

To upload and use your own SSL Certificate for AWS Elastic Load Balancer, you can refer to AWS documentation at <http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_server-certs_manage.html#UploadSignedCert>.

After you created your certificate using ACM or uploaded the SSL Certificate to IAM, you need to get the ARN (Amazon Resource Name) of the certificate.

> Parameter **SSLCertificateId** is required to input to the stack when creating it, which is the ARN of the certificate.

### Example of Creating and Uploading a Self-signed Certificate to AWS IAM Store

To create a self-signed certificate (not recommended for production usage) with domain "**example.com**" using openSSL under Linux:

    # this command will create a private key file named key.pem and a server certificate file named cert.pem
    openssl req -x509 -nodes -newkey rsa:2048 -keyout key.pem -out cert.pem -days 3650 -subj "/C=SG/ST=Singapore/O=Example Company/OU=DevOp/CN=*.example.com"

Then to upload the certificate to AWS IAM store using AWS CLI:

    # this command will upload the cert as name "selfsignedcccert"
    # you need to have permission to upload to IAM SSL Store, check AWS documentation for more details
    # you may need to set environment variable AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY to use aws cli
    aws iam upload-server-certificate --server-certificate-name selfsignedcccert --certificate-body file://cert.pem --private-key file://key.pem

The command above will give output like below if success:

    {
        "ServerCertificateMetadata": {
            "ServerCertificateId": "XXXXXXXXXXXXXXX",
            "ServerCertificateName": "selfsignedcccert",
            "Expiration": "2026-07-18T03:56:26Z",
            "Path": "/",
            "Arn": "arn:aws:iam::XXXXXXXX:server-certificate/selfsignedcccert",
            "UploadDate": "2016-07-20T03:56:46.858Z"
        }
    }

Keep the **Arn** value which is required as parameter **SSLCertificateId** when creating the stack.

### Walkthrough of create a certificate from ACM

1. In the AWS Certificate Manager Console, **Request a certificate** with 2 domain names: "**{your_domain_name}**" and "***.{your_domain_name}**".
2. AWS will send verification emails to the domain's owners, click the approve link in the email to approve the certificate to be approved.

Keep the **Arn** value which is required as parameter **SSLCertificateId** when creating the stack.

# AWS IAM Permissions

The IAM user should have read access to the CloudFormation templates. With a root user or an IAM user with IAM permissions, to grant an IAM user access to the S3 bucket where the templates are stored, following permission statement can be used:

>Note: The AWS account where the root user belongs to should be granted cross account permission for the S3 bucket storing the template before the root user can grant any other IAM users in same account permissions to the S3 bucket. Please contact **Nextlabs** for the AWS cross account permissions.

>Note: The Sid should be replaced by some other random value to avoid conflict.

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": [
                    "arn:aws:s3:::cf-templates-nextlabs/*"
                ]
            }
        ]
    }


The IAM user who is creating the stack also need to be granted those AWS permissions: (recommended only, more fine-grained permission set can be used but more tedious to set)

>Note: The `ACCOUNT-ID-WITHOUT-HYPHENS` should be replaced by the actual AWS account number

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "cloudformation:*",
                    "ec2:*",
                    "rds:*",
                    "elasticloadbalancing:*",
                    "route53:*",
                    "acm:DescribeCertificate",
                    "acm:GetCertificate",
                    "acm:ListCertificates",
                    "iam:GetServerCertificate",
                    "iam:ListServerCertificates",
                    "elasticfilesystem:*",
                    "autoscaling:*"
                ],
                "Resource": [
                    "*"
                ]
            },
            {
                "Effect":"Allow",
                "Action":[
                    "iam:CreateRole",
                    "iam:DeleteRole",
                    "iam:PutRolePolicy",
                    "iam:DeleteRolePolicy",
                    "iam:GetRole",
                    "iam:PassRole"
                ],
                "Resource":[
                    "arn:aws:iam::ACCOUNT-ID-WITHOUT-HYPHENS:role/nextlabs_platform/*"
                ]
            },
            {
                "Effect":"Allow",
                "Action":[
                    "iam:CreateInstanceProfile",
                    "iam:DeleteInstanceProfile",
                    "iam:AddRoleToInstanceProfile",
                    "iam:GetInstanceProfile",
                    "iam:RemoveRoleFromInstanceProfile"
                ],
                "Resource":[
                    "arn:aws:iam::ACCOUNT-ID-WITHOUT-HYPHENS:instance-profile/nextlabs_platform/*"
                ]
            }
        ]
    }

> The permissions are only enough for creating the stack once the prerequisites are fulfilled. Some operations inside the prerequisites may require more permissions.  

## CloudFormation Template URLs

The URL of CloudFormation template is:

* <https://s3.amazonaws.com/cf-templates-nextlabs/platform/latest/stack.yaml>


## The AWS Resources the Stack Creates

In specific, the resources the template creates including:

* A RDS instance (MSSQL or Oracle with license included type, or Postgres)
* A Control Center EC2 instance (let's call it cc_ec2)
* A Java Policy Controller EC2 instances (let's call them jpc_ec2)
* A security group for RDS instance: Only allow connection from cc_ec2
* An Elastic Load Balancer for Control Center (let's call it cc_lb)
* An Application Elastic Load Balancer for Java Policy Controller (let's call it jpc_lb)
* A security group for Control Center: Only allows tcp 8443 and 443 from the cc_lb and jpc_ec2
* A security group for Java Policy Controller: Only allows tcp 443 (JPC Rest Service Port) and tcp 1099 (JPC RMI Port) from jpc_lb
* A security group for Control Center's Load Balancer: Only allows tcp 8443 and 443 from anywhere
* A security group for Java Policy Controller's Load Balancer: Only allows tcp 443 and 1099 from anywhere
* A Route 53 Record Set for Control Center instance (an A record) in the private hosted zone
* A Route 53 Record Set for Java Policy Controller's Load Balancer (an alias record) in the private hosted zone
* An EFS FileSystem and EFS Mount Target for storing Control Center configuration files and certificate files
* A ServerIAMRole and a ServerInstanceProfile for EC2 instances to upload CloudWatch metrics and logs
* An autoscaling Group, one launch configuration and two Target Groups for JPC

## The Procedure to Create the Stack

Open the AWS Management Console, and choose CloudFormation service. Click "**Create Stack**" button. Then in the wizard to select a template, choose the "**Specify an Amazon S3 template URL**" option and input the template URL.

Next in the "**Specify Details**" step, give the stack a name and specify the parameters for the stack. Most parameters are self-explanatory and some of them are described in the prerequisite section.

Then click next and complete the stack creation.

## Use AWS CLI to Create Stack

Sample:

```shell
aws cloudformation create-stack --cli-input-json file://test_input/{input_file}
```
