# nlb-security-prereqs.cf-j2.yml
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'GEV NLB Security Prerequisites - Security Groups, S3 Access Logs Bucket, and WAF Configuration'

Parameters:
  UAI:
    Type: String
    Description: Unique Application Identifier
    AllowedPattern: '^uai[0-9]{7}$'
    ConstraintDescription: Must start with uai followed by 7 digits
  Env:
    Type: String
    Description: Deployment Environment
    AllowedValues: ['dev', 'qa', 'prd', 'lab', 'stg', 'dr']
    Default: 'dev'
  AppName:
    Type: String
    Description: Application Name (lowercase, 3-20 chars)
    AllowedPattern: '^[a-z][a-z0-9\-]*[a-z0-9]$'
    MinLength: 3
    MaxLength: 20
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID for security groups
  NLBScheme:
    Type: String
    Description: NLB Scheme (internet-facing or internal)
    AllowedValues: ['internet-facing', 'internal']
    Default: 'internet-facing'
  AllowedIngressCidr:
    Type: String
    Description: CIDR block for NLB ingress (use 0.0.0.0/0 for internet-facing)
    Default: '0.0.0.0/0'
  NLBPorts:
    Type: CommaDelimitedList
    Description: List of ports the NLB will expose
    Default: '80,443'
  TargetPorts:
    Type: CommaDelimitedList
    Description: List of ports for targets
    Default: '80'
  AccessLogsBucketName:
    Type: String
    Description: Optional custom name for S3 access logs bucket
    Default: ''

Conditions:
  IsInternetFacing: !Equals [!Ref NLBScheme, 'internet-facing']
  HasCustomBucketName: !Not [!Equals [!Ref AccessLogsBucketName, '']]

Resources:
  # Security Group for NLB
  NLBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${UAI}-${AppName}-${Env}-nlb-sg'
      GroupDescription: Security group for NLB
      VpcId: !Ref VpcId
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: appname
          Value: !Ref AppName
        - Key: Name
          Value: !Sub '${UAI}-${AppName}-${Env}-nlb-sg'

  # Security Group for Targets
  TargetSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${UAI}-${AppName}-${Env}-target-sg'
      GroupDescription: Security group for NLB targets
      VpcId: !Ref VpcId
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: appname
          Value: !Ref AppName
        - Key: Name
          Value: !Sub '${UAI}-${AppName}-${Env}-target-sg'

  # Ingress rules for NLB Security Group
  NLBIngressRules:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref NLBSecurityGroup
      IpProtocol: tcp
      FromPort: !Select [0, !Ref NLBPorts]
      ToPort: !Select [0, !Ref NLBPorts]
      CidrIp: !Ref AllowedIngressCidr
    DependsOn: NLBSecurityGroup

  # Ingress rules for Target Security Group from NLB
  TargetIngressRules:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref TargetSecurityGroup
      IpProtocol: tcp
      FromPort: !Select [0, !Ref TargetPorts]
      ToPort: !Select [0, !Ref TargetPorts]
      SourceSecurityGroupId: !Ref NLBSecurityGroup
    DependsOn: TargetSecurityGroup

  # S3 Bucket for Access Logs
  AccessLogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !If 
        - HasCustomBucketName
        - !Ref AccessLogsBucketName
        - !Sub '${UAI}-${AppName}-${Env}-nlb-access-logs-${AWS::AccountId}-${AWS::Region}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: LogsExpiration
            Status: Enabled
            ExpirationInDays: 365
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        IgnorePublicAcls: true
        BlockPublicPolicy: true
        RestrictPublicBuckets: true
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: appname
          Value: !Ref AppName
        - Key: Name
          Value: !Sub '${UAI}-${AppName}-${Env}-nlb-access-logs'

  # S3 Bucket Policy for ELB Logging
  AccessLogsBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref AccessLogsBucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: delivery.logs.amazonaws.com
            Action: s3:PutObject
            Resource: !Sub '${AccessLogsBucket.Arn}/AWSLogs/${AWS::AccountId}/*'
            Condition:
              StringEquals:
                s3:x-amz-acl: bucket-owner-full-control
          - Effect: Allow
            Principal:
              Service: delivery.logs.amazonaws.com
            Action: s3:GetBucketAcl
            Resource: !Sub '${AccessLogsBucket.Arn}'
    DependsOn: AccessLogsBucket

Outputs:
  NLBSecurityGroupId:
    Description: Security Group ID for NLB
    Value: !Ref NLBSecurityGroup
    Export:
      Name: !Sub '${UAI}-${AppName}-${Env}-nlb-sg-id'
  TargetSecurityGroupId:
    Description: Security Group ID for Targets
    Value: !Ref TargetSecurityGroup
    Export:
      Name: !Sub '${UAI}-${AppName}-${Env}-target-sg-id'
  AccessLogsBucketName:
    Description: S3 Bucket Name for Access Logs
    Value: !Ref AccessLogsBucket
    Export:
      Name: !Sub '${UAI}-${AppName}-${Env}-access-logs-bucket'
```
# nlb-security-prereqs-params.yml
```yaml
# AWS NLB Security Prerequisites Parameters
# Template: nlb-security-prereqs.cf-j2.yml

stacks:
  us1:
    NlbSecurityPrereqs:
      template: templates/nlb/nlb-security-prereqs.cf-j2.yml
      params:
        UAI: "uai3055511"  # Replace with actual UAI
        Env: "dev"         # Environment: dev, qa, prd, lab, stg, dr
        AppName: "webapp"  # Application name (3-20 chars, lowercase)
        VpcId: "vpc-12345678"  # Replace with actual VPC ID
        NLBScheme: "internet-facing"  # or "internal"
        AllowedIngressCidr: "0.0.0.0/0"  # For internet-facing, use 0.0.0.0/0; for internal, use VPC CIDR
        NLBPorts: "80,443"  # Comma-separated list of ports for NLB
        TargetPorts: "80"   # Comma-separated list of ports for targets
        AccessLogsBucketName: ""  # Leave empty for auto-generated name

      jinjaparams:
        # No additional jinja parameters needed for this template
```

