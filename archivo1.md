
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
```

# parametros
```yaml
# This is an example input file for deploying the main Application Load Balancer template.
# It defines two separate stacks: one for a 'dev' environment and one for a 'prd' environment.

stacks:
  # Deployment for the us-east-1 region
  us-east-1:
    
    # ----------------------------------------------------------------
    # DEVELOPMENT STACK: An internal-facing ALB for a backend API
    # ----------------------------------------------------------------
    DevApiALB:
      template: templates/loadbalancing/elb-alb-main.cf-j2.yml
      params:
        # --> GEV Standard Parameters
        UAI: "uai9876543"
        AppName: "backend-api"
        Env: "dev"
        
        # --> Networking and Security Parameters
        Scheme: "internal" # This ALB is not exposed to the internet
        SubnetIds: 
          - "subnet-0abcdef123456789" # Private subnet 1
          - "subnet-0fedcba987654321" # Private subnet 2
        VpcId: "vpc-0a1b2c3d4e5f67890"
        
        # WebAclArn is left empty because the scheme is 'internal'
        WebAclArn: "" 
        
        # --> Logging Parameter
        AccessLogsS3BucketName: "gevernova-dev-logs"

      jinjaparams:
        # --> Jinja2 controlled settings
        idle_timeout_seconds: 60       # Default idle timeout for dev
        drop_invalid_headers: true
        create_http_redirect: false    # No redirect needed for an internal API
        extra_tags:
          - Key: "Owner"
            Value: "api-dev-team"
          - Key: "Project"
            Value: "ProjectPhoenix"

    # ----------------------------------------------------------------
    # PRODUCTION STACK: A public-facing ALB for a web application
    # ----------------------------------------------------------------
    ProdWebAppALB:
      template: templates/loadbalancing/elb-alb-main.cf-j2.yml
      params:
        # --> GEV Standard Parameters
        UAI: "uai1234567"
        AppName: "webapp-frontend"
        Env: "prd"

        # --> Networking and Security Parameters
        Scheme: "internet-facing" # This ALB is public
        SubnetIds: 
          - "subnet-0111222333444555" # Public subnet 1
          - "subnet-0666777888999000" # Public subnet 2
        VpcId: "vpc-0a1b2c3d4e5f67890"

        # WebAclArn is MANDATORY for internet-facing ALBs to ensure security compliance
        WebAclArn: "arn:aws:wafv2:us-east-1:111122223333:regional/webacl/GEV-Standard-Prod-WAF/a1b2c3d4-e5f6-7890-g1h2-i3j4k5l6m7n8"
        
        # --> Logging Parameter
        AccessLogsS3BucketName: "gevernova-prod-logs-us-east-1"

      jinjaparams:
        # --> Jinja2 controlled settings
        idle_timeout_seconds: 400       # Longer timeout for potentially long-running web sessions
        drop_invalid_headers: true
        create_http_redirect: true     # Enforce HTTPS by redirecting HTTP traffic
        extra_tags:
          - Key: "DataClassification"
            Value: "Public"
          - Key: "CostCenter"
            Value: "54321"
```
