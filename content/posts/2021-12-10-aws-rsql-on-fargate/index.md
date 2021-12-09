---
layout: post
title: Running Redshift RSQL is a Fargate container
date: '2021-12-10'
author: Brian
tags: 
- AWS
- Redshift
- ECS
- Docker
---

RSQL is a command-line client for Redshift. Unlike the psql command-line, RSQL has control flow commands (IF, ELSE, GOTO, etc.) that are useful for ETL jobs. I want to run RSQL in a Fargate container so I call it from Step Functions ETL workflow. Overall this was fairly straight forward, but I'll document it anyway.


## Setup 

In my use case, I am converting hundreds of Teradata BTEQ scripts to RSQL using the Schema Conversion Tool (SCT). These scripts have interdependencies and must be in the same the folder. Therefore, I copied them all to an EFS volume. If the scripts were stand alone, I would prefer to put them on S3. 

First, I mounted the EFS volume on my development machine. I'm testing this on a Cloud9 instance.  

```bash
sudo yum install amazon-efs-utils
sudo mount -t efs -o tls fs-12345678:/ /mnt/efs
```

Then, I created **/mnt/efs/odbc.ini** file with a single DSN that uses IAM permissions. I'm going to use a tak role in ECS and I have instance profile associated with development machine. Note that the [RSQL documentation](https://docs.aws.amazon.com/redshift/latest/mgmt/rsql-query-tool-starting-tool-connection.html) is a bit confusing here. The **Instanceprofile=1** option will only work on an EC2 instance. However, if you omit it, RSQL (well, technically the ODBC driver) will use the IAM role on both EC2 and ECS. Honestly, I'm not really sure when you would use Instanceprofile=1. 

```INI
[redshift-cluster-1]
Driver=/opt/amazon/redshiftodbc/lib/64/libamazonredshiftodbc64.so
Host=redshift-cluster-1.123456789012.us-east-1.redshift.amazonaws.com
Database=dev
DbUser=awsuser
ClusterID=redshift-cluster-1
Region=us-east-1
IAM=1
```

For demo purposes, I also created **/mnt/efs/test.rsql** with the following that queries a table in the sample database.

```SQL
SELECT * FROM public.category;

\if :ACTIVITYCOUNT = 0
 \remark '****No data found****'
 \goto LETSQUIT 
\else
 \remark '****Data found****'
 \goto LETSDOSOMETHING 
\endif 

\label LETSQUIT 
\remark '****We are quitting****'
\exit 0 

\label LETSDOSOMETHING 
\remark '****We are doing it****'
\exit 0
```


## Build the Container

Next, I create a Dockerfile that installs the RSQL command line.  Note that the versions are hard coded here and should be updated to the latest versions. Also, note that the environment variables reference paths within the container. I'll override them later to refer to files stored on the EFS volume. I didn;t want to couple the container image to my architecture.     

```Dockerfile
FROM amazonlinux

RUN yum install -y unixODBC wget
RUN wget https://s3.amazonaws.com/redshift-downloads/drivers/odbc/1.4.40.1000/AmazonRedshiftODBC-64-bit-1.4.40.1000-1.x86_64.rpm -O AmazonRedshiftODBC.rpm && yum --nogpgcheck localinstall -y AmazonRedshiftODBC.rpm
RUN wget https://s3.amazonaws.com/redshift-downloads/amazon-redshift-rsql/1.0.1/AmazonRedshiftRsql-1.0.1-1.x86_64.rpm -O AmazonRedshiftRsql.rpm && rpm -i AmazonRedshiftRsql.rpm

ENV ODBCINI=/opt/amazon/redshiftodbc/Setup/odbc.ini
ENV ODBCSYSINI=/opt/amazon/redshiftodbc/Setup
ENV AMAZONREDSHIFTODBCINI=/opt/amazon/redshiftodbc/lib/64/amazon.redshiftodbc.ini

CMD rsql
```

Next, I built the container image. 

```Bash
docker build . -t rsql
```

Then, I tested it locally to make sure it works as expected. In the example below, **-v /mnt/efs:/efs** mounts the EFS volume in the container; **-e ODBCINI=/efs/odbc.ini** overrides the default environment variable to refer to my INI file stored on EFS; **-D redshift-cluster-1** tells rsql which DSN to use; and **-f /efs/test.rsql** tells rsql which file to run.

```bash
docker run -v /mnt/efs:/efs -e ODBCINI=/efs/odbc.ini rsql rsql -D redshift-cluster-1 -f /efs/test.rsql
```

Assuming everything worked as expected, you can push the container image to EFS. 

```bash
docker tag rsql:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/rsql:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/rsql:latest
```


## Configure ECS

To run my container as an ECS task, I need a ECS cluster. I'm going to use Fargate so the cluster configuration is trivial. 

```bash
aws ecs create-cluster --cluster-name rsql --capacity-providers FARGATE
```

In addition, I need a task definition. Note that the command, environment, and mountPoints correspond to the options we pass to **docker run** in the test above. Also note that I have created a ecs-admin role. You, of course, should never run as admin and should create a role with least privilege. 

```json
{
    "family": "rsql",
    "networkMode": "awsvpc",
    "cpu": "512",
    "memory": "1024",
    "taskRoleArn": "arn:aws:iam::123456789012:role/ecs-admin",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecs-admin",
    "containerDefinitions": [
        {
            "name": "rsql",
            "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/rsql:latest",
            "essential": true,
            "cpu": 0,
            "command": [
                "rsql",
                "-D",
                "redshift-cluster-1",
                "-f",
                "/efs/test.rsql"
            ],
            "environment": [
                {
                    "name": "ODBCINI",
                    "value": "/efs/odbc.ini"
                }
            ],
            "mountPoints": [
                {
                    "sourceVolume": "rsql",
                    "containerPath": "/efs"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/rsql",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ],
    "volumes": [
        {
            "name": "rsql",
            "efsVolumeConfiguration": {
                "fileSystemId": "fs-12345678",
                "rootDirectory": "/",
                "transitEncryption": "ENABLED",
                "authorizationConfig": {}
            }
        }
    ]
}
```


## Step Function 

With all the building blocks in place, I can create a Step Functions State Machine that executes my RSQL task. My definition looks like this

```json
{
  "Comment": "Simple state machine that runs RSQL as an ECS Task.",
  "StartAt": "RSQL",
  "States": {
    "RSQL": {
      "Type": "Task",
      "Resource": "arn:aws:states:::ecs:runTask",
      "Parameters": {
        "LaunchType": "FARGATE",
        "Cluster": "arn:aws:ecs:us-east-1:123456789012:cluster/rsql",
        "TaskDefinition": "arn:aws:ecs:us-east-1:123456789012:task-definition/rsql",
        "NetworkConfiguration": {
          "AwsvpcConfiguration": {
            "Subnets": [
              "subnet-11111111",
              "subnet-22222222"
            ],
            "securityGroups": [
              "sg-11111111"
            ]
            "AssignPublicIp": "ENABLED"
          }
        },
        "Overrides": {
          "ContainerOverrides": [
            {
              "Name": "rsql",
              "Command.$": "$.commands"
            }
          ]
        }
      },
      "End": true
    }
  }
}
```

Finally, notice the **$.commands** variable in the container override. This allows me to specify the RSQL command line options as an input the Step Functions state. For example, the following is identical the prior examples I have been using. 

```json
{
  "commands": [
    "rsql",
    "-D",
    "redshift-cluster-1",
    "-f",
    "/efs/test.rsql"
  ]
}
```

There you go. You can simple add that Step Functions state to your ETL workflow to call out to RSQL.  