# Resource reference

## Built-in `AWS::Diagram::*` types

| Type | Purpose |
| --- | --- |
| `AWS::Diagram::Canvas` | Top-level drawable area. Exactly one per file. |
| `AWS::Diagram::Cloud` | "AWS Cloud" group. Use `Preset: AWSCloudNoLogo` to suppress the logo. |
| `AWS::Diagram::Resource` | Generic resource — the carrier for most service presets and for `User`/custom icons. |
| `AWS::Diagram::HorizontalStack` | Invisible group, children arranged left-to-right. |
| `AWS::Diagram::VerticalStack` | Invisible group, children arranged top-to-bottom. |
| `AWS::Diagram::DataCenter` | On-premises data center group. |
| `AWS::Diagram::Account` | AWS account group. |
| `AWS::Diagram::Group` | **Deprecated** — do not use. |

## Native AWS container-ish types

These have first-class `Type:` entries (most others are presets on `AWS::Diagram::Resource`):

- `AWS::Region`
- `AWS::EC2::VPC`
- `AWS::EC2::Subnet`
- `AWS::EC2::AvailabilityZone`
- `AWS::EC2::Instance`
- `AWS::EC2::InternetGateway`
- `AWS::EC2::NatGateway`
- `AWS::EC2::NetworkInterface`
- `AWS::AutoScaling::AutoScalingGroup`
- `AWS::ElasticLoadBalancingV2::LoadBalancer`

For everything else (Lambda, S3, DynamoDB, SQS, API Gateway, etc.) use `AWS::Diagram::Resource` + `Preset:`.

## Resource attributes

| Attribute | Type | Default | Notes |
| --- | --- | --- | --- |
| `Type` | string | — | **Required.** |
| `Preset` | string | `""` | Named entry from the definition file. Quote names with spaces. |
| `Icon` | string | `""` | Path to a custom icon (overrides preset). |
| `IconFill` | object | `{Type: none}` | Set `{Type: rect, Color: "rgba(...)"}` to give a transparent icon a solid backing. Needed for border children. |
| `Direction` | string | `horizontal` | `horizontal` or `vertical` — how `Children` stack. |
| `Align` | string | `center` | In a vertical parent: `left|center|right|expand`. In a horizontal parent: `top|center|bottom|expand`. |
| `FillColor` | string | transparent | Groups only. |
| `BorderColor` | string | transparent | |
| `BorderType` | string | `Straight` | `Straight` or `Dashed`. |
| `Title` | string | `""` | Label shown on the resource. |
| `TitleFillColor` | string | transparent | |
| `HeaderAlign` | string | `left` | Groups only. `left|center|right`. |
| `Children` | []string | `[]` | Names of child resources stacked inside. |
| `BorderChildren` | []object | `[]` | `[{Position: N|S|E|W, Resource: <name>}]`. Pins a child to the parent's edge. |
| `SpanResources` | []string | `[]` | Overlay that wraps listed resources (e.g. ASG). Mutually exclusive with `Children`. Cannot nest, no fill color. |
| `Options.UnorderedChildren` | bool | `false` | Reorders children based on link endpoints to reduce crossings. Must be set at every level you want reordered (the LCA and each intermediate parent), not just one. |
| `Options.GroupingOffset` | bool | `false` | Spreads multiple links from the same edge apart (±5/±10 px). |
| `Options.GroupingOffsetDirection` | bool | `false` | Also groups offset links by target direction. |

## Common patterns

### Transparent border icon

```yaml
VPC:
  Type: AWS::EC2::VPC
  BorderChildren:
    - Position: S
      Resource: IGW
IGW:
  Type: AWS::EC2::InternetGateway
  IconFill:
    Type: rect
```

### Subnets side-by-side inside a vertically stacked VPC

```yaml
VPC:
  Type: AWS::EC2::VPC
  Direction: vertical
  Children: [SubnetsRow, ALB]
SubnetsRow:
  Type: AWS::Diagram::HorizontalStack
  Children: [SubnetA, SubnetB]
```

### ASG wrapping existing instances

```yaml
ASG:
  Type: AWS::AutoScaling::AutoScalingGroup
  SpanResources: [Instance1, Instance2]
  BorderType: Dashed
```
