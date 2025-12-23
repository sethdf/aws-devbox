# AWS DevBox

Terraform configuration for a reliable AWS cloud development workstation with spot pricing.

## Why This Configuration?

**Instance: m7a.xlarge (AMD EPYC Gen 4)**
- 4 vCPUs, 16 GB RAM
- **~$0.06/hour spot** (vs $0.20 on-demand) = ~70% savings
- No CPU throttling (unlike t3 burstable instances)

**OS: Ubuntu Server 24.04 LTS**
- Native VS Code Remote support
- 5+ years of support
- Best package availability

**Storage: gp3 with upgraded throughput**
- 100 GB, 3000 IOPS, 250 MiB/s throughput
- Faster `npm install`, `git clone`, etc.
- Daily snapshots via DLM (7-day retention)

**Spot + Hibernation**
- Auto-hibernates on spot interruption (saves RAM state)
- Auto-restarts when capacity returns
- Email notifications on interruption/restart

## Quick Start

```bash
# 1. Configure AWS credentials
aws configure
# or
export AWS_PROFILE=your-profile

# 2. Create terraform.tfvars
cat > terraform.tfvars <<EOF
ssh_public_key     = "ssh-ed25519 AAAA... your-key"
allowed_ssh_cidrs  = ["YOUR.IP.ADDRESS/32"]
aws_region         = "us-east-1"
notification_email = "you@example.com"  # optional
EOF

# 3. Deploy
terraform init
terraform plan
terraform apply

# 4. Connect
ssh ubuntu@$(terraform output -raw public_ip)
```

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `aws_region` | us-east-1 | AWS region |
| `instance_type` | m7a.xlarge | EC2 instance type |
| `volume_size` | 100 | Root volume size (GB) |
| `volume_iops` | 3000 | gp3 IOPS |
| `volume_throughput` | 250 | gp3 throughput (MiB/s) |
| `ssh_public_key` | (required) | Your SSH public key |
| `allowed_ssh_cidrs` | 0.0.0.0/0 | CIDRs allowed to SSH |
| `snapshot_retention_days` | 7 | Days to keep snapshots |
| `use_spot` | true | Use spot instances (~70% savings) |
| `spot_max_price` | "" | Max spot price (empty = on-demand cap) |
| `notification_email` | "" | Email for spot interruption alerts |
| `spot_restart_attempts` | 5 | Retry attempts before giving up |
| `enable_schedule` | true | Enable auto start/stop schedule |
| `schedule_start` | 0 5 * * ? * | Cron for auto-start (5am) |
| `schedule_stop` | 0 23 * * ? * | Cron for auto-stop (11pm) |
| `schedule_timezone` | America/Denver | Timezone for schedule (Mountain) |

## Spot Instance Behavior

**How it works:**
1. Instance runs on spot pricing (~$0.06/hr)
2. If AWS reclaims capacity, you get 2-minute warning
3. Instance auto-hibernates (RAM saved to EBS)
4. Lambda auto-restarts when capacity returns
5. Instance resumes exactly where you left off

**Interruption frequency:** m7a.xlarge typically <5% monthly

**What happens on interruption:**
```
Spot interrupted → Auto-hibernate (2 min) → Lambda detects stopped state
    → Attempts restart (up to 5 tries with backoff)
    → Success: resumes from hibernation
    → Failure: sends email notification
```

**To disable spot and use on-demand:**
```hcl
use_spot = false
```

## Scheduled Start/Stop

By default, the instance:
- **Hibernates at 11pm** Mountain Time (saves RAM state)
- **Starts at 5am** Mountain Time (resumes from hibernation)

This runs 18 hours/day instead of 24, saving ~25% more on top of spot savings.

**To customize the schedule:**
```hcl
schedule_start    = "0 6 * * ? *"   # 6am
schedule_stop     = "0 22 * * ? *"  # 10pm
schedule_timezone = "America/New_York"  # Eastern
```

**To disable scheduling:**
```hcl
enable_schedule = false
```

**Working late?** Just start the instance manually - it will auto-stop at 11pm as usual, or you can stop it yourself when done.

## What's Installed

### Core Development
- Docker + Docker Compose
- Node.js LTS
- Python 3 + pip + venv
- Zsh + Oh My Zsh
- git, curl, jq, htop, tmux

### Productivity Tools
| Tool | Description | Alias |
|------|-------------|-------|
| `fzf` | Fuzzy finder (Ctrl-r for history) | - |
| `zoxide` | Smarter cd that learns your dirs | `z` |
| `direnv` | Auto-load .envrc per directory | - |
| `lazygit` | Git TUI | `lg` |
| `lazydocker` | Docker TUI | `ld` |
| `delta` | Better git diffs (auto-configured) | - |
| `mise` | Version manager (node/python/go) | - |
| `eza` | Modern ls with git status | `ls`, `ll`, `lt` |
| `bat` | Cat with syntax highlighting | `cat` |
| `ripgrep` | Fast grep | `rg` |
| `fd` | Fast find | `fd` |
| `ncdu` | Interactive disk usage | `ncdu` |
| `tldr` | Simplified man pages | `tldr` |

### Cloud Provider CLIs
| Tool | Description | Alias |
|------|-------------|-------|
| AWS CLI v2 | Amazon Web Services | `aws-whoami` |
| AWS SSM Plugin | Session Manager support | - |
| Azure CLI | Microsoft Azure | `az-whoami` |
| Google Cloud CLI | GCP | `gcp-whoami` |

### Windows / Microsoft 365 Admin
| Tool | Description |
|------|-------------|
| PowerShell Core | Cross-platform PowerShell (`pwsh`) |
| CLI for Microsoft 365 | M365 administration (`m365`) |
| Microsoft.Graph | PowerShell module for Graph API |
| Az | PowerShell module for Azure |
| ExchangeOnlineManagement | Exchange Online admin |
| MicrosoftTeams | Teams admin |

## VS Code Remote Setup

Add to `~/.ssh/config`:

```
Host devbox
    HostName <elastic-ip>
    User ubuntu
    IdentityFile ~/.ssh/your-key
```

Then in VS Code: `Remote-SSH: Connect to Host...` → `devbox`

## Cloud Authentication

```bash
# AWS
aws configure
# or
aws sso login --profile your-profile

# Azure
az login

# GCP
gcloud auth login
gcloud config set project YOUR_PROJECT

# Microsoft 365
m365 login

# PowerShell (Graph/Exchange/Teams)
pwsh
Connect-MgGraph -Scopes "User.Read.All"
Connect-ExchangeOnline
Connect-MicrosoftTeams
```

## Manual Controls

```bash
# Stop instance (compute charges stop, EBS continues)
aws ec2 stop-instances --instance-ids $(terraform output -raw instance_id)

# Hibernate instance (saves RAM to disk)
aws ec2 stop-instances --instance-ids $(terraform output -raw instance_id) --hibernate

# Start instance
aws ec2 start-instances --instance-ids $(terraform output -raw instance_id)
```

## Cost Comparison

| Configuration | Monthly Cost |
|---------------|--------------|
| On-demand 24/7 | ~$150 |
| On-demand 50hr/week | ~$43 |
| Spot 24/7 | ~$45 |
| Spot + schedule (18hr/day) | ~$33 |
| **Spot + schedule (actual ~10hr/day use)** | **~$18** |
| Stopped (storage only) | ~$10 |

## Destroy

```bash
terraform destroy
```

Note: Root volume has `delete_on_termination = false` for safety. Delete manually if needed.
