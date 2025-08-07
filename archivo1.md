
---

### 2. Template: `elb-alb-main.cf-j2.yml` (English)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  GEV - Application Load Balancer (ALB) - Main Template.
  Creates the ALB, its Security Group, and logging configuration.
  Requires WAF for public ALBs.

Parameters:
  UAI:
    Type: String
    Description: The Unique Application Identifier (UAI).
    AllowedPattern: '^uai[0-9]{7}$'
    ConstraintDescription: Must be a valid UAI, e.g., uai1234567.
  AppName:
    Type: String
    Description: Short name for the application (used in resource names).
    MinLength: 3
    MaxLength: 20
    AllowedPattern: '^[a-z][a-z0-9-]*$'
    ConstraintDescription: Must contain only lowercase letters, numbers, and hyphens.
  Env:
    Type: String
    Description: Deployment environment.
    AllowedValues: ['dev', 'qa', 'stg', 'prd', 'lab']
    ConstraintDescription: Must be one of the allowed environments.
  Scheme:
    Type: String
    Description: Defines the load balancer's scheme.
    AllowedValues: ['internal', 'internet-facing']
    Default: 'internal'
  SubnetIds:
    Type: 'List<AWS::EC2::Subnet::Id>'
    Description: List of Subnet IDs for the ALB. A minimum of 2 is recommended for high availability.
  WebAclArn:
    Type: String
    Description: ARN of the AWS WAF Web ACL. Required for 'internet-facing' ALBs.
    Default: ''
  AccessLogsS3BucketName:
    Type: String
    Description: Name of the S3 bucket to store access logs.
  VpcId:
    Type: 'AWS::EC2::VPC::Id'
    Description: The VPC ID where the security group will be created.

Conditions:
  IsProd: !Equals [!Ref Env, 'prd']
  IsInternetFacing: !Equals [!Ref Scheme, 'internet-facing']
  HasWafAcl: !Not [!Equals [!Ref WebAclArn, '']]
  CreateHttpRedirect: !Equals ['{{ instance.get("create_http_redirect", true) }}', true]

Rules:
  # This rule stops the deployment if a public ALB does not have a WAF ARN.
  RequireWafForPublicAlb:
    Assertions:
      - Assert: !Or [!Not [!Condition IsInternetFacing], !Condition HasWafAcl]
        AssertDescription: "GEV-FCSC-3 Compliance Error: A valid WebAclArn is mandatory for internet-facing ALBs."

Resources:
  # Security Group for the load balancer itself
  LoadBalancerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${UAI}-${AppName}-${Env}-alb-sg'
      GroupDescription: !Sub 'Security group for ALB ${AppName} in ${Env}'
      VpcId: !Ref VpcId
      # The SecurityGroupIngress is intentionally minimal.
      # The HTTP redirect listener adds its rule below.
      # The main HTTPS listener will add its rule from its own template.
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: app
          Value: !Ref AppName

  # The main resource: Application Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${UAI}-${AppName}-${Env}-alb'
      Scheme: !Ref Scheme
      Type: 'application'
      Subnets: !Ref SubnetIds
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
      LoadBalancerAttributes:
        - Key: 'deletion_protection.enabled'
          Value: !If [IsProd, 'true', 'false']
        - Key: 'logging.s3.enabled'
          Value: 'true'
        - Key: 'logging.s3.bucket'
          Value: !Ref AccessLogsS3BucketName
        - Key: 'logging.s3.prefix'
          Value: !Sub 'elb-logs/${UAI}/${AWS::Region}'
        - Key: 'idle_timeout.seconds'
          Value: '{{ instance.get("idle_timeout_seconds", 60) }}'
        - Key: 'routing.http.drop_invalid_header_fields.enabled'
          Value: '{{ instance.get("drop_invalid_headers", true) }}'
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: app
          Value: !Ref AppName
        - Key: Name
          Value: !Sub '${UAI}-${AppName}-${Env}-alb'
        {% for tag in instance.get('extra_tags', []) %}
        - Key: '{{ tag.Key }}'
          Value: '{{ tag.Value }}'
        {% endfor %}

  # Mandatory WAF association if public
  WAFAssociation:
    Type: AWS::WAFv2::WebACLAssociation
    Condition: IsInternetFacing
    Properties:
      ResourceArn: !Ref ApplicationLoadBalancer
      WebACLArn: !Ref WebAclArn

  # Conditional listener to redirect HTTP to HTTPS
  HttpRedirectListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Condition: CreateHttpRedirect
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: 'redirect'
          RedirectConfig:
            Protocol: 'HTTPS'
            Port: '443'
            StatusCode: 'HTTP_301'

  # Ingress rule for the redirect listener
  HttpRedirectIngressRule:
    Type: AWS::EC2::SecurityGroupIngress
    Condition: CreateHttpRedirect
    Properties:
      GroupId: !Ref LoadBalancerSecurityGroup
      IpProtocol: tcp
      FromPort: 80
      ToPort: 80
      CidrIp: 0.0.0.0/0

Outputs:
  LoadBalancerArn:
    Description: The ARN of the Application Load Balancer.
    Value: !Ref ApplicationLoadBalancer
    Export:
      Name: !Sub '${AWS::StackName}-LoadBalancerArn'

  LoadBalancerDNSName:
    Description: The DNS name of the Application Load Balancer.
    Value: !GetAtt ApplicationLoadBalancer.DNSName
    Export:
      Name: !Sub '${AWS::StackName}-DNSName'

  CanonicalHostedZoneId:
    Description: The canonical hosted zone ID of the ALB for Route 53 alias records.
    Value: !GetAtt ApplicationLoadBalancer.CanonicalHostedZoneID
    Export:
      Name: !Sub '${AWS::StackName}-CanonicalHostedZoneId'

  LoadBalancerSecurityGroupId:
    Description: The ID of the Security Group created for the ALB.
    Value: !Ref LoadBalancerSecurityGroup
    Export:
      Name: !Sub '${AWS::StackName}-SecurityGroupId'
