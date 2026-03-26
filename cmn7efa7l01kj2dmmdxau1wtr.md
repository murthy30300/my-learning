---
title: "GCP to AWS Migration Using MGN (Zero Downtime Guide)"
seoTitle: "GCP to AWS Migration Using MGN (Step-by-Step Guide)"
seoDescription: "Fix kernel boot errors after GCP to AWS migration and learn zero-downtime VM migration using AWS MGN with step-by-step guidance."
datePublished: 2026-03-26T11:38:14.774Z
cuid: cmn7efa7l01kj2dmmdxau1wtr
slug: gcp-to-aws-migratigcp-to-aws-migration-using-mgn-zero-downtime
cover: https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/dcc3bd0c-a199-4e29-8624-e44010ee7012.png
ogImage: https://cdn.hashnode.com/uploads/og-images/66f590c9120870cbbe3a72f4/1e415712-c1d1-4523-b86a-bb9a9c5b08df.png
tags: ec2, aws, cloud-computing, devops, terraform, gcp, cloud-migration, aws-mgn

---

# How to Migrate a GCP VM to AWS Using Application Migration Service (MGN)

> AWS Application Migration Service (MGN) is the recommended way to migrate GCP VMs to AWS. It installs a lightweight replication agent on the source server, continuously replicates disk data over the internet, and lets you launch test instances before final cutover — with zero downtime.

---

> 🏠 **Think of it like moving houses.** You don't move everything in one chaotic night — you hire professional movers (MGN) who slowly shift your belongings to the new house while you're still living in the old one. Only when the new house is fully set up and verified do you hand over the keys and officially "move in." That's exactly what MGN does — live, continuous migration with a test-drive before commitment.

---

## Why Application Migration Service?

When you need to move a running server from GCP to AWS, MGN is preferred over disk snapshot when:

- You want continuous replication with minimal cutover downtime
- You need to migrate multiple VMs from a single dashboard
- You want to test the migrated instance before committing to cutover
- You prefer a managed service over manual disk conversion

> 📦 **The snapshot approach** (moving boxes yourself one weekend) works fine for simple cases — but it's a one-shot deal. If something goes wrong, you're stuck. MGN is the professional moving company that keeps syncing your data right up until the moment you cut over.
>
> If you've been down the manual snapshot road before, you'll know the pain. In a previous guide — **[GCP to AWS VM Migration: Disk Snapshot Guide](https://vishnunukala.hashnode.dev/gcp-to-aws-vm-migration-disk-snapshot-guide)** — we walked through that entire process step by step. This article picks up where that leaves off, offering a smarter, more automated path for production workloads.

The general flow is:
**GCP VM → MGN Agent → Replication Server (AWS) → EBS Staging Volume → Test EC2 → Cutover EC2**

> 🔄 **Breaking down the flow in plain English:**
> - **GCP VM** = Your old apartment you're moving out of
> - **MGN Agent** = The moving crew you hired (installed on your old server)
> - **Replication Server (AWS)** = The moving truck parked between both locations
> - **EBS Staging Volume** = The temporary storage room in the new building
> - **Test EC2** = A trial run — walking through the new apartment before signing the final lease
> - **Cutover EC2** = You've officially moved in. New address, new life. 🎉

---

## Prerequisites

- A running GCP VM
- AWS CLI configured with company profile (`aws configure --profile myorg`)
- GCP CLI (`gcloud`) configured with company account
- Terraform installed (>= 1.5.0)
- AWS MGN initialized in your account (ap-south-1)
- Target AWS VPC with at least one public subnet (for MGN replication server)

---

## Stage 1 — Configure CLI for Company Accounts

Before running any commands, switch both CLIs from personal to company accounts.

> 🔑 **Layman analogy:** Imagine you have two email accounts — personal Gmail and work Gmail. Before sending that important company email, you need to make sure you're logged into the *work* account, not your personal one. That's exactly what this step does — it ensures every cloud command goes to the right account.

### AWS CLI

```bash
# Add company profile
aws configure --profile myorg
# Enter: Access Key ID, Secret Key, region (ap-south-1), output (json)

# Verify
aws sts get-caller-identity --profile myorg --region ap-south-1
```

### GCP CLI

```bash
# Add company account
gcloud auth login
# Opens browser — sign in with company Google account

# Create and activate company config
gcloud config configurations create myorg
gcloud config set account your-company@email.com
gcloud config set project my-gcp-project
gcloud config configurations activate myorg

# Verify
gcloud config list
```

> **Why named profiles?** Using `--profile myorg` in every AWS command ensures you never accidentally run commands against your personal account.

---

## Stage 2 — Terraform: AWS Infrastructure

Terraform provisions all AWS-side resources required before agent installation. Create the following file structure:

> 🏗️ **What is Terraform doing here?** Think of Terraform as your architect who draws the blueprint and then automatically builds the rooms before you move in. Instead of you manually clicking around the AWS console to create subnets, security groups, and IAM users, Terraform does it all programmatically — and repeatably. One file to rule them all.

```
migration/
  aws-infra/
    main.tf
    variables.tf
    outputs.tf
  gcp-infra/
    main.tf
    variables.tf
    outputs.tf
```

### aws-infra/main.tf (core resources)

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region  = "ap-south-1"
  profile = "myorg"
}

