# 1. Launch the instance

In the AWS Management Console, check the Region in the top right is your project's Region
(`<region>`), then go to **EC2 → Instances → Launch instances**.

## Name and tags

- **Name:** something descriptive, e.g. `<name>`.

## Application and OS image (AMI)

- **Quick Start → Ubuntu → Ubuntu Server 26.04 LTS (HVM), SSD Volume Type**, the latest LTS.
  Pick 24.04 LTS only if something you need (e.g. a specific NVIDIA driver or CUDA version)
  doesn't support 26.04 yet.
- **Architecture:** **64-bit (x86)**. It has to match the instance type: m7i-flex is x86. Use
  64-bit (Arm) only for Graviton types (`t4g`, `m7g`, …).

## Instance type

- **m7i-flex.2xlarge** (8 vCPU, 32 GiB), see the [README](README.md). "Flex" means full speed
  about 95% of the time at 5% less than m7i, with no CPU-credit accounting like the `t` types, so
  no surprise burst charges.
- $0.383/hour, ~$280/month if never stopped: check it fits under your spend limit if you plan to leave it on.
- Need more for a big build? Resize to m7i-flex.4xlarge (16 vCPU, 64 GiB) temporarily, see
  [05-run-a-compose-app.md](05-run-a-compose-app.md).

## Key pair (login)

- **Create new key pair:** name `<key-name>` (e.g. `aws-myapp`), type **ED25519**, format
  **.pem**. The browser downloads it once; you can't download it again.
- Move it into place on the Mac right away (see [02-connect.md](02-connect.md)).

## Network settings

Click **Edit**:

- **VPC / subnet:** the default VPC, any subnet (or "No preference"). Note which **Availability
  Zone** it lands in (e.g. `<region>a`) once it's running: EBS volumes only attach to instances
  in the same zone.
- **Auto-assign public IP:** Enable (we swap it for an Elastic IP in step 2).
- **Firewall (security groups):** Create security group, name `ssh-from-home`:
  - Rule: **SSH**, TCP 22, source **My IP**.
  - **Don't** tick "Allow HTTPS/HTTP from the internet" yet. Open 80/443 only once the app has TLS
    and is ready to be public. Until then, reach web UIs through an SSH tunnel (step 5).
  - Never use `0.0.0.0/0` for SSH. If your home IP changes, edit the rule (step 2).

## Configure storage

Two volumes, both **gp3**, both **encrypted**:

| Volume | Size | Why |
|---|---|---|
| Root (`/`) | **40 GiB** | OS, Docker images and build cache (`/var/lib/docker`), swap file |
| Data (`/data`) | **100 GiB** to start | Datasets, models, outputs |

- **gp3**, not gp2: cheaper per GB and the baseline (3,000 IOPS, 125 MB/s) doesn't depend on
  size. The defaults are fine; raise throughput later only if a workload is disk-bound.
- Don't use io1/io2 (expensive, for databases with steady high IOPS) or st1/sc1 (HDD).
- Click **Advanced** on each volume:
  - **Encrypted:** Yes, KMS key `aws/ebs` (the default; no extra cost).
  - Root: **Delete on termination: Yes**.
  - Data: **Delete on termination: No.** This is the point of a separate data volume: it
    outlives the instance, so you can later detach it and attach it to a bigger or GPU instance
    in the same Availability Zone without copying anything.
- Click **Add new volume** for the data volume (device name `/dev/sdb` is fine; Ubuntu shows it
  as `/dev/nvme1n1`).
- gp3 volumes can grow later without downtime (EC2 → Volumes → Modify, then grow the filesystem),
  but can't shrink. Start small.

Instance store (NVMe that comes with some types like `g4dn`, `m7id`) is wiped on every stop, so
it's only for scratch space. m7i-flex has none.

## Advanced details

Expand the section and set:

- **Termination protection:** Enable. Stops a stray click from deleting the box.
- **Metadata version:** **V2 only (token required)**. Blocks the SSRF trick of stealing instance
  credentials through the metadata service.
- **Metadata response hop limit:** **2**. Lets processes inside Docker containers reach the
  metadata service (needed once the box has an IAM role, e.g. for S3). The default of 1 makes
  those requests time out.
- **IAM instance profile:** none for now. Add a role later when the box needs S3; never put access
  keys on the instance.
- Leave everything else at its defaults.

## Launch

Check the **Summary** panel (1 instance, m7i-flex.2xlarge, 2 volumes), then **Launch instance**.
Wait until **Instance state** is Running and **Status check** shows 3/3 passed (a minute or two).
