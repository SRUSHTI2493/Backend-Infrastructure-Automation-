# Backend-Infrastructure-Automation-

Backend Infrastructure Automation (CI/CD + Terraform + ECR)

🟦 1. PROJECT OVERVIEW 
“I implemented a complete CI/CD and Infrastructure Automation system for the Backend using GitHub Actions and Terraform.
The goal was to automatically test, build, and deploy backend code to environment-specific AWS ECR repositories (dev & master).
I fixed the CI pipeline, refactored Terraform code, created environment-based ECR repos, and ensured automated infrastructure provisioning.”

🚀 The project uses GitHub Actions + Terraform to fully automate backend CI and AWS infra 🚀. Whenever code is pushed to dev or master, GitHub Actions runs tests, linting, type-checking, and builds the backend 🔍⚙️. At the same time, Terraform automatically creates or updates an environment-specific ECR repository like backend-dev or backend-master using the branch name 🌿➡️🏗️. Secrets are pulled securely from GitHub, and Terraform outputs the repo URL so the build workflow can push Docker images 🐳📦. Safety rules like prevent_destroy = true protect production from accidental deletion 🔒. Overall, the whole pipeline ensures clean code, safe infra changes, separate dev/prod environments, and fully automated deployments 🤖✨.

🔥 BUSINESS REASON:

Backend images must go to different repos for dev and prod

Developers should not manually create ECR repos

Infrastructure must be managed through Terraform

Every commit should be validated through CI

Ensures repeatability, reduces errors, speeds up deployment
🟦 2. FULL PROJECT FILE STRUCTURE 
<img width="793" height="595" alt="image" src="https://github.com/user-attachments/assets/04b94e7f-4239-4dbf-98ef-809372d0f007" />

<img width="1000" height="650" alt="image" src="https://github.com/user-attachments/assets/51927542-21e8-4f85-a3a4-5c07b6b9e552" />

2) What each file does & why it exists (short)

.github/workflows/ci.yml (or backend-ci.yml)

🟦 Purpose: Run CI checks on every push/PR to dev and master.

Steps: checkout → setup pnpm → setup node → install deps → format check → lint → typecheck → unit tests → build.

 🟦 Why: catches issues early; prevents bad code from merging.

.github/workflows/backend-build-push.yml

🟦Purpose: Build Docker image and push to ECR when tests pass (often on tagged commits or main merges).

 🟦Why: Produce immutable image artifacts for deployments.

.github/workflows/infra-preview.yml

🟦Purpose: Run terraform plan (preview) to show infra changes in a PR.

🟦Why: Review infra changes before applying.

.github/workflows/infra-deploy.yml

🟦Purpose: Run terraform apply (actual provision) on dev or master pushes or a workflow dispatch.

Key: Pass TF_VAR_environment = ${{ github.ref_name }} so Terraform creates resources for that environment (e.g., backend-dev).

🟦Why: Automate infra creation & keep infra as code.

apps/infra/provider.tf

🟦Purpose: Configure AWS provider and region.

🟦Why: Terraform needs provider config to talk to AWS.

apps/infra/variables.tf

🟦Purpose: Define variable "environment" { ... }.

🟦Why: Make ECR repo name & other resources environment-specific.

apps/infra/main.tf

🟦Purpose: Create aws_ecr_repository and aws_ecr_lifecycle_policy.

Example resource behaviors:

name = "backend-${var.environment}"

force_delete = false for safety

lifecycle { prevent_destroy = true } to avoid accidental deletions in production

🟦Why: Ensure separate ECR repos per env and clean image retention.

apps/infra/outputs.tf

🟦Purpose: Expose ecr_url and repo_name (from Terraform).

🟦Why: Downstream workflows (like build-push) read these outputs to tag & push images.

3) Which files depend on which (mapping / dependency)

infra-deploy.yml → runs Terraform in apps/infra → uses provider.tf, variables.tf, main.tf, outputs.tf.

backend-build-push.yml → needs the ECR repo_name and ecr_url produced by Terraform (or reads them via terraform output or environment secrets).