data "aws_vpc" "existing" { id = var.vpc_id }

data "aws_internet_gateway" "existing" {
  filter { name = "attachment.vpc-id", values = [var.vpc_id] }
}

# Public subnet for MGN replication server
resource "aws_subnet" "mgn_public" {
  vpc_id                  = data.aws_vpc.existing.id
  cidr_block              = var.mgn_public_subnet_cidr
  availability_zone       = "ap-south-1c"
  map_public_ip_on_launch = true
  tags = { Name = "myapp-db-stage-mgn-public" }
}

resource "aws_route_table" "mgn_public" {
  vpc_id = data.aws_vpc.existing.id
  route { cidr_block = "0.0.0.0/0", gateway_id = data.aws_internet_gateway.existing.id }
  tags = { Name = "myapp-db-stage-mgn-rt" }
}

resource "aws_route_table_association" "mgn_public" {
  subnet_id      = aws_subnet.mgn_public.id
  route_table_id = aws_route_table.mgn_public.id
}

# IAM — MGN Agent User (credentials installed on GCP VMs)
resource "aws_iam_user" "mgn_agent" {
  name = "myapp-db-stage-mgn-agent"
}

resource "aws_iam_user_policy_attachment" "mgn_agent" {
  user       = aws_iam_user.mgn_agent.name
  policy_arn = "arn:aws:iam::aws:policy/AWSApplicationMigrationFullAccess"
}

resource "aws_iam_access_key" "mgn_agent" {
  user = aws_iam_user.mgn_agent.name
}

# IAM — EC2 role for launched instances
resource "aws_iam_role" "mgn_ec2" {
  name = "myapp-db-stage-mgn-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Principal = { Service = "ec2.amazonaws.com" }, Action = "sts:AssumeRole" }]
  })
}

resource "aws_iam_role_policy_attachment" "mgn_ec2" {
  role       = aws_iam_role.mgn_ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AWSApplicationMigrationEC2Access"
}

resource "aws_iam_instance_profile" "mgn_ec2" {
  name = "myapp-db-stage-mgn-ec2-profile"
  role = aws_iam_role.mgn_ec2.name
}

# Security Group — MGN Replication Server
resource "aws_security_group" "mgn_replication" {
  name   = "myapp-db-stage-mgn-replication-sg"
  vpc_id = data.aws_vpc.existing.id

  ingress {
    description = "MGN agent from GCP VM"
    from_port = 1500; to_port = 1500; protocol = "tcp"
    cidr_blocks = ["${var.gcp_vm_public_ip}/32"]
  }
  ingress { from_port = 443; to_port = 443; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0;   to_port = 0;   protocol = "-1";  cidr_blocks = ["0.0.0.0/0"] }
  tags = { Name = "myapp-db-stage-mgn-replication-sg" }
}

