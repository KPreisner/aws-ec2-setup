# AWS EC2 setup

How to set up an Ubuntu EC2 box for running Docker Compose apps and doing data work over SSH, from
a Mac.

| Step | File |
|---|---|
| 1. Launch the instance in the console (type, disks, network, security) | [01-launch-instance.md](01-launch-instance.md) |
| 2. Give it a fixed IP and connect over SSH (and VS Code) | [02-connect.md](02-connect.md) |
| 3. Base OS setup: updates, swap, the data volume | [03-base-setup.md](03-base-setup.md) |
| 4. Install Docker, Task, uv, the AWS CLI, Claude Code, and Git | [04-install-tools.md](04-install-tools.md) |
| 5. Run a Compose app and reach it through an SSH tunnel | [05-run-a-compose-app.md](05-run-a-compose-app.md) |
| 6. Costs, stopping, and cleaning up | [06-costs-and-cleanup.md](06-costs-and-cleanup.md) |

## Placeholders

Replace these throughout:

| Placeholder | Meaning | Example |
|---|---|---|
| `<name>` | The instance's Name tag | `myapp-staging` |
| `<host>` | SSH alias on the Mac, the instance's hostname | `myapp-ec2` |
| `<key-name>` | EC2 key pair, saved as `~/.ssh/<key-name>.pem` | `aws-myapp` |
| `<region>` | Your AWS Region | `us-east-2` |
| `<timezone>` | The box's timezone (`timedatectl list-timezones`) | `America/New_York` |
| `<app>` | The app's name, and its data directory `/data/<app>` | `myapp` |
| `<repo>` | The app's GitHub repo | `myapp` |
| `<github-user>` | Your GitHub username | |
| `<your name>`, `<your email>` | For git commits | |
| `<instance-id>` | From EC2 → Instances | `i-0123456789abcdef0` |
| `<elastic-ip>` | From step 2 | |
| `<your-domain>` | A domain you control, for going public later | `example.com` |

## Before you start

- **Region:** create everything in one Region. On the new AWS experience, each project has a
  single Region; confirm it in AWS Settings → View all projects → Overview → Additional Info →
  Region.
- **Spend limit:** if your project has one (AWS Settings → Billing), hitting it pauses the
  project. Sudden "Access Denied" errors on things that used to work usually mean that.
- **CLI on the Mac:** `aws login` signs in; run it again when the session expires.

## vCPU quotas

EC2 limits how many vCPUs can run at once, per instance family group. New accounts often have a
modest Standard quota and **0** for GPU instances:

| Quota | Code |
|---|---|
| Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) | `L-1216C47A` |
| Running On-Demand G and VT (GPU) | `L-DB2E81BA` |

Check yours:

```
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A --query "Quota.Value"
aws service-quotas get-service-quota --service-code ec2 --quota-code L-DB2E81BA --query "Quota.Value"
```

A GPU instance fails to launch with `VcpuLimitExceeded` until the G quota is raised: **Service
Quotas → Amazon EC2 → Running On-Demand G and VT instances → Request increase** (8 covers a
g4dn/g5/g6 `.2xlarge`). Approval can take a day or more.

## Picking an instance type

These guides use **m7i-flex.2xlarge** (8 vCPU, 32 GiB, $0.383/hour, ~$280/month nonstop): enough
memory for a mid-sized data app, cheap enough to leave on. For big builds, resize it to
**m7i-flex.4xlarge** for the day, then back (see [05-run-a-compose-app.md](05-run-a-compose-app.md)).
Size from real numbers once the app runs (`docker stats`).

16 vCPU / 64 GiB options, `us-east-2` on-demand Linux prices (October 2026):

| Type | CPU | $/hour | ~$/month nonstop |
|---|---|---|---|
| m7i-flex.4xlarge | Intel Sapphire Rapids, x86 | 0.766 | ~$560 |
| m7i.4xlarge | Intel Sapphire Rapids, x86 | 0.806 | ~$590 |
| m8i.4xlarge | Intel Granite Rapids, x86 | 0.847 | ~$620 |
| m7a.4xlarge | AMD EPYC Genoa, x86 | 0.927 | ~$675 |
| m8g.4xlarge | Graviton4, arm64 | 0.718 | ~$525 |
| m7g.4xlarge | Graviton3, arm64 | 0.653 | ~$475 |
| r7i.2xlarge | Intel, x86, 8 vCPU / 64 GiB | 0.529 | ~$385 |

GPU, for later:

| Type | vCPU | RAM + GPU | ~$/month nonstop |
|---|---|---|---|
| g4dn.xlarge | 4 | 16 GB + T4 16 GB | ~$385 |
| g6.xlarge | 4 | 16 GB + L4 24 GB | ~$590 |

x86 keeps images and binaries portable between a CPU box and the GPU types; arm64 (Graviton) is
cheaper and matches an Apple Silicon Mac's images, but not the GPU boxes. Changing type within
the same architecture is stop → Change instance type → start; disks and the Elastic IP stay.