ci.yml → only concerns apps/backend (no Terraform dependency).

main.tf uses var.environment (from variables.tf). outputs.tf references aws_ecr_repository.backend.
4) Step-by-step workflow (developer → CI → Infra) — use this as your interview script
Developer flow (what a developer does)

Create feature branch:
git checkout -b feat/my-change

Implement changes in apps/backend/...

Run locally: pnpm install && pnpm run lint && pnpm run test
(Fix issues until green)

Commit and push:
git add . && git commit -m "feat: X" && git push origin feat/my-change

Open Pull Request targeting dev

CI runs automatically (ci.yml)

Fix discovered lint/type/test issues in the PR

When review passed, merge to dev.

backend-build-push.yml can create/push images (if configured)

infra-preview.yml shows planned infra changes for PRs that touched apps/infra

When everything is stable, merge dev → master (or create release), CI runs for master, infra apply runs if needed.

CI / Infra flow (automated):

On PR / push: ci.yml runs: checkout → pnpm → node → install → lint/test/build.

If infra changes in PR: infra-preview.yml runs terraform plan in apps/infra.

Shows what Terraform will change (safety).

When branch merged to dev/master:

infra-deploy.yml runs terraform init & terraform apply in apps/infra.

The workflow sets TF_VAR_environment = ${{ github.ref_name }} so names become backend-dev or backend-master.

Terraform creates ECR repo if it doesn’t exist (or errors if repo exists and force_delete/prevent_destroy mismatch).

Terraform outputs ecr_url and repo_name.

Build & Push:

backend-build-push.yml (or release workflow) uses the outputs or env to tag and push the image to the environment-specific ECR repo.

Tags used: ${{ github.ref_name }} for versioned releases, and ${{ github.sha }} for commit-specific image.

Deploy (app-specific):

If you have deployment workflows (ECS, EKS, etc.), they will reference the pushed image URI and update the cluster/service.

5) Common issues i fixed 

pnpm install order: pnpm must be installed before setup-node caching expects pnpm; otherwise CI can’t find pnpm in PATH. Fixed by using pnpm/action-setup before actions/setup-node.

Duplicate Terraform providers: You must only declare required_providers in one place and only configure the default provider once. Moved provider config to provider.tf and removed duplicate blocks.

ECR repo name conflicts: If the repo name is constant (backend) both dev & master collide. Fixed by using name = "backend-${var.environment}".

Terraform errors when repo exists: either set force_delete appropriately or choose unique names per environment. Also prevent_destroy = true to avoid accidental destruction.

Passing environment from workflow: Use env: TF_VAR_environment: ${{ github.ref_name }} before terraform apply to pass the environment variable into Terraform.

🚀 ECS Fargate Deployment (After Image Push to ECR)

Once the Docker image is pushed to ECR, your next step is to run that image in AWS ECS Fargate so your backend API becomes LIVE.

📁 Files Needed for ECS Deployment (Terraform)

These are the required Terraform files in your apps/infra folder.

1️⃣ ecs-cluster.tf
📌 Purpose:

Defines your ECS Cluster where Fargate tasks will run.

📄 File Code:
resource "aws_ecs_cluster" "backend_cluster" {
  name = "backend-${var.environment}-cluster"
}

🧠 Why it's needed?

Because ECS tasks must run inside a cluster.
Think of the cluster as the container orchestrator that manages tasks.

2️⃣ task-definition.tf
📌 Purpose:

Defines HOW to run your backend container.

📄 File Code:
resource "aws_ecs_task_definition" "backend_task" {
  family                   = "backend-${var.environment}-task"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  network_mode             = "awsvpc"

  execution_role_arn = aws_iam_role.ecs_execution_role.arn
  task_role_arn       = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name      = "backend"
      image     = "${aws_ecr_repository.backend.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 3000
          hostPort      = 3000
        }
      ]
    }
  ])
}

🧠 Why it's needed?

This file tells ECS:

Which image to run 🐳

