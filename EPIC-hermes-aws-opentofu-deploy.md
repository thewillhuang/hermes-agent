# Epic: Deploy Hermes Agent to AWS using OpenTofu

**Status:** Ready for grooming / decomposition  
**Priority:** High  
**Labels:** epic, infrastructure, aws, deployment, opentofu, devops  

## Goal
Enable production-grade deployment of Hermes Agent (CLI, gateway, profiles, skills, cron jobs, multi-platform messaging) to AWS using OpenTofu (the open-source Terraform fork). The target machine has an AWS admin account and AWS CLI pre-installed, so the solution must support local `aws` + `tofu` workflows for planning, applying, and destroying infrastructure.

## Business Value
- Moves Hermes from "run on my laptop" to "run reliably in the cloud".
- Supports persistent gateway for Telegram/Discord/Slack bots, cron jobs, webhooks, and long-running agent workers.
- Provides IaC so deployments are reproducible, auditable, and version-controlled.
- Aligns with agentneo SDLC and multi-agent pipelines that need always-on infrastructure.
- Uses OpenTofu to avoid Terraform licensing concerns.

## Scope (MVP - Single Instance Gateway)
- VPC + public/private subnets + security groups (SSH + gateway ports + webhook ports).
- EC2 t3.medium (or configurable) instance with Amazon Linux 2023.
- Systemd service (or Docker) to run `hermes gateway start` persistently.
- IAM role with least-privilege (CloudWatch Logs, SSM Parameter Store, optional S3).
- SSM Parameter Store or Secrets Manager for Hermes config secrets (API keys, tokens).
- CloudWatch Logs group + metric filters for agent.log / errors.log / gateway.log.
- User-data script that:
  - Installs Hermes via the official install script.
  - Clones or pulls the desired profile/skill set.
  - Configures `~/.hermes/config.yaml` and `.env` from SSM.
  - Starts the gateway and enables linger for the service user.
- OpenTofu module structure under `infrastructure/aws/hermes-gateway/`.
- Example `opentofu.tfvars` and `backend.tf` (S3 + DynamoDB for state).
- Full documentation in `docs/deployment/aws-with-opentofu.md`.
- `tofu validate`, `tofu fmt -check`, and basic GitHub Action for the module.

## Out of Scope (Future Epics / Child Issues)
- ECS Fargate or EKS deployment with auto-scaling and load balancer.
- RDS/Aurora replacement for SQLite session store.
- Multi-AZ, multi-region active-active setup.
- Cost optimization (Spot, Savings Plans, right-sizing).
- Integration with existing agentneo Kanban dispatcher (golden copy sync, etc.).
- Custom AMI baking or Packer.
- Monitoring dashboard (Grafana + CloudWatch).

## Acceptance Criteria
- [ ] `tofu init -backend=true` succeeds using local AWS credentials.
- [ ] `tofu plan` completes without errors using the admin AWS CLI profile.
- [ ] `tofu apply` provisions resources and the Hermes gateway becomes reachable (SSH + public webhook endpoint if enabled).
- [ ] After apply, `hermes doctor` and `hermes gateway status` report healthy inside the instance.
- [ ] A test Telegram/Discord message is received and processed by the deployed gateway.
- [ ] `tofu destroy` removes all resources cleanly with no orphaned items.
- [ ] No secrets in state or git; all sensitive values come from variables or SSM.
- [ ] Documentation includes copy-paste commands for the user’s exact environment.
- [ ] Follows Hermes security model (no broad IAM policies, encrypted EBS if possible, security groups minimal).

## Technical Architecture
```
infrastructure/aws/
├── hermes-gateway/
│   ├── main.tf              # VPC, subnets, SG, EC2, IAM, SSM params
│   ├── variables.tf
│   ├── outputs.tf
│   ├── user-data.sh.tftpl   # Template for instance bootstrap
│   ├── versions.tf
│   └── README.md
├── modules/
│   └── hermes-common/       # Reusable IAM policy, logging, etc.
└── examples/
    └── single-gateway/
        ├── main.tf
        ├── opentofu.tfvars.example
        └── backend.hcl.example
```

**Key Resources:**
- `aws_vpc`, `aws_subnet`, `aws_security_group`
- `aws_instance` (with user_data template)
- `aws_iam_role` + `aws_iam_instance_profile` + policy for logs + ssm
- `aws_ssm_parameter` for Hermes secrets
- `aws_cloudwatch_log_group`

**Provider:** AWS (region configurable, default us-east-1 or eu-west-1)

**State Backend Recommendation:** S3 bucket + DynamoDB table (created once, reused).

## Implementation Plan (Bite-sized Tasks)
Use subagent-driven-development + isolated worktrees per task.

### Task 1: Research & Design
**Objective:** Audit existing Hermes installation, Docker, gateway, and config mechanisms.

**Files:**
- Read: `scripts/install.sh`, `Dockerfile`, `gateway/run.py`, `hermes_cli/config.py`, `docs/user-guide/messaging.md`

**Steps:**
1. Run `hermes doctor` and note requirements.
2. Identify how gateway is started as a service.
3. Document required env vars and config keys for a headless gateway.
4. Decide between raw systemd + install script vs Docker on EC2.

