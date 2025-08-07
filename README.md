# AWS Application Load Balancer (ALB) - Main Template

- **Topic**: `templates/loadbalancing/elb-alb-main.cf-j2.yml`

## Purpose and Summary

This AWS CloudFormation template, enhanced with Jinja2, deploys a highly configurable and secure AWS **Application Load Balancer (ALB)**. It serves as the core component for any web application architecture and is designed to comply with GE Vernova's Foundational Cloud Security Commandments (GEV FCSC).

The template creates the load balancer, a dedicated security group for it, and optionally, an HTTP to HTTPS redirect listener. **It does not create primary listeners (like port 443) or Target Groups**, as these should be deployed using separate, modular templates to promote reusability.

## Scope

- **In Scope:**
    - Creation of the `AWS::ElasticLoadBalancingV2::LoadBalancer` resource.
    - Creation of a dedicated `AWS::EC2::SecurityGroup` for the ALB.
    - Mandatory association with an **AWS WAF** if the scheme is `internet-facing`.
    - Configuration of access logs.
    - Conditional creation of a port 80 listener to redirect to HTTPS.
- **Out of Scope:**
    - Creation of the VPC, Subnets, WAF WebACL, S3 Bucket for logs, and ACM Certificates. These are prerequisites.
    - Creation of primary Listeners (e.g., HTTPS) and Target Groups. Use the corresponding modular templates for these.

## Prerequisites

Before deploying this template, ensure you have the following:
1.  **VPC and Subnet IDs:** An existing VPC and at least two subnets (preferably private) where the ALB will be deployed.
2.  **WAF Web ACL ARN (if public):** The ARN of an AWS WAF v2 `WebACL` to be associated with the ALB if it's `internet-facing`.
3.  **S3 Bucket Name for Logs:** An existing S3 bucket where the ALB's access logs will be stored.

## Parameters

### Jinja2 Parameters (`jinjaparams`)

These parameters are defined in the input file and control the template's rendering logic.

| Name | Type | Description | Example | Optional? |
| :--- | :--- | :--- | :--- | :--- |
| `idle_timeout_seconds` | Number | The time in seconds that the connection is allowed to be idle. | `60` | Yes (default: `60`) |
| `drop_invalid_headers` | Boolean | If `true`, the ALB drops invalid HTTP headers. | `true` | Yes (default: `true`) |
| `create_http_redirect` | Boolean | If `true`, creates a listener on port 80 that redirects all traffic to HTTPS (port 443). | `true` | Yes (default: `true`) |
| `extra_tags` | List | A list of additional tags to apply to the resources. | `[{ "Key": "CostCenter", "Value": "12345" }]` | Yes |

### CloudFormation Parameters

These are the standard parameters passed to the CloudFormation stack.

| Name | Type | Description | Constraints | Optional? |
| :--- | :--- | :--- | :--- | :--- |
| `UAI` | String | The Unique Application Identifier. | `^uai[0-9]{7}$` | **No** |
| `AppName` | String | A short name for the application. | `^[a-z][a-z0-9-]*`, 3-20 chars | **No** |
| `Env` | String | The deployment environment. | `dev`, `qa`, `stg`, `prd`, `lab` | **No** |
| `Scheme` | String | Defines if the ALB is internal or public. | `internal`, `internet-facing` | **No** |
| `SubnetIds` | `List<AWS::EC2::Subnet::Id>` | A list of Subnet IDs to deploy the ALB into. | - | **No** |
| `WebAclArn` | String | **REQUIRED if `Scheme` is `internet-facing`**. ARN of the WAF Web ACL. | - | Yes (but enforced by condition) |
| `AccessLogsS3BucketName` | String | The name of the S3 bucket for access logs. | - | **No** |

## Outputs

| Name | Description |
| :--- | :--- |
| `LoadBalancerArn` | ARN of the created Application Load Balancer. |
| `LoadBalancerDNSName` | The public or internal DNS name of the load balancer. |
| `CanonicalHostedZoneId` | The canonical hosted zone ID for Route 53 alias records. |
| `LoadBalancerSecurityGroupId` | The ID of the security group created for the load balancer. |

## Example of Use (`stack_input_prd.yml`)

```yaml
stacks:
  us1:
    MyWebAppALB:
      template: templates/loadbalancing/elb-alb-main.cf-j2.yml
      params:
        UAI: "uai1234567"
        AppName: "myapi"
        Env: "prd"
        Scheme: "internet-facing"
        SubnetIds: ["subnet-0123abc...", "subnet-0456def..."]
        WebAclArn: "arn:aws:wafv2:us-east-1:111122223333:regional/webacl/GEV-Standard-Prod/..."
        AccessLogsS3BucketName: "gevernova-central-logs"
      jinjaparams:
        idle_timeout_seconds: 300
        create_http_redirect: true
        extra_tags:
          - Key: "DataClassification"
            Value: "Confidential"