How much CPU/RAM to give 💾

Which port to expose 🌐

What IAM roles are needed 🔐

It's basically your Docker run instruction in Terraform form.


3️⃣ ecs-service.tf
📌 Purpose:

This file runs your task and keeps it alive on Fargate.

📄 File Code:
resource "aws_ecs_service" "backend_service" {
  name            = "backend-${var.environment}-service"
  cluster         = aws_ecs_cluster.backend_cluster.id
  task_definition = aws_ecs_task_definition.backend_task.arn
  launch_type     = "FARGATE"

  desired_count = 1

  network_configuration {
    subnets         = var.private_subnets
    security_groups = [aws_security_group.backend_sg.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.backend_tg.arn
    container_name   = "backend"
    container_port   = 3000
  }

  depends_on = [
    aws_lb_listener.backend_listener
  ]
}

🧠 Why it's needed?

The ECS service:

Runs 1 or more containers

Restarts them if they crash

Connects them to load balancer

Pulls latest image on updates

It ensures your backend stays live 24/7.


4️⃣ alb.tf (Application Load Balancer)
📌 Purpose:

Exposes your backend API publicly on a stable URL.

📄 File Code:
resource "aws_lb" "backend_alb" {
  name               = "backend-${var.environment}-alb"
  load_balancer_type = "application"
  subnets            = var.public_subnets
  security_groups    = [aws_security_group.alb_sg.id]
}

resource "aws_lb_target_group" "backend_tg" {
  name     = "backend-${var.environment}-tg"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "backend_listener" {
  load_balancer_arn = aws_lb.backend_alb.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.backend_tg.arn
  }
}

🧠 Why it's needed?

Because:

✔ Users can't directly call Fargate tasks
✔ ALB provides a public URL
✔ ALB forwards traffic to ECS tasks

It is the entry point of your backend.


5️⃣ iam.tf
📌 Purpose:

IAM roles required for ECS to pull images from ECR & write logs.

📄 File Code:
resource "aws_iam_role" "ecs_execution_role" {
  name = "ecsExecutionRole-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },
      Effect = "Allow"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution_policy" {
  role       = aws_iam_role.ecs_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

🧠 Why it's needed?

ECS needs permissions to pull images from ECR ✔

ECS needs permissions to push logs to CloudWatch ✔

Without IAM roles, ECS cannot run tasks.


6️⃣ security-groups.tf
📌 Purpose:

Define network access rules for ALB and ECS tasks.

Example:
resource "aws_security_group" "alb_sg" {
  name = "alb-sg-${var.environment}"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "backend_sg" {
  name = "backend-sg-${var.environment}"

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }
}

🧠 Why it's needed?

ALB → ECS traffic rules

Secure your backend

No public access directly to ECS

7️⃣ vpc.tf

(Optional if your company already has VPC)

📌 Purpose:

Defines networking resources — subnets, routing, etc.

🌟 FINAL FLOW (Add to README)

Here is the clean summary with emojis—you can paste this in your README:

🚀 Backend Deployment Workflow — ECR → ECS Fargate

Once the Docker image is pushed to Amazon ECR, this infrastructure deploys and runs the backend API using AWS ECS Fargate:

1️⃣ ECS Cluster 🏗
Creates the container orchestration environment.

2️⃣ Task Definition 📦
Defines the container (image, port, CPU, RAM, logs, IAM roles).

3️⃣ Application Load Balancer (ALB) 🌐
Provides a public URL and routes traffic to ECS tasks.

4️⃣ ECS Service 🔁
Runs the backend as Fargate tasks and keeps them healthy.

5️⃣ IAM Roles 🔐
Allow ECS to pull images from ECR and write logs.

6️⃣ Security Groups 🔥
Allow only ALB → ECS traffic securely.

7️⃣ VPC/Subnets 🌍
Networking layer for tasks to run inside AWS.

🟢Result:
Your backend API becomes LIVE using ECS Fargate, automatically pulling the latest image from ECR, with auto-restart, load balancing, and zero-downtime deployments.
