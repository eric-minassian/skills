---
name: cdk
description: Rules and conventions for writing, reviewing, or modifying AWS CDK TypeScript infrastructure code
---

# AWS CDK (TypeScript) — Rules and Conventions

Follow these rules when writing, reviewing, or modifying AWS CDK TypeScript code.

---

## Project Structure

```
project/
├── bin/
│   └── app.ts                       # App entry point — composition ONLY
├── lib/
│   ├── stacks/
│   │   ├── foundation-stack.ts      # VPC, DynamoDB, S3, Cognito (rarely changes)
│   │   └── app-stack.ts             # Lambda, API GW, CloudFront (every deploy)
│   ├── constructs/                  # Custom L2 wrappers and L3 patterns
│   └── stages/                      # cdk.Stage subclasses for pipelines
├── config/
│   ├── types.ts                     # Shared EnvironmentConfig interface
│   ├── dev.ts
│   ├── staging.ts
│   └── prod.ts
├── src/                             # Runtime code (Lambda handlers)
├── test/                            # Mirrors lib/ structure
├── cdk.json
└── tsconfig.json
```

- `bin/app.ts` instantiates stacks/stages and wires cross-stack references. Zero resource definitions.
- `lib/` holds all infrastructure code, subdivided into `stacks/`, `constructs/`, `stages/`.
- `src/` holds runtime application code, separate from infrastructure.
- `test/` mirrors `lib/` structure.
- Keep CDK infra and application code in the same repo (mono-repo) unless a shared construct library needs its own package.

---

## Stacks

### Start with one stack. Split only when there is a concrete reason.

Most projects have 20-80 resources, well under the CloudFormation 500-resource limit. Do not preemptively split into many stacks.

A single stack gives you atomic deployments, zero CloudFormation export locks, simple rollbacks, and less cognitive overhead.

### When to split: the two-stack pattern

Split by **rate of change** when the project grows:

1. **Foundation stack** — VPC, DynamoDB, S3, Cognito, RDS. Deployed rarely. Termination protection enabled.
2. **Application stack** — Lambda, API Gateway, CloudFront, Step Functions, CloudWatch, IAM roles for compute. Changes every deploy.

Keep the cross-stack surface small (3-5 references: table ARN, bucket name, VPC ID, etc.).

### When to split further

Only for concrete reasons:
- Multi-account deployments
- Team ownership boundaries
- Hitting the 500-resource limit
- Significantly different deployment cadences at scale

### Cross-stack references

Pass resources via typed props interfaces. Only export what must be shared.

```typescript
interface AppStackProps extends cdk.StackProps {
  table: dynamodb.Table;
  bucket: s3.Bucket;
}
```

### Nested stacks

Avoid. Use only at the 500-resource CloudFormation limit.

---

## Naming

### Casing rules

| Thing | Casing | Example |
| --- | --- | --- |
| Construct IDs | PascalCase | `'UserUploadsBucket'` |
| Stack construct IDs | PascalCase | `'FoundationStack'` |
| Stage construct IDs | PascalCase | `'Prod'`, `'Dev'` |
| Explicit `stackName` prop | kebab-case | `'my-service-prod'` |
| Explicit physical names (`bucketName`, etc.) | kebab-case | `'mycompany-uploads-prod'` |
| Tag keys | kebab-case with optional `org:` prefix | `'mycompany:environment'` |
| Export names | Namespaced PascalCase | `'MyApp-Prod-VpcId'` |

### Construct IDs

Use PascalCase. Must be unique within their parent scope. This is the single input you control — everything downstream (logical ID, physical name) is derived from it.

```
Construct ID:   'UserUploads'
     ↓
Logical ID:     UserUploads4DD88B4F       (CDK-generated, in CloudFormation template)
     ↓
Physical Name:  prod-stack-useruploads4dd88b4f-a1b2c3  (CloudFormation-generated)
```

**Changing a construct ID changes the logical ID, which causes CloudFormation to replace the resource. For stateful resources (databases, buckets, queues with data), this means data loss.**

### The `'Default'` ID trick

When wrapping an existing resource in a custom construct, use `'Default'` as the inner construct ID. CDK strips `'Default'` from the logical ID calculation, preserving the original logical ID and preventing resource replacement.

