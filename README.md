# todo-infra

CloudFormation for a highly available, containerised Java To-Do application on
Amazon ECS Fargate, with Amazon RDS for PostgreSQL behind RDS Proxy and
ElastiCache for Redis as a read cache.

Application code lives in a separate repository: **`todo-elasticache-app`**.

- Region: `eu-west-1` (change consistently in both repos if you move it)
- Stack name: `todo-app-dev` (one stack; everything else is nested inside it)
- SSM parameter namespace: `/todo-app/dev/*`

## One stack, one workflow

The whole environment is a single CloudFormation stack. `main.yaml`
declares each component as a nested stack, and
`.github/workflows/deploy.yml` deploys it on every push to `main`:

```
push to main
  └─ cfn-lint
       └─ assume the deploy role through GitHub OIDC
            └─ aws cloudformation package   (upload children to S3, rewrite TemplateURLs)
                 └─ create change set       (printed to the run summary)
                      └─ execute            (CloudFormation assumes the stack role)
```

There are no per-stack deployment files to keep in step, and nothing to click:
changing a template is the deployment. A change set is always created first, so
the run log shows exactly what will happen before it happens, and
`workflow_dispatch` offers a plan-only run that stops after printing it.

```
main.yaml            deployed from `main`: nests everything in templates/
bootstrap.yaml       deployed from `bootstrap`: the deployment machinery
templates/
├── ecr.yaml         ECR repository, lifecycle policy, SSM parameters
├── github-oidc.yaml GitHub OIDC trust + the ECR push role for the app workflow
├── network.yaml     VPC, 5 subnet tiers x 2 AZ, route tables, S3 gateway endpoint, flow logs
├── security.yaml    6 security groups (explicit egress) + interface VPC endpoints
├── data.yaml        RDS PostgreSQL Multi-AZ, RDS Proxy, RDS-managed secret, SSM parameters
├── cache.yaml       ElastiCache Redis replication group, SSM parameters
├── app.yaml         ALB, blue/green target groups, ECS cluster/service, task roles, auto scaling
└── pipeline.yaml    CodeDeploy, CodePipeline, EventBridge ECR trigger, artifact bucket
```

### Two branches, two workflows

| Branch | Template | Workflow | Deploys |
| --- | --- | --- | --- |
| `main` | `main.yaml` | `deploy.yml` | the environment: eight nested stacks |
| `bootstrap` | `bootstrap.yaml` | `bootstrap.yml` | the deployment machinery |

`bootstrap.yaml` creates what the deploy workflow needs before it can run at
all: the S3 bucket `package` uploads to, the OIDC role the workflow assumes,
the role CloudFormation assumes to build resources, and the CodeConnections
connection CodePipeline reads the application repository through.

It sits on its own branch for three reasons. These are the only resources that
can grant access to the account; they change almost never; and a change to them
has nothing to do with shipping the application, so it should not ride the same
trigger. Keeping them apart also means `main` cannot quietly alter the role that
`main` itself is deployed with.

The deploy role trusts exactly two OIDC subjects — `refs/heads/main` and
`refs/heads/bootstrap` of this repository — and nothing else.

**The first creation cannot be automated.** `bootstrap.yml` authenticates with
a role that `bootstrap.yaml` creates, so on a green field neither exists yet.
Create the stack once from the console; the workflow maintains it after that,
and refuses with a clear message rather than trying to create it.

One step inside it has no automation in any interface: a human must complete
the GitHub OAuth handshake to move the connection from `PENDING` to
`AVAILABLE`.

Because these resources are sensitive, `bootstrap.yml` always prints the change
set, and its manual-run default is **plan-only** — the opposite of the
environment deploy.

## Order of operations

**1. Create the bootstrap stack, once, by hand.** Console → *CloudFormation →
Create stack → With new resources → Upload a template file* → `bootstrap.yaml`.
Name it `todo-app-dev-bootstrap`, acknowledge the IAM capability. Every later
change to it goes through the `bootstrap` branch instead:

