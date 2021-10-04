---
layout: post
title: AWS Serverless Demo Applications
date: '2021-10-04'
author: Brian
tags: 
- AWS
- Lambda
- Serverless
- API Gateway
- ALB
---

I published a post this summer with a few [simple demo applications](tbd) I use when configuring AWS infrastructure. I needed something similar for a serverless application on AWS. In other words, a Lambda function sitting behind an API Gateway or Application Load Balancer (ALB). 

I tend to use this simple Node.js function. It will return whatever it received as input. This is useful when you are debugging the infrastructure. For example, configuring Cognito with API Gateway or OIDC with ALB.

```js
exports.handler = async (event, context) => {
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "isBase64Encoded": false,
        "body": JSON.stringify(event) 
    };
};
```

Deploying this as a serverless function behind an API Gateway REST API is really easy. Here is a SAM template. 

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31

Resources:
  SimpleFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs14.x
      Timeout: 300
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Method: get
            Path: /
      InlineCode: |
        exports.handler = async (event, context) => {
            return {
                "statusCode": 200,
                "headers": {
                    "Content-Type": "application/json"
                },
                "isBase64Encoded": false,
                "body": JSON.stringify(event) 
            };
        };
```

Deploying an HTTP API is nearly identical. I simply change **ApiEvent** to **HttpApiEvent**.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31

Resources:
  SimpleFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs14.x
      Timeout: 300
      Events:
        HttpApiEvent:
          Type: HttpApi
          Properties:
            Path: /
            Method: GET
      InlineCode: |
        exports.handler = async (event, context) => {
            return {
                "statusCode": 200,
                "headers": {
                    "Content-Type": "application/json"
                },
                "isBase64Encoded": false,
                "body": JSON.stringify(event) 
            };
        };
```

The Application Load Balancer is a little more complicated. First, SAM templates don't support ALB as a event source. Second, the ALB requires you specify subnets and a security group. I use the following template. 

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31

Parameters:
  Subnets:
    Description: Subnets the Application Load Balancer should be deployed in
    Type: List<AWS::EC2::Subnet::Id>

  SecurityGroup:
    Description: Security Group applied to the Application Load Balancer
    Type: AWS::EC2::SecurityGroup::Id

Resources:
  SimpleFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs14.x
      Timeout: 300
      InlineCode: |
        exports.handler = async (event, context) => {
            return {
                "statusCode": 200,
                "headers": {
                    "Content-Type": "application/json"
                },
                "isBase64Encoded": false,
                "body": JSON.stringify(event) 
            };
        };

  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets: !Ref Subnets
      SecurityGroups:
        - !Ref SecurityGroup

  LoadBalancerListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref LoadBalancerTargetGroup

  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt [ SimpleFunction, Arn ] 
      Action: 'lambda:InvokeFunction'
      Principal: elasticloadbalancing.amazonaws.com

  LoadBalancerTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckEnabled: false
      TargetType: lambda
      Targets:
      - Id: !GetAtt [ SimpleFunction, Arn ]
```