```typescript
// Before: new s3.Bucket(this, 'DataBucket')  →  logical ID: DataBucket<hash>

// After wrapping — safe, same logical ID:
class DataPipeline extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);
    new s3.Bucket(this, 'Default');  // 'Default' is stripped from logical ID
  }
}
new DataPipeline(this, 'DataBucket');  // →  logical ID: DataBucket<hash> (unchanged)
```

### Physical resource names

**Do not set explicit physical names** (`bucketName`, `tableName`, `functionName`) unless required. Let CloudFormation auto-generate them.

Hardcoded names:
- Block resource replacement (new resource fails with `AlreadyExists` while old one still holds the name)
- Prevent deploying the same stack twice in one account (name collisions)
- Cause deployment deadlocks requiring manual intervention

Set explicit names only when:
- External systems require a known, fixed name
- The name IS the configuration (Route53 hosted zones, custom domains)
- Resources are shared cross-account and ARN stability matters

Use `PhysicalName.GENERATE_IF_NEEDED` for cross-environment references where CDK needs a deterministic name but you don't want to choose one.

### Stage names

Use short PascalCase environment names: `Dev`, `Staging`, `Prod` (or `Beta`, `Gamma`, `Prod`). The stage ID auto-prefixes all child stack names: `Prod-FoundationStack`, `Prod-AppStack`.

### Actions that change logical IDs (cause resource replacement)

- Renaming a construct ID
- Wrapping a resource in a new parent construct (without using `'Default'`)
- Renaming a parent construct
- Moving a stack into or out of a Stage
- Changing the stack construct ID

Always run `cdk diff` before deploying to catch unexpected logical ID changes on stateful resources.

---

## Constructs

- Start with **L2** constructs. Drop to **L1** (`Cfn*`) only when L2 doesn't expose a needed property.
- Promote repeated multi-resource patterns into **L3** constructs when you have real duplication (2-3+ instances).
- Extend `Construct`, not `Stack`, for reusable abstractions.
- Model business logic with constructs. Use stacks only as deployment boundaries.

### Custom L2 wrappers

Use to enforce org-wide defaults (encryption, public access blocks, SSL):

```typescript
export class SecureBucket extends s3.Bucket {
  constructor(scope: Construct, id: string, props?: s3.BucketProps) {
    super(scope, id, {
      ...props,
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      enforceSSL: true,
    });
  }
}
```

---

## Environment Configuration

Use TypeScript config files with a shared interface for type safety:

```typescript
// config/types.ts
export interface EnvironmentConfig {
  env: { account: string; region: string };
  instanceType: string;
  minCapacity: number;
}
```

- Use **static stack creation** for the promotion pipeline (dev -> staging -> prod).
- Use **dynamic creation** (`-c stage=dev`) only for ephemeral developer stacks.
- Use SSM Parameter Store only for secrets or values that must be centralized.

---

## Testing

### Test your logic, not CDK's.

Ask: "If this property were wrong, would I have a production incident or security issue?" If yes, test it. If no, skip it.

### What to test

- Custom construct conditional logic
- Security-critical config (encryption, IAM policies, public access)
- Validation logic and error cases
- Resource counts from custom constructs
- Aspect behavior

### What NOT to test

- Vanilla L2 construct output (`new s3.Bucket()` producing `AWS::S3::Bucket`)
- Every single resource property
- CloudFormation intrinsics (`Fn::Ref`, `Fn::GetAtt`)
- CDK-internal logical IDs

### Fine-grained assertions (primary test type)

```typescript
import { Template, Match } from "aws-cdk-lib/assertions";

test("creates encrypted bucket with versioning", () => {
  const stack = new cdk.Stack();
  new SecureBucket(stack, "TestBucket");
  const template = Template.fromStack(stack);

  template.hasResourceProperties("AWS::S3::Bucket", {
    VersioningConfiguration: { Status: "Enabled" },
    BucketEncryption: {
      ServerSideEncryptionConfiguration: Match.arrayWith([
        Match.objectLike({
          ServerSideEncryptionByDefault: { SSEAlgorithm: "aws:kms" },
        }),
      ]),
    },
  });
});
```

### Snapshot tests

