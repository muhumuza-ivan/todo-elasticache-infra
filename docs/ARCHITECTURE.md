# Architecture

Single Region (`eu-west-1`), two Availability Zones, one custom VPC
(`10.20.0.0/16`) with five dedicated subnet tiers per AZ.

## Network diagram

```mermaid
flowchart TB
    dev([Developer]) -->|git push| ghapp[GitHub: todo-app]
    user([End user]) -->|HTTP :80| alb

    subgraph GHA[GitHub Actions - OIDC, no static keys]
      build[Build image<br/>mvn package + docker buildx]
    end
    ghapp --> build
    build -->|sts:AssumeRoleWithWebIdentity<br/>docker push| ecr[(Amazon ECR<br/>todo-app)]

    ecr -->|ECR Image Action: PUSH tag=latest| eb{{EventBridge rule}}
    eb -->|StartPipelineExecution| cp[CodePipeline]
    ghinfra[GitHub: todo-elasticache-infra<br/>main branch] -->|GitHub Actions: package + deploy| cfn[CloudFormation root stack<br/>8 nested stacks]
    ghapp -->|CodeConnections source<br/>taskdef.json + appspec.yaml| cp
    cp --> cd[CodeDeploy<br/>blue/green]

    subgraph VPC["VPC 10.20.0.0/16"]
      direction TB
      subgraph AZ1["Availability Zone A"]
        pub1[Public subnet<br/>10.20.0.0/24]
        ecs1[ECS subnet<br/>10.20.10.0/24]
        rds1[RDS subnet<br/>10.20.20.0/24]
        prx1[Proxy subnet<br/>10.20.30.0/24]
        cch1[Cache subnet<br/>10.20.40.0/24]
      end
      subgraph AZ2["Availability Zone B"]
        pub2[Public subnet<br/>10.20.1.0/24]
        ecs2[ECS subnet<br/>10.20.11.0/24]
        rds2[RDS subnet<br/>10.20.21.0/24]
        prx2[Proxy subnet<br/>10.20.31.0/24]
        cch2[Cache subnet<br/>10.20.41.0/24]
      end

      alb[Application Load Balancer<br/>:80 prod / :8080 test]
      tasks[ECS Fargate service<br/>1-4 tasks, awsvpc]
      proxy[RDS Proxy]
      db[(RDS PostgreSQL<br/>Multi-AZ, db.t3)]
      redis[(ElastiCache Redis<br/>primary + replica)]
      vpce[[Interface endpoints<br/>ecr.api, ecr.dkr, logs,<br/>ssm, ssmmessages, secretsmanager]]
      s3e[[S3 gateway endpoint]]

      pub1 -.-> alb
      pub2 -.-> alb
      ecs1 -.-> tasks
      ecs2 -.-> tasks
      prx1 -.-> proxy
      prx2 -.-> proxy
      rds1 -.-> db
      rds2 -.-> db
      cch1 -.-> redis
      cch2 -.-> redis

      alb -->|HTTP :8080| tasks
      tasks -->|TLS :6379 cache-aside reads| redis
      tasks -->|TLS :5432 writes| proxy
      proxy -->|:5432| db
      tasks --> vpce
      tasks --> s3e
    end

    cd -->|shift listeners| alb
    cd -->|register revision| tasks
    vpce -.-> ecr
    tasks -->|logs| cw[(CloudWatch Logs<br/>/ecs/todo-app-dev)]
    tasks -->|GetSecretValue| sm[(Secrets Manager<br/>db credentials)]
    vpce -.-> sm
```

## Request path

1. A browser hits the ALB on port 80. The ALB is the only internet-facing
   component; its security group allows 80 in from the internet and 8080 out to
   the ECS security group, nothing else.
2. The ALB forwards to a Fargate task in a private ECS subnet. Tasks have
   `AssignPublicIp: DISABLED` and no NAT gateway.
3. `GET /api/tasks` first asks Redis (`todo:tasks:all`). On a hit the response is
   returned straight from cache; on a miss the task reads PostgreSQL through the
   RDS Proxy and repopulates the key with a 60 second TTL.
4. Every write (`POST`/`PUT`/`DELETE`) goes to PostgreSQL through the proxy and
   then invalidates the affected cache keys, so the next read re-warms them.

## Why there is no NAT gateway

The only outbound calls a task makes are to ECR (image pull), CloudWatch Logs,
SSM Parameter Store, Secrets Manager and S3 (ECR layers). All six are reachable
over VPC endpoints, so the private subnets carry no default route at all. That
removes roughly $65/month of NAT charges per AZ and removes internet egress as
an exfiltration path.

## Security group matrix

| Group | Ingress | Egress |
| --- | --- | --- |
| `alb-sg` | :80 from `AllowedHttpCidr`, :8080 from `TestListenerCidr` | :8080 to `ecs-sg` |
| `ecs-sg` | :8080 from `alb-sg` | :5432 to `rdsproxy-sg`, :6379 to `cache-sg`, :443 to `vpce-sg`, :443 to S3 prefix list |
| `rdsproxy-sg` | :5432 from `ecs-sg` | :5432 to `rds-sg` |
| `rds-sg` | :5432 from `rdsproxy-sg` | none (pinned to 127.0.0.1/32) |
| `cache-sg` | :6379 from `ecs-sg` | none (pinned to 127.0.0.1/32) |
| `vpce-sg` | :443 from `ecs-sg` | none (pinned to 127.0.0.1/32) |

Leaving `SecurityGroupEgress` out of a CloudFormation security group silently
creates an allow-all `0.0.0.0/0` rule, so every group here declares egress
explicitly and the three that never initiate connections are pinned to
`127.0.0.1/32`.

## Deployment flow

| Step | Component | Detail |
| --- | --- | --- |
| 1 | GitHub Actions (`todo-app`) | Builds the image, assumes an IAM role via OIDC, pushes `:<sha>` then `:latest` to ECR |
| 2 | EventBridge | Rule matches `ECR Image Action` / `PUSH` / `SUCCESS` / `image-tag: latest` |
| 3 | CodePipeline | Source stage pulls `taskdef.json` + `appspec.yaml` from the app repo and `imageDetail.json` from ECR |
| 4 | CodeDeploy | Registers a new task definition revision (`<IMAGE1_NAME>` replaced), starts the green task set behind the :8080 test listener |
| 5 | CodeDeploy | Shifts the :80 production listener to green, keeps blue for 5 minutes, then terminates it |
| 6 | CloudWatch alarms | 5xx or unhealthy-host alarms roll the deployment back automatically |

## Diagrams

`docs/architecture.drawio.xml` is the reference diagram: standard AWS
Architecture Icons, AWS Cloud / Region / VPC / Availability Zone grouping, all
five subnet tiers with their real CIDRs, and the release path numbered 1-4.
Open it at <https://app.diagrams.net> (File -> Open From -> Device) or with the
draw.io extension in VS Code or JetBrains. Export to PNG or SVG from
File -> Export as.
