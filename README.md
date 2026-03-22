# Static website on AWS ECS Fargate + GitHub Actions

Simple HTML site containerised with nginx, deployed to ECS Fargate via GitHub Actions. Two pipelines — CI builds and tests the image, CD deploys it. Nothing runs unless CI passes first.

## How it works

Push to main → CI builds the Docker image, spins it up locally, curls it to make sure it actually responds, then pushes to ECR. Once that's green, the CD pipeline picks up, registers the task definition, and creates or updates the ECS service. At the end it prints the public IP so you know where to go.

- index.html
- Dockerfile
- .dockerignore
- ecs/task-def.json
- .github/workflows/ci.yml
- .github/workflows/cd.yml


## To replicate this

### 1. Fork the repo and make it public

GitHub Actions needs the repo to be public if you're on a free account.

### 2. Create an IAM user for the pipeline

Go to IAM → Users → Create user. Call it something like `github-deployer`, pick programmatic access. Attach these managed policies:

- `AmazonEC2ContainerRegistryFullAccess`
- `AmazonECS_FullAccess`
- `AmazonEC2ReadOnlyAccess` (needed for the subnet/SG lookups in the CD pipeline)

Save the access key and secret — you won't see the secret again after you leave that page.

### 3. Create the ECS task execution role

This is the role ECS uses to pull your image from ECR and write logs. The CD pipeline expects it to exist with a specific name.

IAM → Roles → Create role → AWS service → Elastic Container Service Task. Attach `AmazonECSTaskExecutionRolePolicy`. Name it exactly `ecsTaskExecutionRole`.

### 4. Add secrets to GitHub

Settings → Secrets and variables → Actions:

 `AWS_ACCESS_KEY_ID` ->  from step 2
 `AWS_SECRET_ACCESS_KEY` -> from step 2
 `AWS_REGION` -> e.g. `us-east-1`

### 5. Push something to main

The pipeline creates the ECR repo, ECS cluster, and ECS service automatically if they don't exist — nothing to pre-create. Once the CD workflow finishes, open the "Print App URL" step in the run logs and you'll see the public IP.

> The task gets a direct public IP since there's no load balancer in this setup. That IP changes on every deploy, which is one of the first things I'd fix with more time.


## What I'd do with more time

- The biggest gap right now is that the site has no stable URL. Every time you deploy, the ECS task gets a new public IP. The fix is an ALB in front of the service — fixed DNS name, ACM certificate for HTTPS, Route 53 domain pointing at it.
- After that I'd move the tasks into private subnets. Right now they're in the default VPC with a public IP each, which isn't great. Tasks should sit behind the ALB with no direct internet exposure. While I was at it I'd add a VPC endpoint for ECR so image pulls don't go over the public internet.
- CloudWatch logging isn't configured on the container right now — if something breaks there's nowhere to look. I'd add a log group to the task definition and wire up a dashboard for latency, error rate, and ECS CPU/memory, with alarms to SNS.
- On the pipeline side, I'd swap the static IAM access keys for GitHub OIDC — no long-lived credentials in GitHub Secrets, the runner just assumes a role with a short-lived token. 
- Auto-scaling next — target-tracking on ALB request count is simple to set up and means you're not manually adjusting desired count when traffic spikes. And properly wiring up blue/green via CodeDeploy so releases are zero-downtime with automatic rollback if the health check fails during cutover.
- Longer term everything should be in cloudformation, cdk or Terraform. Right now the infrastructure is created by AWS CLI commands in the pipeline, which works but means recreating it from scratch involves manual steps.



## Why ECS and not something simpler

- The most obvious alternative is S3 + CloudFront — drop the HTML in a bucket, enable static hosting, CloudFront in front. Cheaper, easier, and honestly the right answer for a site that's purely static and never going to need a backend. I went with ECS because the brief said this is for a new product, and products tend to grow. Containerised from the start means you can add a backend or swap the runtime later without rearchitecting. S3 hosting doesn't give you that path.
- EC2 was the other option. I've used it plenty but Fargate is strictly easier to operate here — no AMIs to patch, no instance sizing, you just pay for what the task actually uses. I'd only reach for EC2 if I needed something Fargate doesn't support, and nothing here requires it.
- App Runner would've been even simpler — pull straight from ECR with barely any config. The reason I didn't go that way is it abstracts away too much. You lose control over networking, task definition details, and deployment strategy. For something you'd hand to an ops team, ECS gives you levers that App Runner doesn't.



## What it would take to make this production grade

- Environments: Everything currently goes to one place. A real setup needs dev, staging, and prod in separate AWS accounts — not just separate clusters in one account. Staging should mirror prod as closely as possible. Promotion from staging to prod needs a manual approval gate in the pipeline so a human signs off before it goes live.
- IaC: All infrastructure needs to be in code. Right now things get created by AWS CLI commands in the pipeline, which works but isn't reviewable or reproducible. Terraform or CDK in the repo, reviewed like any other PR. That covers VPC, subnets, ALB, security groups, ECS cluster, ECR, IAM roles — everything.
- Security: The task execution role has broader access than it needs and should be scoped down to the specific ECR repo. ECR image scanning with Amazon Inspector get caught common vulnerabilities, before prod. The security group here uses the default VPC group which allows all inbound — production needs explicit rules allowing only the ALB to reach the tasks. WAF on the ALB for basic OWASP coverage is also pretty straightforward to add.
- Observability: CloudWatch logging needs to be added to the task definition first — once that's in place, next step is a dashboard covering latency, error rate, ECS CPU/memory, and ALB healthy host count. Alarms on those with SNS for on-call. X-Ray tracing is worth adding once you have more than one container, so you can actually follow a request through the system.
- Pipeline: Branch protection on main — nothing merges without a review and green CI. SAST in CI. DAST against staging before prod. Semantic versioning on images rather than git SHAs — makes rollbacks much easier to reason about.
- Few More:  Cost allocation tags on everything so spend is attributable per team and environment.