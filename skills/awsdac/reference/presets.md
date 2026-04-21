# Preset reference

Presets are named icon entries in the definition file. Use them on `AWS::Diagram::Resource` (or any resource type) via `Preset: "<name>"`. Names are case- and space-sensitive — quote names with spaces or parentheses.

For the authoritative catalog, fetch:
https://raw.githubusercontent.com/awslabs/diagram-as-code/main/definitions/definition-for-aws-icons-light.yaml

## Non-service presets

| Preset | Use |
| --- | --- |
| `AWSCloudNoLogo` | `AWS::Diagram::Cloud` without the AWS logo in the corner |
| `PublicSubnet` | `AWS::EC2::Subnet` styled as public |
| `PrivateSubnet` | `AWS::EC2::Subnet` styled as private |
| `User` | Person icon on `AWS::Diagram::Resource` |
| `Generic group` | Neutral grouping box |
| `Bucket with objects` | S3 bucket with object icons |
| `VPC` | VPC styling (use `AWS::EC2::VPC` type directly when possible) |
| `AMI` | AMI icon |
| `Application Load Balancer` | ALB styling on an ELBv2 |
| `Network Load Balancer` | NLB styling |

## Service presets

Most AWS services follow one of two naming patterns in the definition file:

- `"AWS <Service>"` — e.g. `"AWS Lambda"`, `"AWS Batch"`, `"AWS Amplify"`, `"AWS CloudFormation"`, `"AWS Step Functions"`, `"AWS Glue"`, `"AWS Fargate"`, `"AWS WAF"`, `"AWS Shield"`
- `"Amazon <Service>"` — e.g. `"Amazon S3"`, `"Amazon DynamoDB"`, `"Amazon SQS"`, `"Amazon SNS"`, `"Amazon API Gateway"`, `"Amazon CloudFront"`, `"Amazon Route 53"`, `"Amazon Cognito"`, `"Amazon EventBridge"`, `"Amazon Kinesis"`

When unsure of the exact name, grep the definition file for the service. Example:

```yaml
Lambda:
  Type: AWS::Diagram::Resource
  Preset: "AWS Lambda"
  Title: "Process order"
Table:
  Type: AWS::Diagram::Resource
  Preset: "Amazon DynamoDB"
  Title: "orders"
```

## Local or custom definitions

To use a local copy (for offline builds or custom icons):

```yaml
DefinitionFiles:
  - Type: LocalFile
    LocalFile: ./definitions/definition-for-aws-icons-light.yaml
```

Or override at invocation: `awsdac diagram.yaml --override-def-file ./my-defs.yaml`.