# Security Group — Target EC2
resource "aws_security_group" "target_ec2" {
  name   = "myapp-db-stage-ec2-sg"
  vpc_id = data.aws_vpc.existing.id

  ingress { description = "SSH within VPC"; from_port = 22;   to_port = 22;   protocol = "tcp"; cidr_blocks = [data.aws_vpc.existing.cidr_block] }
  ingress { description = "PostgreSQL";     from_port = 5432; to_port = 5432; protocol = "tcp"; cidr_blocks = [data.aws_vpc.existing.cidr_block] }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
  tags = { Name = "myapp-db-stage-ec2-sg" }
}
```

> 🔐 **IAM Users & Security Groups in plain English:**
> - **IAM user (`mgn_agent`)** = A special ID badge you create *just* for the moving crew (the agent). It has exactly the permissions it needs — nothing more. You don't hand the movers your master key.
> - **Security Groups** = The bouncer at the door. The replication server's SG only lets traffic in from your specific GCP VM's IP on port 1500 — nobody else gets in.

### aws-infra/variables.tf

```hcl
variable "vpc_id"                { default = "vpc-0abc1234def56789a" }
variable "gcp_vm_public_ip"      { default = "203.0.113.45" }
variable "mgn_public_subnet_cidr"{ default = "<FREE_CIDR>" }  # e.g. 10.0.32.0/24
```

### aws-infra/outputs.tf

```hcl
output "mgn_agent_access_key" { value = aws_iam_access_key.mgn_agent.id;     sensitive = true }
output "mgn_agent_secret_key" { value = aws_iam_access_key.mgn_agent.secret; sensitive = true }
output "mgn_replication_sg_id"{ value = aws_security_group.mgn_replication.id }
output "target_ec2_sg_id"     { value = aws_security_group.target_ec2.id }

output "agent_install_command" {
  sensitive = true
  value = <<-EOT
    sudo wget -O ./aws-replication-installer-init.py \
      https://aws-application-migration-service-ap-south-1.s3.ap-south-1.amazonaws.com/latest/linux/aws-replication-installer-init.py
    sudo python3 aws-replication-installer-init.py \
      --region ap-south-1 \
      --aws-access-key-id ${aws_iam_access_key.mgn_agent.id} \
      --aws-secret-access-key ${aws_iam_access_key.mgn_agent.secret}
  EOT
}
```

### gcp-infra/main.tf

```hcl
terraform {
  required_providers {
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
}

provider "google" {
  project     = var.gcp_project_id
  zone        = var.gcp_zone
  credentials = file(var.gcp_credentials_file)
}

data "google_compute_instance" "vm" {
  name    = var.gcp_vm_name
  zone    = var.gcp_zone
  project = var.gcp_project_id
}

# Firewall rule — allow MGN egress from tagged VMs
resource "google_compute_firewall" "mgn_egress" {
  name      = "myapp-db-stage-mgn-egress"
  network   = data.google_compute_instance.vm.network_interface[0].network
  direction = "EGRESS"
  priority  = 1000
  allow { protocol = "tcp"; ports = ["443", "1500"] }
  target_tags        = ["myapp-db-stage-mgn"]
  destination_ranges = ["0.0.0.0/0"]
}
```

### gcp-infra/outputs.tf

```hcl
output "vm_external_ip" {
  value = data.google_compute_instance.vm.network_interface[0].access_config[0].nat_ip
}
output "vm_zone" { value = var.gcp_zone }
```

---

## Stage 3 — Find a Free CIDR Block

Before applying Terraform, find a free subnet CIDR in your VPC:

> 📍 **CIDR in plain English:** A CIDR block is like an address range in an apartment building. If floors 1–10 are already occupied (10.0.0.0/20), floor 11 is used (10.0.16.0/20), and floors 12–20 are maintenance (10.0.128.0/20) — you need to find a **free floor** (like 10.0.32.0/24) for the moving crew's temporary office. That's your MGN staging subnet.

```bash
# Check VPC CIDR
aws ec2 describe-vpcs \
  --vpc-ids vpc-0abc1234def56789a \
  --query "Vpcs[0].CidrBlock" \
  --output text --profile myorg --region ap-south-1

# Check existing subnet CIDRs
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0abc1234def56789a" \
  --query "Subnets[*].CidrBlock" \
  --output text --profile myorg --region ap-south-1
```

Pick a `/24` block that doesn't overlap. Example: if existing subnets use `10.0.0.0/20`, `10.0.16.0/20`, `10.0.128.0/20`, `10.0.144.0/20` — use `10.0.32.0/24`.

---

## Stage 4 — Deploy Terraform

```bash
# Step 1 — GCP infra first (get VM public IP)
cd migration/gcp-infra
terraform init && terraform apply
terraform output vm_external_ip   # copy this IP → paste into aws-infra/variables.tf

# Step 2 — AWS infra
cd ../aws-infra
terraform init
terraform plan   # verify: should show only new resources
terraform apply

# Step 3 — Get agent install credentials
terraform output -raw mgn_agent_access_key
terraform output -raw mgn_agent_secret_key
```

> **Important:** Always run `terraform plan` before `apply` to confirm `0 to change, 0 to destroy` on existing resources.

> 🛡️ **Why GCP infra first?** Because you need the GCP VM's public IP before you can lock down the AWS security group to only accept traffic from it. It's like getting the moving truck's license plate number before you tell the building security which truck to let through.

---

## Stage 5 — Initialize AWS MGN

MGN must be initialized once per AWS account per region before IAM policies become available.

> 🎬 **Think of initialization as registering the moving service in your city.** Before the movers can legally operate, they need to file paperwork with the local authorities. You only do this once. Every migration after that uses the same registered service.

```bash
# Initialize via console
# Go to: https://ap-south-1.console.aws.amazon.com/mgn/home
# Click "Get started" → "Set up service"

# Verify initialization
aws mgn describe-replication-configuration-templates \
  --profile myorg --region ap-south-1
# Should return {"items": [...]} not UninitializedAccountException
```

### Update MGN Replication Template

After initialization, point the template to your VPC's public subnet:

```bash
aws mgn update-replication-configuration-template \
  --cli-input-json '{
    "replicationConfigurationTemplateID": "<TEMPLATE_ID>",
    "stagingAreaSubnetId": "<PUBLIC_SUBNET_ID>",
    "replicationServersSecurityGroupsIDs": ["<MGN_REPLICATION_SG_ID>"]
  }' \
  --profile myorg --region ap-south-1
