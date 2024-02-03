# STEDI Human Balance Analytics

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
