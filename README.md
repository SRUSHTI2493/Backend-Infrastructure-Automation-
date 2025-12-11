# Backend-Infrastructure-Automation-

Backend Infrastructure Automation (CI/CD + Terraform + ECR)

🟦 1. PROJECT OVERVIEW (WHAT YOU TELL IN INTERVIEW)

“I implemented a complete CI/CD and Infrastructure Automation system for the Backend using GitHub Actions and Terraform.
The goal was to automatically test, build, and deploy backend code to environment-specific AWS ECR repositories (dev & master).
I fixed the CI pipeline, refactored Terraform code, created environment-based ECR repos, and ensured automated infrastructure provisioning.”

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

Purpose: Run CI checks on every push/PR to dev and master.

Steps: checkout → setup pnpm → setup node → install deps → format check → lint → typecheck → unit tests → build.

Why: catches issues early; prevents bad code from merging.

.github/workflows/backend-build-push.yml

Purpose: Build Docker image and push to ECR when tests pass (often on tagged commits or main merges).

Why: Produce immutable image artifacts for deployments.

.github/workflows/infra-preview.yml

Purpose: Run terraform plan (preview) to show infra changes in a PR.

Why: Review infra changes before applying.

.github/workflows/infra-deploy.yml

Purpose: Run terraform apply (actual provision) on dev or master pushes or a workflow dispatch.

Key: Pass TF_VAR_environment = ${{ github.ref_name }} so Terraform creates resources for that environment (e.g., backend-dev).

Why: Automate infra creation & keep infra as code.

apps/infra/provider.tf

Purpose: Configure AWS provider and region.

Why: Terraform needs provider config to talk to AWS.

apps/infra/variables.tf

Purpose: Define variable "environment" { ... }.

Why: Make ECR repo name & other resources environment-specific.

apps/infra/main.tf

Purpose: Create aws_ecr_repository and aws_ecr_lifecycle_policy.

Example resource behaviors:

name = "backend-${var.environment}"

force_delete = false for safety

lifecycle { prevent_destroy = true } to avoid accidental deletions in production

Why: Ensure separate ECR repos per env and clean image retention.

apps/infra/outputs.tf

Purpose: Expose ecr_url and repo_name (from Terraform).

Why: Downstream workflows (like build-push) read these outputs to tag & push images.

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