```

> **Note:** There is only ONE MGN replication template per AWS account. When migrating VMs across different VPCs, update this template each time to point to the correct VPC's public subnet.

---

## Stage 6 — Configure MGN Default Launch Template

Before installing agents, configure the default launch template in MGN to prevent instances launching in the wrong (default) VPC:

```
MGN Console → Settings → Launch template → Edit
→ VPC: <TARGET_VPC_ID>
→ Subnet: <TARGET_SUBNET_ID>
→ Security group: <TARGET_EC2_SG_ID>
→ Save
```

> **Critical:** If you skip this step, MGN will launch test instances in the default VPC instead of your target VPC.

> 🏠 **Skipping this step is like forgetting to give your movers the new apartment address.** They'll drop all your furniture at some random default location — and then you'll wonder why your couch ended up in a stranger's house. Always configure this *before* the agent goes in.

---

## Stage 7 — Install MGN Agent on GCP VM

### Add Firewall Tag

```bash
gcloud compute instances add-tags <VM_NAME> \
  --tags=myapp-db-stage-mgn \
  --zone=asia-south1-b \
  --project=my-gcp-project
```

> 🏷️ **What's a firewall tag?** It's like putting a sticker on the moving box that says "this box is allowed to go through the loading dock." Without the tag, GCP's firewall won't allow outbound traffic from the VM to the MGN replication server. The tag activates the firewall rule you created in Stage 2.

### SSH into GCP VM

For VMs with public IP:
```bash
gcloud compute ssh <VM_NAME> \
  --zone=asia-south1-b \
  --project=my-gcp-project
```

For VMs with private IP (via bastion):
```bash
ssh -i <key.pem> ubuntu@203.0.113.10
ssh -i <key.pem> ubuntu@10.0.2.55
```

### Install Agent

```bash
# Check kernel version first
uname -r

# Download installer (run as single line)
sudo wget -O ./aws-replication-installer-init.py https://aws-application-migration-service-ap-south-1.s3.ap-south-1.amazonaws.com/latest/linux/aws-replication-installer-init.py

# Run installer
sudo python3 aws-replication-installer-init.py \
  --region ap-south-1 \
  --aws-access-key-id <ACCESS_KEY> \
  --aws-secret-access-key <SECRET_KEY>