Use as a supplement only, never the sole strategy. Always review `.snap` diffs in PRs. They break on CDK version bumps and asset hash changes.

### Validation tests

Use `Annotations.fromStack()` to test Aspects and construct validation logic.

### Integration tests

Use `@aws-cdk/integ-tests-alpha` for runtime verification. Run pre-merge or nightly, not every commit.

### Key Match utilities

- `Match.objectLike({})` — deep partial match (default for `hasResourceProperties`)
- `Match.arrayWith([])` — array contains at least these elements
- `Match.anyValue()` — any non-absent value
- `Match.absent()` — property must NOT be present
- `Match.not(pattern)` — inverts any match
- `Match.stringLikeRegexp()` — regex match
- `Match.serializedJson({})` — parse and match stringified JSON

### Test helpers

Extract common setup into shared helpers:

```typescript
export function templateFrom(fn: (stack: cdk.Stack) => void): Template {
  const stack = new cdk.Stack(new cdk.App(), "TestStack", {
    env: { account: "123456789012", region: "us-east-1" },
  });
  fn(stack);
  return Template.fromStack(stack);
}
```

---

## Security

- Always use `grant*` methods over manual `PolicyStatement` construction.
- Apply `cdk-nag` at the app level: `Aspects.of(app).add(new AwsSolutionsChecks())`.
- Suppress cdk-nag rules with documented reasons using `NagSuppressions.addResourceSuppressions()`.
- Use permission boundaries in shared/sandbox accounts.
- Never put secrets in source code or CDK context. Use Secrets Manager or SSM Parameter Store.

---

## Aspects

Use for cross-cutting concerns: mandatory tagging, encryption enforcement, compliance validation.

- `Annotations.of(node).addError()` blocks deployment.
- `Annotations.of(node).addWarning()` warns but allows it.
- Aspects do not propagate across `Stage` boundaries.
- Prefer Aspects over custom L2 wrappers for governance — wrappers don't catch third-party constructs.

---

## CDK Pipelines

- Group stacks into `cdk.Stage` subclasses for pipeline deployment.
- Use Stages for sequential deployment, Waves for parallel.
- Use `CodePipelineSource.connection()` over OAuth tokens.
- Use dependency lock files.
- Bootstrap all target accounts/regions before first deployment.
- Add `ManualApprovalStep` before production stages.

---

## Code Reuse

- Shared construct libraries go in their own package with semantic versioning.
- Publish to a private registry (CodeArtifact or npm private).
- Use projen for scaffolding construct libraries.
- Declare `aws-cdk-lib` and `constructs` as `peerDependencies` in libraries.

---

## Escape Hatches

Use only when L2 doesn't expose a needed CloudFormation property. Access the L1 via `node.defaultChild as CfnXxx`. Use `addPropertyOverride()` for surgical fixes. Comment why it's necessary.

---

## Dependencies

- Single `aws-cdk-lib` dependency with `^` range.
- Pin alpha packages (`@aws-cdk/*-alpha`) to exact versions.
- CDK CLI and Construct Library version independently. Keep CLI current.

---

## Performance

- Set `"aws:cdk:disable-stack-trace": true` in `cdk.json` context.
- Use SWC transpiler: `@swc/core` + `"ts-node": { "swc": true }` in tsconfig.
- Always commit `cdk.context.json` to source control.
- Use `cdk deploy --concurrency N` to deploy independent stacks in parallel.
- Use `NodejsFunction` (esbuild) for Lambda bundling.
- Minimize `.fromLookup()` calls.
- Use `cdk watch --hotswap-fallback` during development.

---

## Do NOT

- Hardcode physical resource names. Let CDK generate them.
- Use `process.env` inside constructs. Accept config via typed props.
- Use CloudFormation Conditions or Parameters. Use TypeScript `if` statements at synth time.
- Abstract prematurely. Wait for 2-3 real duplicates.
- Change logical IDs of stateful resources (causes replacement/deletion).
- Use snapshot tests as the only test strategy.
- Mirror construct implementation in tests. Test decisions and conditions.
- Scatter compliance checks across test files. Centralize with Aspects and cdk-nag.
- Split into many stacks preemptively. Start with one, split by rate of change when needed.
- Skip `cdk.context.json` from source control.
- Use nested stacks unless at the 500-resource limit.