```bash
git switch -c bootstrap     # first time; thereafter: git switch bootstrap
git push -u origin bootstrap
```

**2. Complete the GitHub handshake.** *Developer Tools → Settings →
Connections → `todo-app-dev-github` → Update pending connection*. Install the
AWS Connector for GitHub and grant it access to the **application**
repository. Confirm the status reads `AVAILABLE`.

**3. Set the repository secrets** on `todo-elasticache-infra`, from the
bootstrap stack outputs (Settings → Secrets and variables → Actions →
Secrets):

| Secret | Value |
| --- | --- |
| `AWS_REGION` | `eu-west-1` |
| `AWS_DEPLOY_ROLE_ARN` | output `DeployRoleArn` |
| `AWS_STACK_ROLE_ARN` | output `StackRoleArn` |
| `CFN_TEMPLATE_BUCKET` | output `TemplateBucketName` |

**4. Push to `main`.** The workflow builds the whole environment. First run is
roughly 25 minutes, dominated by the Multi-AZ RDS instance.

**5. Set the application repository secrets** from the root stack outputs,
then push the application so its first image reaches ECR — see
[CI configuration](#ci-configuration).

> **The ECS service cannot stabilise until an image exists in ECR.** There is no
> NAT gateway, so there is no public fallback image. If the first deploy stalls
> on `AppStack`, this is why: push the application, then re-run the workflow.

From then on, editing a template on `main` is the deployment, and editing
`bootstrap.yaml` on `bootstrap` updates the machinery. Keep the `bootstrap`
branch in step with `main` for the workflow files themselves — GitHub runs a
workflow from the branch that was pushed, so `.github/workflows/bootstrap.yml`
has to exist on `bootstrap`.

### Values to confirm before the first deploy

All of these are parameters of `main.yaml`, with defaults:

- `CreateOidcProvider` is `'false'` because this account already holds a
  provider for `token.actions.githubusercontent.com`; IAM permits one per
  issuer URL. In a fresh account set it to `'true'` and clear
  `ExistingOidcProviderArn`.
- `S3PrefixListId` is Region specific — `pl-6da54004` for eu-west-1.
- `AppRepositoryId` must be the application repository's `owner/repo`.

## How configuration reaches the container

Nothing about the environment is baked into the image or rewritten by the
pipeline. Each stack publishes what it owns to SSM Parameter Store, and
`taskdef.json` in the application repository references those parameters:

| Parameter | Written by | Consumed as |
| --- | --- | --- |
| `/todo-app/dev/db/host` | `data.yaml` (RDS Proxy endpoint) | `DB_HOST` |
| `/todo-app/dev/db/port` | `data.yaml` | `DB_PORT` |
| `/todo-app/dev/db/name` | `data.yaml` | `DB_NAME` |
| `/todo-app/dev/db/secret-arn` | `data.yaml` (RDS-managed secret) | `DB_SECRET_ARN` |
| `/todo-app/dev/redis/host` | `cache.yaml` | `REDIS_HOST` |
| `/todo-app/dev/redis/port` | `cache.yaml` | `REDIS_PORT` |
| `/todo-app/dev/redis/ssl` | `cache.yaml` | `REDIS_SSL` |
| `/todo-app/dev/codeconnections/arn` | `bootstrap.yaml` | root stack → pipeline |

Stack-to-stack wiring inside the root stack does not go through SSM: the ECR
URI and repository name are handed to the app and pipeline stacks directly with
`!GetAtt EcrStack.Outputs.*`. Only values consumed at *container run time*, or
produced by the hand-created bootstrap stack, use Parameter Store.

**Nobody generates the database password.** `data.yaml` sets
`ManageMasterUserPassword: true`, so RDS creates it, owns the secret and
rotates it every seven days. It never appears in a template or a change set.

## CI configuration

| Repository | Secret | Value |
| --- | --- | --- |
| `todo-elasticache-infra` | `AWS_REGION` | `eu-west-1` |
| `todo-elasticache-infra` | `AWS_DEPLOY_ROLE_ARN` | bootstrap output `DeployRoleArn` |
| `todo-elasticache-infra` | `AWS_STACK_ROLE_ARN` | bootstrap output `StackRoleArn` |
| `todo-elasticache-infra` | `CFN_TEMPLATE_BUCKET` | bootstrap output `TemplateBucketName` |
| `todo-elasticache-app` | `AWS_REGION` | `eu-west-1` |
| `todo-elasticache-app` | `AWS_ROLE_ARN` | root output `GitHubActionsRoleArn` |
| `todo-elasticache-app` | `ECR_REPOSITORY` | `todo-app` |

No long-lived access keys exist anywhere. Both roles are assumed through
GitHub's OIDC provider, and each trust policy pins the `sub` claim to one
repository and one branch.

## Design notes

**Two roles behind the deployment.** The *deploy* role
(`todo-app-dev-gha-deploy`) is what GitHub Actions assumes: it can drive
CloudFormation and upload templates, and that is all. The *stack* role
(`todo-app-dev-cfn-exec`) is what CloudFormation assumes to create resources,
and the workflow can only hand it over, never use it. The stack role uses
service-level wildcards for the services this project owns — a missing action
surfaces as a rollback twenty minutes into an RDS create, which is worse than a
broad grant in a single-purpose account — but IAM is pinned to the
`todo-app-*` name prefix, because that is the one privilege that could escape
the role.

**Ordering is explicit.** The children still wire themselves together with
`Export`/`ImportValue`, so the root uses `DependsOn` to sequence them:
network → security → {data, cache} → app → pipeline. Where a value can be
handed over directly it is, with `!GetAtt Child.Outputs.X`, which creates the
dependency implicitly.

**No NAT gateway.** Private subnets have no default route. ECR, CloudWatch
Logs, SSM, Secrets Manager and ECS Exec are reached over interface endpoints,
S3 (ECR layers) over a gateway endpoint.

**Egress is never implicit.** A CloudFormation security group without
`SecurityGroupEgress` gets an allow-all rule. Every group here declares its
egress; `rds-sg`, `cache-sg` and `vpce-sg` never initiate connections and are
pinned to `127.0.0.1/32`.

**Five subnet tiers.** ALB, ECS tasks, RDS Proxy, RDS and ElastiCache each get
their own subnets and security group, so a compromise in one tier cannot reach
another except on the one port that tier is allowed to use.

**Mutable image tags are load bearing.** The build workflow re-points `:latest`
at each new digest and the EventBridge rule fires on exactly that tag. An
immutable repository would make every deployment after the first one fail.

**Blue/green with automatic rollback.** The deployment group watches an ALB 5xx
alarm and an unhealthy-host alarm; either trips a rollback, and the blue task
set is kept for five minutes after cutover.

**Cost.** Roughly: ALB ~$18/mo, RDS `db.t3.micro` Multi-AZ ~$30/mo, two
`cache.t4g.micro` nodes ~$25/mo, RDS Proxy ~$11/mo, six interface endpoints
~$45/mo, Fargate 0.5 vCPU/1 GB ~$18/mo. Setting `MultiAz` to `false` and
`NumCacheNodes` to 1 roughly halves the data tier for a lab.

## Teardown

One stack, so one delete:

```
aws cloudformation delete-stack --stack-name todo-app-dev
```

CloudFormation removes the nested stacks in reverse dependency order. Then
delete `todo-app-dev-bootstrap`.

- The pipeline's S3 artifact bucket must be emptied before its nested stack
  will delete.
- `data.yaml` leaves a final RDS snapshot (`DeletionPolicy: Snapshot`) that
  keeps costing until you remove it. The RDS-managed secret is deleted with the
  instance.
- `ecr.yaml` sets `EmptyOnDelete: true`, so the registry deletes with images in
  it.
- The IAM OIDC provider survives: no stack owns it. It pre-existed this project.