```

Press **Enter** when prompted to select disks (replicates all disks).

> 🔧 **The agent is essentially a background worker** that wakes up continuously, looks at what changed on your GCP disk since the last sync, and quietly sends those changes over to AWS — like a diligent assistant who keeps your backup up to date without you ever asking. You install it once, and it runs silently in the background.

### Verify Installation

```bash
# Check log if installation fails
cat ~/aws_replication_agent_installer.log | tail -50
```

---

## Stage 8 — Monitor Replication

Once the agent installs, the VM appears in MGN console as a source server.

```
MGN Console → Source servers
```

| Status | Meaning | Action |
|--------|---------|--------|
| Not ready / Initiating | Agent installed, initial sync starting | Wait |
| Initial sync X% \| Xhr left | Full disk copy in progress | Wait |
| Ready for testing | Sync complete | Launch test instance |
| Stalled / Lagging | Agent lost connection or GCP VM stopped | Restart GCP VM |
| Test in progress | Test EC2 running | Verify app |
| Ready for cutover | Testing verified | Launch cutover |

> **Note:** MGN replicates the full allocated disk size, not just used space. A 400 GiB disk with 35 GiB used still takes ~20 hours to sync at ~20 GB/hr.

> ⏳ **Why does it replicate the full disk and not just the used space?** Imagine you're scanning an entire filing cabinet for sensitive documents — even though most drawers are empty, the scanner still has to open every drawer to confirm. MGN copies the entire allocated disk to guarantee nothing is missed. A 400 GiB disk at 35 GiB used = ~20 hours at 20 GB/hr. Grab a coffee (or two).

---

## Stage 9 — Launch Test Instance

Once status shows **Ready for testing**:

```
MGN Console → select server
→ Test and cutover → Launch test instances → Confirm
```

MGN automatically creates a **conversion server** (m5.large) that converts the GCP disk format to AWS EBS, then launches the target EC2.

> 🧪 **This is your test drive before buying the car.** The test EC2 is a fully functional copy of your GCP VM running in AWS. Your production GCP VM is still live and serving traffic. You're just kicking the tires on the AWS side — checking if the database is healthy, services are running, and nothing blew up during the disk format conversion.

### Verify the Test Instance

```bash
# Get private IP
aws ec2 describe-instances \
  --instance-ids <INSTANCE_ID> \
  --query "Reservations[0].Instances[0].PrivateIpAddress" \
  --output text --profile myorg --region ap-south-1

# Connect via SSM (if IAM role attached)
aws ssm start-session \
  --target <INSTANCE_ID> \
  --profile myorg --region ap-south-1

# Check MongoDB
sudo systemctl status mongod
mongosh --eval "db.adminCommand('listDatabases')"
```

### After Verification

```
MGN Console → server → Test and cutover → Mark as Ready for cutover
```

---

## Stage 10 — Cutover (Final Migration)

Cutover makes the AWS EC2 the live server. GCP VM is **never automatically stopped** — you control decommissioning.

```
MGN Console → select server
→ Test and cutover → Launch cutover instances → Confirm
```

MGN performs a final sync, converts the disk, and launches the production EC2.

> 🔑 **This is the moment you hand over the keys.** MGN does one last sync to catch any changes made since testing, converts the disk, and launches the production EC2. Your GCP VM keeps running — you decide when to shut it down. Think of it as keeping the old apartment for a week while you settle into the new one, just in case you left something behind.

After cutover:
1. Update DNS / app configs to point to new AWS private IP
2. Monitor for 1-2 days
3. Mark cutover complete in MGN console
4. Manually stop GCP VM (keep for 1 week as fallback)
5. Delete GCP VM after confirmed stable

---

## Adding More VMs to the Same VPC

For additional VMs in the **same VPC** — no new Terraform needed. Just add a new SG file and install the agent:

```hcl
# aws-infra/new-vm.tf
resource "aws_security_group" "target_ec2_new_vm" {
  name   = "new-vm-ec2-sg"
  vpc_id = "vpc-0abc1234def56789a"
  # ... ingress rules for that VM's ports
}
```

Then install agent on the GCP VM using same credentials.

> ♻️ **Once the infrastructure is in place, adding more VMs is like adding more rooms to a house that's already built.** You don't re-lay the foundation — you just add a new door (security group) and let the movers (agent) in.

---

## Migrating VMs Across Different VPCs

Since there is only one MGN replication template per account, switch it when moving to a different VPC:

```bash
aws mgn update-replication-configuration-template \
  --cli-input-json '{
    "replicationConfigurationTemplateID": "<TEMPLATE_ID>",
    "stagingAreaSubnetId": "<NEW_VPC_PUBLIC_SUBNET_ID>",
    "replicationServersSecurityGroupsIDs": ["<NEW_VPC_MGN_SG_ID>"]
  }' \
  --profile myorg --region ap-south-1
