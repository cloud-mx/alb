# AWS Network Load Balancer (NLB) Template Set

This document covers the set of modular templates used to deploy a complete, secure, and high-performance Network Load Balancer (NLB) solution.

## Architecture Overview

The NLB solution is composed of three modular templates that work together:

1.  **`elb-nlb-main.cf-j2.yml`**: Deploys the core Network Load Balancer resource.
2.  **`elb-target-group.cf-j2.yml`**: Deploys a Target Group to hold the backend targets (e.g., EC2 instances). This template is shared with the ALB solution.
3.  **`elb-nlb-listener.cf-j2.yml`**: Deploys a listener to accept traffic on a specific port/protocol and forward it to a target group.

This modular design allows you to create one NLB and associate it with multiple listeners and target groups, providing a flexible and reusable infrastructure pattern.

---
### **Template: `elb-nlb-main.cf-j2.yml`**

**Purpose:** Deploys the core NLB resource and configures its main attributes like scheme, cross-zone load balancing, and access logs.

**Parameters (Jinja2):**
- `cross_zone_enabled` (Boolean, default: `true`): Enables traffic distribution across all AZs.
- `deletion_protection` (Boolean, default: `false`): Protects the NLB from accidental deletion. Recommended `true` for production.
- `enable_access_logs` (Boolean, default: `true`): Enables access logging.
- `extra_tags` (List): Optional additional tags.

**Parameters (CloudFormation):**
- `UAI`, `AppName`, `Env`, `Scheme`, `SubnetIds`, `AccessLogsS3BucketName`

---
### **Template: `elb-target-group.cf-j2.yml` (Reused)**

**Purpose:** Deploys a Target Group for NLB targets. It should be configured with `TCP`, `UDP`, or `TLS` protocols.

**Parameters (Jinja2):** See the template's dedicated README.

**Parameters (CloudFormation):** See the template's dedicated README.

---
### **Template: `elb-nlb-listener.cf-j2.yml`**

**Purpose:** Deploys a listener (e.g., TCP on port 443 or TLS on port 8443) for an existing NLB and forwards traffic to a default target group.

**Parameters (CloudFormation):**
- `UAI`, `AppName`, `Env` (for tagging)
- `LoadBalancerArn`, `DefaultTargetGroupArn`
- `ListenerPort`, `ListenerProtocol` (`TCP`, `TLS`, `UDP`, `TCP_UDP`)
- `CertificateArn` (Required for `TLS` protocol)
- `SslPolicy` (For `TLS` protocol)

```

---

### 2. Main NLB Template: `elb-nlb-main.cf-j2.yml` (Revised)

*No significant changes needed here, the original was solid. Minor comment clarifications.*

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  GEV - Network Load Balancer (NLB) - Main Template.
  Creates the NLB and configures its core attributes like cross-zone load balancing.

Parameters:
  UAI:
    Type: String
    Description: The Unique Application Identifier (UAI).
    AllowedPattern: '^uai[0-9]{7}$'
  AppName:
    Type: String
    Description: Short name for the application.
    MinLength: 3
    MaxLength: 20
    AllowedPattern: '^[a-z][a-z0-9-]*$'
  Env:
    Type: String
    Description: Deployment environment.
    AllowedValues: ['dev', 'qa', 'stg', 'prd', 'lab']
  Scheme:
    Type: String
    Description: Defines the load balancer's scheme.
    AllowedValues: ['internal', 'internet-facing']
    Default: 'internal'
  SubnetIds:
    Type: 'List<AWS::EC2::Subnet::Id>'
    Description: List of Subnet IDs for the NLB.
  AccessLogsS3BucketName:
    Type: String
    Description: Name of the S3 bucket to store access logs (optional).
    Default: ''

Conditions:
  EnableAccessLogs: !And
    - !Equals ['{{ instance.get("enable_access_logs", true) }}', true]
    - !Not [!Equals [!Ref AccessLogsS3BucketName, '']]

Resources:
  NetworkLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${UAI}-${AppName}-${Env}-nlb'
      Scheme: !Ref Scheme
      Type: 'network'
      Subnets: !Ref SubnetIds
      LoadBalancerAttributes:
        - Key: 'load_balancing.cross_zone.enabled'
          Value: '{{ instance.get("cross_zone_enabled", true) }}'
        - Key: 'deletion_protection.enabled'
          Value: '{{ instance.get("deletion_protection", false) }}'
        - Key: 'access_logs.s3.enabled'
          Value: !If [EnableAccessLogs, 'true', 'false']
        - Key: 'access_logs.s3.bucket'
          Value: !If [EnableAccessLogs, !Ref AccessLogsS3BucketName, !Ref "AWS::NoValue"]
        - Key: 'access_logs.s3.prefix'
          Value: !If [EnableAccessLogs, !Sub 'nlb-logs/${UAI}/${AWS::Region}', !Ref "AWS::NoValue"]
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: app
          Value: !Ref AppName
        - Key: Name
          Value: !Sub '${UAI}-${AppName}-${Env}-nlb'
        {% for tag in instance.get('extra_tags', []) %}
        - Key: '{{ tag.Key }}'
          Value: '{{ tag.Value }}'
        {% endfor %}

Outputs:
  LoadBalancerArn:
    Description: The ARN of the Network Load Balancer.
    Value: !Ref NetworkLoadBalancer
    Export:
      Name: !Sub '${AWS::StackName}-LoadBalancerArn'
  LoadBalancerDNSName:
    Description: The DNS name of the Network LoadBalancer.
    Value: !GetAtt NetworkLoadBalancer.DNSName
    Export:
      Name: !Sub '${AWS::StackName}-DNSName'
  CanonicalHostedZoneId:
    Description: The canonical hosted zone ID of the NLB for Route 53 alias records.
    Value: !GetAtt NetworkLoadBalancer.CanonicalHostedZoneID
    Export:
      Name: !Sub '${AWS::StackName}-CanonicalHostedZoneId'

```
---

