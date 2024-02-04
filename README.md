# STEDI Human Balance Analytics

Udacity Data Engineering with AWS course project

Main AWS service used in this project: **Glue**, **Athena**

## Introduction

This project focuses on leveraging Spark and AWS Glue to build a data lakehouse solution for STEDI, a team working on a hardware device called the STEDI Step Trainer. The primary goal is to collect and process data from sensors on the Step Trainer and a companion mobile app. This curated data will be used to train a machine learning model to accurately detect steps in real-time.

## Project Details

1.STEDI Step Trainer Functionality

- The STEDI Step Trainer is a hardware device designed to train users in balance exercises.
- Equipped with sensors that collect data to train a machine learning algorithm for step detection.
- Companion mobile app interacts with device sensors and collects additional customer data.

2.Data Collection

- Motion sensor on the Step Trainer records object detection distance.
- Mobile app utilizes a smartphone accelerometer to capture motion in X, Y, and Z directions.
- Privacy considerations dictate that only data from early adopters who consented should be used for research.

3.Project Goal

- Develop AWS Glue jobs to extract, process, and curate data from Step Trainer sensors and the mobile app.
- Create a data lakehouse solution on AWS to facilitate machine learning model training.

4.Privacy Considerations

- Emphasize the importance of privacy in deciding which data can be used for training the machine learning model.
- Utilize only the data from early adopters who explicitly agreed to share their Step Trainer and accelerometer data for research purposes.

## Setup

### Git clone this repository
```bash
git clone [this repo]
```

### Configuring AWS Glue: S3 VPC Gateway Endpoint

Step 1: Creating an S3 Bucket

```bash
aws s3 mb s3://stedi-data-lakehouse/customer/landing
aws s3 mb s3://stedi-data-lakehouse/customer/trusted
aws s3 mb s3://stedi-data-lakehouse/customer/curated
aws s3 mb s3://stedi-data-lakehouse/accelerometer/landing
aws s3 mb s3://stedi-data-lakehouse/accelerometer/trusted
aws s3 mb s3://stedi-data-lakehouse/step_trainer/landing
aws s3 mb s3://stedi-data-lakehouse/step_trainer/trusted
aws s3 mb s3://stedi-data-lakehouse/step_trainer/curated
```

Step 2: S3 Gateway Endpoint
By default, Glue Jobs can't reach any networks outside of your Virtual Private Cloud (VPC). Since the S3 Service runs in different network, we need to create what's called an S3 Gateway Endpoint. This allows S3 traffic from your Glue Jobs into your S3 buckets. Once we have created the endpoint, your Glue Jobs will have a network path to reach S3.

First use the AWS CLI to identify the VPC that needs access to S3:

```bash
aws ec2 describe-vpcs
```

output example:

```text
{
    "Vpcs": [
        {
            "CidrBlock": "172.31.0.0/16",
            "DhcpOptionsId": "dopt-756f580c",
            "State": "available",
            "VpcId": "vpc-7385c60b",
            "OwnerId": "863507759259",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-664c0c0c",
                    "CidrBlock": "172.31.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": true
        }
    ]
}
```

From output, save `VpcId`.

Routing Table

Next, identify the routing table you want to configure with your VPC Gateway. You will most likely only have a single routing table if you are using the default workspace. Look for the RouteTableId:

```bash
aws ec2 describe-route-tables
```

output example:

```text
{
    "RouteTables": [

        {
      . . .
            "PropagatingVgws": [],
            "RouteTableId": "rtb-bc5aabc1",
            "Routes": [
                {
                    "DestinationCidrBlock": "172.31.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                }
            ],
            "Tags": [],
            "VpcId": "vpc-7385c60b",
            "OwnerId": "863507759259"
        }
    ]
}
```

From output, save `RouteTableId`.

Create an S3 Gateway Endpoint
Finally create the S3 Gateway, replacing the blanks with the **VPC** and **Routing Table Ids**

```bash
aws ec2 create-vpc-endpoint --vpc-id _______ --service-name com.amazonaws.us-east-1.s3 --route-table-ids _______
```


### Creating the Glue Service Role

