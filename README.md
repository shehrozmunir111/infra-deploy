# infra-deploy — AWS deployment for HealthPA + expense-forecasting

A free-tier-friendly, production-shaped deployment (Terraform IaC) that provisions
**one** AWS environment and runs **both** apps on it. This repo owns no application
code — it only describes the cloud. The two app repos are cloned onto the host at boot.

## What you get

```
                    ┌──────────────────────── AWS VPC ───────────────────────────┐
   Internet ──►  EC2 t3.micro  (1 GB RAM + 6 GB swap)                             │
       :80  ─────────►  HealthPA          (Caddy + FastAPI + Celery + Redis)      │
       :8080 ────────►  expense-forecasting (Caddy + FastAPI + Celery + Redis)    │
                    │        │                                                     │
                    │        └──► RDS PostgreSQL (private subnet) ───────────────┐ │
                    │                 ├── healthpa            (HealthPA's DB)     │ │
                    │                 └── expenseforecast_db  (expense's DB)      │ │
                    │   EC2 IAM role ─► SES (email), S3 (uploads), SSM (secrets)  │ │
                    └─────────────────────────────────────────────────────────────┘
```

| Concern | How it's handled | Résumé talking point |
|---|---|---|
| Infra | 100% Terraform, dedicated infra repo | "Infrastructure as Code, platform repo" |
| Secrets | SSM Parameter Store (SecureString) + EC2 IAM role | "No static credentials; least-privilege IAM" |
| DB | One managed RDS, two databases, private subnet | "Managed Postgres, network isolation" |
| Background jobs | Celery worker + beat (both apps) | "Async task queues" |
| TLS | Caddy auto-issues Let's Encrypt (with a domain) | "Automated HTTPS" |
| Email | SES via instance role | "Transactional email on SES" |
| Reboots | systemd unit per app | "Self-healing host" |

**Cost:** $0 for ~12 months on a new AWS account (t3.micro EC2, db.t3.micro RDS, S3,
SES all free-tier). Both apps share the **one** free instance. After 12 months ~$15–20/mo.

> ⚠️ **RAM is tight by design.** Two full stacks on a 1 GB t3.micro (with 6 GB swap) is
> intentional to stay free. If it struggles, stop a Celery beat or upgrade `instance_type`
> to `t3.small` (just change the var and `terraform apply`).

---

## Repos involved

| Repo | Role |
|---|---|
| **infra-deploy** (this) | Terraform — provisions AWS, deploys both apps |
| **HealthPA** | App + its `infra/docker-compose.prod.yml` (served on :80) |
| **expense-forecasting** | App + its `infra/docker-compose.prod.yml` (served on :8080) |

Both app repos must be pushed to GitHub before deploying — the EC2 host clones them.

---

## Prerequisites (one-time)

1. **AWS account** + [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html):
   ```bash
   aws configure                 # admin Access Key / Secret, region us-east-1
   aws sts get-caller-identity   # confirm
   ```
2. **Terraform** ≥ 1.5 — https://developer.hashicorp.com/terraform/install
3. **Verify SES** so email sends (HealthPA uses it):
   ```bash
   aws ses verify-email-identity --email-address noreply@yourdomain.com --region us-east-1
   # click the link AWS emails you. Sandbox accounts must also verify test recipients.
   ```
4. **EC2 key pair** (for SSH):
   ```bash
   aws ec2 create-key-pair --key-name healthpa-key \
     --query 'KeyMaterial' --output text > healthpa-key.pem
   chmod 600 healthpa-key.pem
   ```
5. **Push both app repos** to GitHub. Private? Use `https://<TOKEN>@github.com/you/<repo>.git`.

---

## Deploy

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:
- `ssh_allowed_cidr` → `curl ifconfig.me` then add `/32`
- `ec2_key_name` → `healthpa-key`
- `git_repo_url` + `expense_git_repo_url` → your two repo URLs
- `db_password`, `app_secret_key` (`python -c "import secrets; print(secrets.token_hex(32))"`)
- API keys: `groq_api_key`, `openai_api_key`, `pinecone_api_key`, `anthropic_api_key`
- `ses_sender_email`, `admin_email`

Then:

```bash
terraform init
terraform plan      # ~27 resources
terraform apply     # type "yes"
```

RDS takes ~5–10 min. Outputs:

```
app_url      = "http://<elastic-ip>"
expense_url  = "http://<elastic-ip>:8080"
ssh_command  = "ssh -i <your-key>.pem ec2-user@<elastic-ip>"
```

The host then needs ~5–10 min (slower because it builds **two** stacks sequentially on a
tiny box). Watch it:

```bash
ssh -i healthpa-key.pem ec2-user@<elastic-ip>
sudo tail -f /var/log/cloud-init-output.log
```

Open `app_url` (HealthPA) and `expense_url` (expense-forecasting). Done. 🎉

---

## Add a domain + HTTPS (optional)

Point an A record at `app_public_ip`, set `domain_name` in tfvars, `terraform apply`.
Caddy auto-issues a Let's Encrypt cert for HealthPA on :443. (For a pretty domain on the
second app too, ask me to add a shared edge proxy with subdomain routing.)

---

## Day-2 operations

Each app lives in its own directory on the host:
- HealthPA → `/opt/healthpa`
- expense  → `/opt/expense`

**Status / logs:**
```bash
cd /opt/healthpa && sudo docker compose -f infra/docker-compose.prod.yml ps
cd /opt/expense  && sudo docker compose -f infra/docker-compose.prod.yml logs -f celery_worker
```

**Change a secret:** edit it in `terraform.tfvars` → `terraform apply` (rewrites SSM), then on the host re-pull the env:
```bash
# HealthPA:
sudo aws ssm get-parameter --region us-east-1 --name /healthpa/env \
  --with-decryption --query 'Parameter.Value' --output text | sudo tee /opt/healthpa/.env >/dev/null
cd /opt/healthpa && sudo docker compose -f infra/docker-compose.prod.yml up -d
# expense uses /healthpa/expense-env  →  /opt/expense/.env
```

**Deploy new app code:**
```bash
cd /opt/healthpa && sudo git pull && sudo docker compose -f infra/docker-compose.prod.yml up -d --build
sudo docker compose -f infra/docker-compose.prod.yml exec -T api alembic upgrade head   # HealthPA only
cd /opt/expense  && sudo git pull && sudo docker compose -f infra/docker-compose.prod.yml up -d --build
```

**RAM relief** (if the box thrashes): `sudo docker compose -f infra/docker-compose.prod.yml stop celery_beat` in the less-critical app.

---

## Tear down (stop all billing)

```bash
cd terraform
terraform destroy
```
Deletes EC2, RDS (both DBs), S3, IAM, VPC. Empty the S3 bucket first if it complains.

---

## Notes & gotchas

- **Single instance = single point of failure.** Fine for portfolio/demo. Real HA = ECS Fargate + ALB (not free).
- **Two stacks on 1 GB** is the tight-but-free choice. 6 GB swap + sequential build + `--concurrency=1` keep it alive. `t3.small` is the easy upgrade.
- **expense uses a second database** (`expenseforecast_db`) on the same RDS server — created automatically at boot via psql.
- **Free-tier guardrails:** RDS autoscaling off, Multi-AZ off. Add an [AWS Budgets](https://console.aws.amazon.com/billing/home#/budgets) $1 alert.
- **No static AWS keys:** boto3 uses the EC2 instance role; the production `.env` ships with empty AWS keys on purpose.
- **CI/CD level-up:** ask me to add a GitHub Actions workflow (build → ECR → host pulls) per app.