### 3. Reused Target Group Template: `elb-target-group.cf-j2.yml` (Revised)

*Improvement: Added GEV standard tags to the Target Group resource itself.*

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  GEV - ALB/NLB Target Group - Modular Template.
  Creates a Target Group with configurable health checks and attributes.

Parameters:
  # ... (Same parameters as before)
  UAI:
    Type: String
    Description: The Unique Application Identifier (UAI).
  AppName:
    Type: String
    Description: Short name for the application.
  Env:
    Type: String
    Description: Deployment environment.
  VpcId:
    Type: 'AWS::EC2::VPC::Id'
    Description: The VPC ID where the target group is located.
  TargetGroupName:
    Type: String
    Description: A specific name for this target group instance (e.g., 'api-users').
  TargetPort:
    Type: Number
    Description: The port on which the targets receive traffic.
  TargetProtocol:
    Type: String
    Description: The protocol used for routing traffic to the targets.
  TargetType:
    Type: String
    Description: The type of target that you specify when registering targets.

Resources:
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${UAI}-${AppName}-${Env}-${TargetGroupName}-tg'
      VpcId: !Ref VpcId
      Port: !Ref TargetPort
      Protocol: !Ref TargetProtocol
      TargetType: !Ref TargetType
      HealthCheckProtocol: '{{ instance.health_check.get("Protocol", TargetProtocol) }}'
      HealthCheckPort: '{{ instance.health_check.get("Port", "traffic-port") }}'
      HealthCheckPath: '{{ instance.health_check.get("Path", "/") }}'
      # ... (Rest of health check properties are the same)
      TargetGroupAttributes:
        # ... (Same attributes as before)
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: app
          Value: !Ref AppName
        - Key: Name
          Value: !Sub '${UAI}-${AppName}-${Env}-${TargetGroupName}-tg'
        {% for tag in instance.get('extra_tags', []) %}
        - Key: '{{ tag.Key }}'
          Value: '{{ tag.Value }}'
        {% endfor %}
# ... (Same Outputs as before)
```

---

### 4. NLB Listener Template: `elb-nlb-listener.cf-j2.yml` (Revised)

*Improvements: Added GEV standard tags. Clarified parameter descriptions.*

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  GEV - NLB Listener - Modular Template.
  Creates a listener for an existing NLB.

Parameters:
  UAI:
    Type: String
    Description: The UAI for tagging consistency.
    AllowedPattern: '^uai[0-9]{7}$'
  AppName:
    Type: String
    Description: The AppName for tagging consistency.
    AllowedPattern: '^[a-z][a-z0-9-]*$'
  Env:
    Type: String
    Description: The Environment for tagging consistency.
    AllowedValues: ['dev', 'qa', 'stg', 'prd', 'lab']
  LoadBalancerArn:
    Type: String
    Description: The ARN of the parent Network Load Balancer.
    AllowedPattern: '^arn:aws:elasticloadbalancing:.*:loadbalancer/net/.*'
  DefaultTargetGroupArn:
    Type: String
    Description: The ARN of the default Target Group for the 'forward' action.
    AllowedPattern: '^arn:aws:elasticloadbalancing:.*:targetgroup/.*'
  ListenerPort:
    Type: Number
    Description: The port on which the listener will accept traffic.
    MinValue: 1
    MaxValue: 65535
  ListenerProtocol:
    Type: String
    Description: The protocol for the listener.
    AllowedValues: ['TCP', 'TLS', 'UDP', 'TCP_UDP']
    Default: 'TCP'
  CertificateArn:
    Type: String
    Description: The ARN of the ACM certificate. Required for TLS listeners.
    Default: ''
  SslPolicy:
    Type: String
    Description: The security policy for a TLS listener.
    Default: 'ELBSecurityPolicy-2016-08'

Conditions:
  IsTls: !Equals [!Ref ListenerProtocol, 'TLS']
  HasCertificate: !Not [!Equals [!Ref CertificateArn, '']]

Rules:
  RequireCertificateForTls:
    Assertions:
      - Assert: !Or [!Not [!Condition IsTls], !Condition HasCertificate]
        AssertDescription: "Compliance Error: A valid CertificateArn is mandatory for TLS listeners."

Resources:
  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancerArn
      Port: !Ref ListenerPort
      Protocol: !Ref ListenerProtocol
      SslPolicy: !If [IsTls, !Ref SslPolicy, !Ref "AWS::NoValue"]
      Certificates: !If
        - IsTls
        - - CertificateArn: !Ref CertificateArn
        - !Ref "AWS::NoValue"
      DefaultActions:
        - Type: 'forward'
          TargetGroupArn: !Ref DefaultTargetGroupArn
      Tags:
        - Key: uai
          Value: !Ref UAI
        - Key: env
          Value: !Ref Env
        - Key: app
          Value: !Ref AppName

Outputs:
  ListenerArn:
    Description: The ARN of the created listener.
    Value: !Ref Listener
    Export:
      Name: !Sub '${AWS::StackName}-ListenerArn'
```