**Verification:** Summary written to the epic or a research note.

### Task 2: Create OpenTofu Module Skeleton
**Objective:** Bootstrap the directory structure and basic provider config.

**Files to Create:**
- `infrastructure/aws/hermes-gateway/versions.tf`
- `infrastructure/aws/hermes-gateway/variables.tf`
- `infrastructure/aws/hermes-gateway/main.tf` (empty module)

**Step 1:** Write minimal `versions.tf` with OpenTofu 1.6+ and AWS provider ~5.0.

**Step 2:** Define core variables (region, instance_type, key_name, hermes_version, etc.).

**Verification:** `tofu init` and `tofu validate` pass in the worktree.

### Task 3: Networking & Security Groups
**Objective:** Create VPC, subnets, and tight security groups.

**Files:** `main.tf` (networking section)

**Step 1:** Add VPC + 2 AZ subnets (public for gateway).

**Step 2:** Security group allowing SSH (from admin IP or 0.0.0.0/0 with note) and gateway port (default 8080 or configurable) + webhook ports.

**Step 3:** Egress all (or restricted).

**Verification:** `tofu plan` shows correct resources.

### Task 4: IAM Role & Instance Profile
**Objective:** Least-privilege IAM for the EC2 instance.

**Files:** `main.tf` (IAM section) + `policies/hermes-gateway-policy.json`

**Required permissions:**
- `logs:CreateLogStream`, `logs:PutLogEvents`
- `ssm:GetParameter`, `ssm:GetParameters`
- Optional: `s3:*` for artifacts, `secretsmanager:*`

**Verification:** Policy passes IAM simulator or at least `tofu plan`.

### Task 5: EC2 Instance + User Data
**Objective:** Launch instance and bootstrap Hermes.

**Files:**
- `user-data.sh.tftpl` (template)
- `main.tf` (aws_instance resource)

**User data script must:**
1. Update system.
2. Install Hermes via curl | bash.
3. Create hermes user or use ec2-user.
4. Pull config from SSM Parameter Store into `~/.hermes/.env` and `config.yaml`.
5. Enable and start `hermes-gateway` systemd service (or `hermes gateway install` + start).
6. Set up log rotation for ~/.hermes/logs/.

**Verification:** After apply, SSH into instance and run `hermes gateway status`.

### Task 6: SSM Parameters & Secrets Injection
**Objective:** Store and inject Hermes secrets securely.

**Files:** Additional `aws_ssm_parameter` resources + variable for secret names.

**Example:** `hermes_telegram_token`, `hermes_openrouter_key`, etc.

**Verification:** Instance can read parameters without exposing them in `tofu show`.

### Task 7: CloudWatch Integration
**Objective:** Ship logs and basic alarms.

**Files:** `aws_cloudwatch_log_group`, CloudWatch agent config or direct journald forwarding.

**Verification:** Logs appear in CloudWatch console.

### Task 8: Documentation & Examples
**Objective:** Write user-facing docs and example usage.

**Files:**
- `docs/deployment/aws-with-opentofu.md`
- `infrastructure/aws/hermes-gateway/README.md`
- `examples/single-gateway/opentofu.tfvars.example`

**Content must include:**
- Prerequisites (AWS CLI, OpenTofu, SSH key).
- Step-by-step: `tofu init`, `tofu plan`, `tofu apply`.
- How to update config after deploy (SSM edit + instance reboot or signal).
- Destroy instructions.
- Cost estimate (t3.medium ~$15-20/mo + data).

### Task 9: Validation & Self-Test
**Objective:** Add a simple health-check test that can run after apply.

**Files:** Perhaps a small Python or shell script in the module that curls the gateway health endpoint or checks `hermes status`.

**Verification:** `tofu apply` + test script passes.

### Task 10: PR & Review
**Objective:** Open PR from the worktree branch, run CI, request review.

**Steps:**
1. Commit only the new module + docs (no unrelated changes).
2. Push and create PR via gh or MCP.
3. Ensure `tofu validate` and fmt checks pass in CI.
4. Self-review against agentneo-sdlc standards.

## Tradeoffs
- **EC2 vs Fargate:** EC2 chosen for MVP because Hermes gateway is stateful (SQLite sessions, local skills, long-running crons). Fargate is stateless by default and more complex.
- **Systemd vs Docker:** Systemd + native install script is simpler for first version; Docker can be added later.
- **Public IP vs Private + Bastion:** Public IP with tight SG for simplicity (admin machine can SSH). Production users should use private subnets + VPN/bastion.
- **OpenTofu State:** Local state is fine for personal use; S3 backend is documented as recommended.

## Risks
- AWS costs if left running.
- Security group misconfiguration exposing the gateway.
- User-data script complexity (idempotency, error handling).
- OpenTofu provider drift if Hermes install script changes.

## Definition of Done
- Merged PR with module + docs.
- User can follow the README on their admin machine and have a working Hermes gateway on AWS in < 10 minutes.
- Sentinel / groomer can later reference this module when deploying agentneo workers to AWS.

---

**Created from user request on 2026-05-20**  
**Next action:** Groomer decomposes into child issues or worker picks up Task 1.