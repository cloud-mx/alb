# AWS Network Load Balancer (NLB) - Main Template

- **Topic**: `templates/loadbalancing/elb-nlb-main.cf-j2.yml`

## Purpose and Summary

This AWS CloudFormation template, enhanced with Jinja2, deploys a **Network Load Balancer (NLB)**. NLBs are designed to handle tens of millions of requests per second while maintaining ultra-low latencies, making them ideal for TCP/UDP-based, performance-critical applications.

This template creates the NLB resource and configures its core attributes, such as cross-zone load balancing.

## Scope

- **In Scope:**
    - Creation of the `AWS::ElasticLoadBalancingV2::LoadBalancer` resource with `Type: network`.
    - Configuration of cross-zone load balancing.
    - Configuration of access logs to an S3 bucket.
- **Out of Scope:**
    - Creation of the VPC, Subnets, or S3 bucket for logs.
    - Creation of Listeners and Target Groups. Use the corresponding modular templates for these.

## Prerequisites

Before deploying this template, ensure you have:
1.  **VPC and Subnet IDs:** An existing VPC and at least one subnet.
2.  **S3 Bucket Name for Logs:** An existing S3 bucket to store access logs (optional but recommended).

## Parameters

### Jinja2 Parameters (`jinjaparams`)

| Name | Type | Description | Example | Optional? |
| :--- | :--- | :--- | :--- | :--- |
| `cross_zone_enabled` | Boolean | If `true`, distributes traffic across all registered targets in all enabled Availability Zones. | `true` | Yes (default: `true`) |
| `deletion_protection` | Boolean | If `true`, prevents the load balancer from being accidentally deleted. | `true` | Yes (default: `false`) |
| `enable_access_logs` | Boolean | If `true`, enables access logging to the specified S3 bucket. | `true` | Yes (default: `true`) |
| `extra_tags` | List | A list of additional tags to apply to the resource. | `[{ "Key": "Service", "Value": "GameServer" }]` | Yes |

### CloudFormation Parameters

| Name | Type | Description | Constraints | Optional? |
| :--- | :--- | :--- | :--- | :--- |
| `UAI` | String | The Unique Application Identifier. | `^uai[0-9]{7}$` | **No** |
| `AppName` | String | A short name for the application. | `^[a-z][a-z0-9-]*`, 3-20 chars | **No** |
| `Env` | String | The deployment environment. | `dev`, `qa`, `stg`, `prd`, `lab` | **No** |
| `Scheme` | String | Defines if the NLB is internal or public. | `internal`, `internet-facing` | **No** |
| `SubnetIds` | `List<AWS::EC2::Subnet::Id>` | A list of Subnet IDs to deploy the NLB into. | - | **No** |
| `AccessLogsS3BucketName`| String | The name of the S3 bucket for access logs. Required if `enable_access_logs` is `true`. | - | Yes |

## Outputs

| Name | Description |
| :--- | :--- |
| `LoadBalancerArn` | ARN of the created Network Load Balancer. |
| `LoadBalancerDNSName` | The DNS name of the Network Load Balancer. |

## Example of Use (`stack_input_nlb.yml`)

```yaml
stacks:
  us-east-1:
    HighPerfNLB:
      template: templates/loadbalancing/elb-nlb-main.cf-j2.yml
      params:
        UAI: "uai3344556"
        AppName: "high-perf-svc"
        Env: "prd"
        Scheme: "internal"
        SubnetIds: ["subnet-0123abc...", "subnet-0456def..."]
        AccessLogsS3BucketName: "gevernova-prod-logs-us-east-1"
      jinjaparams:
        cross_zone_enabled: true
        deletion_protection: true