AWS uses Identity and Access Management (IAM) service to manage users, and roles (which can be reused by users and services). A Service Role in IAM is a Role that is used by an AWS Service to interact with cloud resources.

For AWS Glue to act on your behalf to access S3 and other resources, you need to grant access to the Glue Service by creating an IAM Service Role that can be assumed by Glue:

```bash
aws iam create-role --role-name my-glue-service-role --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "glue.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}'
```

Grant Glue Privileges on the S3 Bucket

Replace the two blanks below with the S3 bucket name you created earlier, and execute this command to allow your Glue job read/write/delete access to the bucket and everything in it. You will notice the S3 path starts with `arn:aws:s3:::` . An ARN is an AWS Resource Name. The format is generally:

```
arn:[aws/aws-cn/aws-us-gov]:[service]:[region]:[account-id]:[resource-id]
```

You may notice that after `s3` there are three colons `:::` without anything between them. That is because S3 buckets can be cross-region, and cross AWS account. For example you may wish to share data with a client, or vice versa. Setting up an S3 bucket with cross AWS account access may be necessary.

Replace the blanks in the statement below with your S3 bucket name (ex: stedi-data-lakehouse) 

```bash
aws iam put-role-policy --role-name my-glue-service-role --policy-name S3Access --policy-document '{ "Version": "2012-10-17", "Statement": [ { "Sid": "ListObjectsInBucket", "Effect": "Allow", "Action": [ "s3:ListBucket" ], "Resource": [ "arn:aws:s3:::stedi-data-lakehouse" ] }, { "Sid": "AllObjectActions", "Effect": "Allow", "Action": "s3:*Object", "Resource": [ "arn:aws:s3:::stedi-data-lakehouse/*" ] } ] }'
```

Glue Policy

Last, we need to give Glue access to data in special `S3` buckets used for Glue configuration, and several other resources. Use the following policy for general access needed by Glue:

```bash
aws iam put-role-policy --role-name my-glue-service-role --policy-name GlueAccess --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "glue:*",
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:ListAllMyBuckets",
                "s3:GetBucketAcl",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeRouteTables",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcAttribute",
                "iam:ListRolePolicies",
                "iam:GetRole",
                "iam:GetRolePolicy",
                "cloudwatch:PutMetricData"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:PutBucketPublicAccessBlock"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*/*",
                "arn:aws:s3:::*/*aws-glue-*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::crawler-public*",
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:AssociateKmsKey"
            ],
            "Resource": [
                "arn:aws:logs:*:*:/aws-glue/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "aws-glue-service-resource"
                    ]
                }
            },
            "Resource": [
                "arn:aws:ec2:*:*:network-interface/*",
                "arn:aws:ec2:*:*:security-group/*",
                "arn:aws:ec2:*:*:instance/*"
            ]
        }
    ]
}'
```

Step 3: S3 Copy source files into S3 buckets

```bash
aws s3 cp data/customer/landing/* s3://stedi-data-lakehouse/customer/landing --recursive
aws s3 cp data/customer/accelerometer/* s3://stedi-data-lakehouse/accelerometer/landing --recursive
aws s3 cp data/customer/stedi_trainer/* s3://stedi-data-lakehouse/stedi_trainer/landing --recursive
```

Step 4. Create Glue Tables

Go to `AWS Athena` and run query in the files in the folder, `glue_table` in your local to create glue tables.

Step 5. Create Glue Jobs

1. Go to `AWS Glue` and click `Notebooks` under  `ETL job` on the left-side menu.
2. Click `Script editor`.
3. Upload a python file in the folder, `glue_job_script` in your local.
4. Go to `job detail` tabe and set glue job name with file name.
5. Set `IAM Role` as `my-glue-service-role`.
6. Save

Step 6. Run the Glue Jobs

Run the glue jobs in the following order.

1. `customer_landing_to_trusted`
2. `customer_trusted_to_curated`
3. `accelerometer_landing_trusted`
4. `step_trainer_landing_to_trusted`
5. `machine_learning_to_trusted`

Step 7. Share Curated Data with Data Science Team.

Share the following s3 location with data science team for their machine learning work.

`s3://stedi-data-lake-house/step_trainer/curated`
