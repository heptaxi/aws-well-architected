# Fundamentals

https://www.youtube.com/watch?v=-ObImxw1PmI

# Core Pillar
- Identity and Access Management (IAM)
- Data Encryption (at rest and in transit)
- Secure Networking (VPC etc)

# IAM
- Identity: Authentication
- Access Management: Authorization

Do not do anything with a root account (AWS Account) except administrative tasks.

## Identities
a Principal
    - Entity making a call to an AWS Service

Multiple types of identities:
an IAM User
    - Long-term security credentials
    - For humands
an IAM Role
    - Short-term credentials = Temporary security credentials
    - For humans, applications or machines

## Creating an IAM Role
Choose who/what is is for:
- AWS Service (non-human process)
- Another Account (cross-account access)
- Web-Identity or SAML 2.0 Federation (human)

## How authentication works in AWS
You have a pair of credentials: AccessKeyId and SecretAccessKey.
Doing API request with signed HMAC of body and headers, signed with SecretAccessKey.

## Permissions Management
IAM Policies define the permissions to Identities or Resources.
Mainly a JSON object, with one or many statements, saying effect, action, resource and some conditions.

arn = Amazon Resource Name.

Core Permission Resolution Principals:
- Access need to be explicitly granted by one or more statements
- Access is Denied if any statement denies access

Follow the least privilege principal.

To learn about the actions, look at the doc for service by service authorization details.

Example:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:BatchGetItem",
                "dynamodb:GetItem",
                "dynamodb:Query",
            ],
            "Resource": [
                "arn:aws:<service>:<region>:<accountID>:<resourceType>/<resourcePath>",
                "arn:aws:dynamodb:us-east2:111222333:table/MyTableName/index/*",
                "arn:aws:dynamodb:us-east2:111222333:table/MyTableName",
            ],
            "Condition": {
                "StringEquals": {
                    "secretsmanager:ResourceTag/Project":"${aws:PrincipalTag/Project}"
                }
            }
        }
    ]
}
```

##  Resource Based Policies
Policy attached to a resource, example S3 Bucket
Example, to give cross account acccess -> both sides need to say Allow Access.

```
{
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam:111222333:root" //trust account 111222333 to generate policies allowing access
    }
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::other-account-bucket/*"
}
```

Some services don't support resource based policies, example DynamoDB.
So we need to:
- create a role R1 in account B
    - attach an IAM policy on the Role to give it permissions to do stuff
    - create a Resource Base Policy (a Trust Policy) to trust account A's root user (or any other)
- create a Role in account A
    - attach it an IAM Policy to give it sts:AssumeRole permissioons on the Role R1 as the resource

## AWS Organizations
Organizational Unit = OU.
Organization is a hierachy of OUs.

- use Service Control Policies to bound access throughout AWS Orgs.
- use Permission Boundaries to make sure users can not escalate their privilege

# AWS Key Management Service (KMS)
a managed service that handles key management, with envelope encryption.
Keeps a master Key, secure, use it to encrypt data key (symetric). 

- KMS.GenerateDataKey: symmetric data key (plaintext or encrypted)
- use plaintext data key to encrypt data, then discard it
- store encrypted data key alongside with encrypted data
- To decrypt:
    - KMS.Decrypt(encryptedDataKey)-> plaintextDataKey
    - use plaintext symmetric datakey to decrypt data

If using a Customer Managed Key (CMK) by KMS to encrypt data in s3, you need
to give permissions to the Role to decrypt the data (using your CMK).

```
{
    "Effect": "Allow",
    "Action": "kms:Decrypt",
    "Resource":"arn:aws:kms:us-east-2:111222333:key/key123"
},
{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource":"arn:aws:s3:::myBucket/*"
},
```

# Virtual Private Cloud (VPC)
AWS is made of many regions, each region is made of at least 3 Availability Zones (AZ).
Region: example eu-west-1
Availability zone: example eu-west-1a,1b,1c
VPC is a customer's network accross a region.

- CIDR
IP Adress Range for the network.
example 10.0.0.0/16 means 10.0.X.Y

- Subnet
an IP Range of the VPC.
Private Subnet: deny public access over the internet to the subnet (ex 10.0.0.0/24)
Public Subnet: allow public access over the internet to the subnet (ex 10.0.52.0/24)

- Security group
mainly a firewall.

Mainly use the 3-Tier strategy: Frontend (mainly loadBalancers) -> Applications -> Databases.

Security Group defintion:
allow specific type of traffic, using a protocol, on port, from source
    - inbound rules
    - outbound rules

Frontend SG: allow HTTPS over TCP on port 443 from all external sources (0.0.0.0:0)
Applications SG: allow HTTP over TCP on port 8443 from a specific security group (by ID).

- Routing
a Public subnet has a route to the internet through a Gateway.
a Private subnet has no publicly routable IP adresses, and no route to any Internet Gateway.

Route table: default is Dest=10.0.0.0/16, Target=local, Status=Active.
To Allow public internet traffic: Dest=0.0.00/0, Target=igw, Status=Active.

- VPC Endpoints
```
dig logs.us-east-2.amazonaws.com +shot
52.95.18.51
````

AWS Services are not in your VPC, to give access to AWS Service in your private subnet that has no route to public IP, use VPC endpoints.
--> route to local and overrides DNS records so that it resolves to your local network.

Create a security group dedicated to AWS Services, and even use VPC Endpoints Policies (IAM). Policy is not attached to identity or resource but to a network, and it defines a
boundary.
Example
```
{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "logs:*",
    "Resource": "arn:aws:logs:us-east-2:111222333:*",
    "Condition": {
        "StringEquals": {
            "aws:PrincipalOrgID": "o-a1234"
        }
    }
}
```