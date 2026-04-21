---
name: awsdac
description: Generate AWS architecture diagrams from YAML using awslabs/diagram-as-code (awsdac). Use when the user asks to draw, render, or author an AWS architecture diagram as code, or to convert a CloudFormation template to a diagram.
---

# awsdac — AWS Diagram as Code

CLI tool from awslabs that renders AWS architecture diagrams as PNG from a YAML spec, using the official AWS architecture icon set. Repo: https://github.com/awslabs/diagram-as-code.

Use this skill when the user wants to:
- Author an AWS architecture diagram as YAML
- Render a diagram PNG from an existing DAC file
- Convert a CloudFormation template to a diagram

## Install & invoke

```sh
brew install awsdac                                         # macOS
go install github.com/awslabs/diagram-as-code/cmd/awsdac@latest   # Go 1.21+
```

```sh
awsdac diagram.yaml -o out.png          # render YAML to PNG
awsdac template.yaml -c -o out.png      # [beta] CFN template -> PNG directly
awsdac template.yaml -c -d -o dac.yaml  # [beta] CFN -> editable DAC YAML (preferred for non-trivial)
```

Input can be a local path or URL. Output is **PNG only** (no SVG/PDF). Default output is `output.png`.

Flags worth knowing: `-c` (CFN template), `-d` (emit DAC YAML), `-t` (Go text/template preprocess), `-v` (verbose), `--override-def-file` (swap icon definitions).

## Core concepts

Every file has this shape:

```yaml
Diagram:
  DefinitionFiles:      # required — where icon presets live
    - Type: URL
      Url: https://raw.githubusercontent.com/awslabs/diagram-as-code/main/definitions/definition-for-aws-icons-light.yaml
  Resources:            # required — map of ResourceName -> spec
    Canvas: ...
  Links: []             # optional — arrows
```

**Canvas rule**: exactly one `AWS::Diagram::Canvas`. Resources not reachable from Canvas via `Children` (or `BorderChildren` / `SpanResources`) are silently dropped.

**Type vs Preset**. `Type:` selects a resource class; only container-like AWS types exist natively (`AWS::EC2::VPC`, `AWS::EC2::Subnet`, `AWS::EC2::Instance`, `AWS::EC2::InternetGateway`, `AWS::EC2::NatGateway`, `AWS::EC2::AvailabilityZone`, `AWS::Region`, `AWS::AutoScaling::AutoScalingGroup`, `AWS::ElasticLoadBalancingV2::LoadBalancer`, a few more). For most services, use `AWS::Diagram::Resource` with a `Preset:`:

```yaml
MyLambda:
  Type: AWS::Diagram::Resource
  Preset: "AWS Lambda"
  Title: "Handler"
```

Preset names are case- and space-sensitive — quote names with spaces. See `reference/presets.md` for the common catalog.

**Layout primitives**:
- `Children: [A, B]` — stacks A then B in the parent's `Direction` (`horizontal` default, or `vertical`).
- `AWS::Diagram::HorizontalStack` / `AWS::Diagram::VerticalStack` — invisible grouping to override flow.
- `Align` — cross-axis alignment on a child (`left|center|right|expand` in a vertical parent; `top|center|bottom|expand` in a horizontal parent).
- `BorderChildren: [{Position: N, Resource: IGW}]` — pin a child to a parent's edge (use `IconFill: {Type: rect}` on transparent border icons like IGW).
- `SpanResources: [X, Y]` — overlay (e.g. ASG) wrapping existing resources. Cannot have `Children`, cannot nest, no fill color.

## Minimal working example

```yaml
Diagram:
  DefinitionFiles:
    - Type: URL
      Url: https://raw.githubusercontent.com/awslabs/diagram-as-code/main/definitions/definition-for-aws-icons-light.yaml
  Resources:
    Canvas:
      Type: AWS::Diagram::Canvas
      Children: [AWSCloud]
    AWSCloud:
      Type: AWS::Diagram::Cloud
      Preset: AWSCloudNoLogo
      Children: [EC2]
    EC2:
      Type: AWS::EC2::Instance
```

For a richer example (ALB → EC2 across two subnets with IGW and User), see `examples/alb-ec2.yaml`.

## Links

```yaml
Links:
  - Source: ALB
    SourcePosition: NNW     # 16-wind rose: N NNE NE ENE E ESE SE SSE S SSW SW WSW W WNW NW NNW, or omit for auto
    Target: Instance1
    TargetPosition: SSE
    Type: orthogonal        # default straight; orthogonal = right-angle bends
    TargetArrowHead: { Type: Open }   # Open | Default
    LineStyle: dashed       # normal | dashed
    Labels:
      SourceRight: { Title: "https" }
```

Prefer explicit `N/S/E/W` positions for primary arrows; auto-positioning relies on a lowest common ancestor and can be unstable. See `reference/links.md` for full option list.

## Gotchas

- Exactly one Canvas; unreachable resources are dropped without warning.
- Tabs in YAML produce cryptic "mapping values" errors — use spaces.
- Default `Direction` is `horizontal`. For a VPC stacking subnets above an ALB, set `Direction: vertical` on the VPC and group the subnets in a `HorizontalStack`.
- `Preset:` names live in the definition file and are distinct from `Type:` names. Quote any preset with spaces.
- Transparent icons on a border (IGW, VGW) need `IconFill: {Type: rect}`.
- `AWS::Diagram::Group` is deprecated — use `AWS::Diagram::Resource` or a stack.
- `SpanResources` overlays: no `Children`, no nesting, no fill.
- `UnorderedChildren: true` must be set on every level that should reorder, including the LCA — not just the top.
- CFN conversion (`-c`) is beta. For anything non-trivial, emit DAC YAML with `-c -d`, then edit and re-render.
- First run caches the AWS icon PPTX (~MB). Offline use needs a local `DefinitionFiles` path or pre-populated cache.

## Deeper references

- `reference/resources.md` — full resource attribute table (`Direction`, `Align`, `IconFill`, `BorderType`, `Options.*`, etc.) and the built-in `AWS::Diagram::*` types.
- `reference/links.md` — all link fields, arrowhead shapes, label slots, orthogonal single- vs double-arm rules.
- `reference/presets.md` — common preset names (`AWSCloudNoLogo`, `PublicSubnet`, `PrivateSubnet`, `User`, `Application Load Balancer`, `AWS Lambda`, etc.). For the authoritative list, fetch the definition file URL above.
- `examples/alb-ec2.yaml` — fuller example wiring User → IGW → ALB → two EC2s across subnets.

Fetch the upstream docs (`doc/introduction.md`, `doc/resource-types.md`, `doc/links.md` in the repo) when you need an attribute not listed here.