```

> **Warning:** Switching the template while a replication is active will stall existing replications. Finish one VPC fully before switching to the next.

> ⚠️ **Think of the MGN template as a single-track railway line.** Only one train (VPC) can use the track at a time. If you switch the points while a train is mid-journey, it derails. Always let one migration complete before redirecting the template to a new VPC.

---

## Kernel Compatibility Issue

MGN agent fails on GCP kernels above **6.8.0-1032-gcp** due to a missing kernel function `rq_list_move` that was removed in newer Linux kernels.

| Kernel | Status |
|--------|--------|
| ≤ 6.8.0-1030-gcp | Works ✅ |
| 6.8.0-1045, 1048, 1052-gcp | COMPILE_DRIVER_ERROR ❌ |
| 6.17.0-1008-gcp | Completely unsupported ❌ |

For affected VMs use the **Disk Snapshot** migration method instead (see blog1).

> 🔩 **Why does the kernel version matter?** The MGN agent compiles a small kernel module to intercept disk writes. When a newer kernel removes or renames an internal function (`rq_list_move`), the agent's module can't compile — like trying to plug a 3-pin plug into a 2-pin socket. It simply won't fit.
>
> If your GCP VM is running a newer kernel and you hit this wall, don't panic — we've been there. The **Disk Snapshot method** is your reliable fallback:
> - 📖 **[GCP to AWS VM Migration: Disk Snapshot Guide](https://vishnunukala.hashnode.dev/gcp-to-aws-vm-migration-disk-snapshot-guide)** — the manual, step-by-step approach that bypasses the agent entirely.
> - 🛠️ **[Fix GCP to AWS Kernel Boot Errors](https://vishnunukala.hashnode.dev/fix-gcp-to-aws-kernel-boot-errors)** — if you've already tried the snapshot method and your EC2 won't boot, this guide walks you through diagnosing and fixing kernel-related boot failures on AWS.

---

## Common Errors and Fixes

### AccessDeniedException on agent install
```
Fix: Attach AWSApplicationMigrationFullAccess to the MGN agent IAM user
aws iam attach-user-policy \
  --user-name myapp-db-stage-mgn-agent \
  --policy-arn arn:aws:iam::aws:policy/AWSApplicationMigrationFullAccess \
  --profile myorg
```

### Test instance launched in wrong VPC
```
Fix: Create AMI from wrong instance → relaunch in correct subnet
aws ec2 create-image --instance-id <ID> --name "vm-correct-subnet" --no-reboot --profile myorg
aws ec2 run-instances --image-id <AMI_ID> --subnet-id <CORRECT_SUBNET> --associate-public-ip-address --profile myorg
aws ec2 terminate-instances --instance-ids <OLD_ID> --profile myorg
```

### Replication stalled
```
Fix: Start GCP VM + check MGN template points to correct VPC subnet
MGN Console → select server → Replication → Resume replication
```

### SSM not connecting (TargetNotConnected)
```
Fix: Attach IAM role to instance, then install SSM agent via serial console
sudo snap install amazon-ssm-agent --classic
sudo systemctl enable --now amazon-ssm-agent
```

> 🔍 **Pro tip on SSM:** SSM Session Manager is like remote desktop without opening any firewall ports. But it needs the instance to have an IAM role with SSM permissions AND the SSM agent running. If either is missing, you get `TargetNotConnected` — the digital equivalent of knocking on a door with no one inside to answer.

---

## Summary

| Step | What Happens |
|------|-------------|
| Stage 1 | Configure AWS + GCP CLI with company accounts |
| Stage 2 | Terraform provisions IAM, SGs, public subnet |
| Stage 3 | Find free CIDR block in target VPC |
| Stage 4 | Deploy Terraform — GCP infra first, then AWS |
| Stage 5 | Initialize MGN in AWS account |
| Stage 6 | Configure MGN default launch template |
| Stage 7 | Install replication agent on each GCP VM |
| Stage 8 | Monitor initial sync until Ready for testing |
| Stage 9 | Launch test EC2 and verify application |
| Stage 10 | Cutover — launch production EC2, decommission GCP VM |

---

> 🗺️ **The full migration trilogy:**
> 1. 📦 **[GCP to AWS VM Migration: Disk Snapshot Guide](https://vishnunukala.hashnode.dev/gcp-to-aws-vm-migration-disk-snapshot-guide)** — Manual snapshot-based migration, great for simple VMs or when the MGN agent won't cooperate.
> 2. 🛠️ **[Fix GCP to AWS Kernel Boot Errors](https://vishnunukala.hashnode.dev/fix-gcp-to-aws-kernel-boot-errors)** — Your lifeline when the migrated VM refuses to boot on AWS.
> 3. 🚀 **This article** — MGN-based live migration with zero downtime and a safety net.