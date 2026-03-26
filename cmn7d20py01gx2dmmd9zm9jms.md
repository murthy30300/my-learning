---
title: "GCP to AWS Migration Using Disk Snapshot (Step-by-Step)"
seoTitle: "GCP to AWS VM Migration Using Disk Snapshot Guide"
seoDescription: "Step-by-step guide to migrate a GCP VM to AWS using disk snapshots, VMDK export, RAW conversion, and AMI import."
datePublished: 2026-03-26T10:59:56.330Z
cuid: cmn7d20py01gx2dmmd9zm9jms
slug: gcp-to-aws-vm-migration-disk-snapshot-guide
cover: https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/91f93484-b7bf-4699-9107-a5ec4ef51ad4.png
ogImage: https://cdn.hashnode.com/uploads/og-images/66f590c9120870cbbe3a72f4/4585ae2b-f630-423f-a8cc-d9cb1078f8a7.png
tags: ec2, linux, aws, google-cloud, devops, infrastructure, sre, gcp, vmmigration, cloud-migration

---

> Migrating infrastructure between cloud providers sounds daunting, but with the right process it's repeatable and reliable. This guide walks you through the **Disk Snapshot** approach to move a GCP Compute Engine VM to an AWS EC2 instance — every command, every detail.

* * *

## Why Disk Snapshot Migration?

When you need to move a running server from GCP to AWS, you have a few options. The **Disk Snapshot** method is the most reliable when:

*   You want a byte-for-byte copy of the disk (OS + data + configs intact)
    
*   You can't or don't want to reinstall and reconfigure the application from scratch
    
*   You need a one-time lift-and-shift migration
    

The general flow is: **GCP Disk → Snapshot → Image → VMDK → S3 → RAW → AMI → EC2**

* * *

## Prerequisites

*   A running GCP VM (in this guide: `analytics-server-new`)
    
*   A GCP bucket for staging the export
    
*   An AWS S3 bucket for staging the import
    
*   AWS credentials with EC2 and IAM permissions
    
*   AWS CLI installed on both your GCP VM and local machine
    

* * *

## Step 1 — Add Your SSH Public Key to the GCP VM

Before you start, make sure your SSH key pair's **public key** is added to the VM's `~/.ssh/authorized_keys`. This ensures you can SSH into any restored instance later.

```bash
# On the GCP VM
echo "YOUR_PUBLIC_KEY" >> ~/.ssh/authorized_keys
```

> **Why?** When you restore the disk image as an EC2 instance, AWS won't inject keys unless you use EC2 Instance Connect or you've pre-baked the key into the disk.

* * *

## Step 2 — Create a GCP Bucket for Staging

You need a GCP Cloud Storage bucket to temporarily hold the exported disk image. Create one in the **same region** as your VM to avoid egress charges.

```bash
# Via GCP Console or CLI
gsutil mb -l YOUR_REGION gs://YOUR_BUCKET
```

* * *

## Step 3 — Snapshot the GCP Disk

A snapshot captures the exact state of your disk at a point in time. This is non-destructive and your VM keeps running.

```bash
gcloud compute disks snapshot analytics-server-new \
  --snapshot-names=analytics-server-snapshot \
  --zone=YOUR_ZONE
```

> **Theory:** GCP snapshots are incremental and stored in Google's infrastructure. They are not yet a portable image — we need to convert this snapshot into an exportable format.

* * *

## Step 4 — Create a GCP Image from the Snapshot

Convert the snapshot into a GCP Compute Image. This is the intermediate format needed before exporting.

```bash
gcloud compute images create analytics-server-image \
  --source-snapshot=analytics-server-snapshot
```

* * *

## Step 5 — Export the Image to GCS as VMDK

Export the image to your GCS bucket in **VMDK format**. VMDK (Virtual Machine Disk) is VMware's disk format and is one of the formats supported by AWS VM Import/Export.

```bash
gcloud compute images export \
  --image=analytics-server-image \
  --destination-uri=gs://YOUR_BUCKET/analytics-server.vmdk \
  --export-format=vmdk \
  --zone=YOUR_ZONE
```

> **Note:** This export can take 15–45 minutes depending on disk size. The `--zone` flag is required even for image exports.

* * *

## Step 6 — Verify the Exported File in GCS

Confirm the VMDK file landed in your bucket and check its size.

```bash
gsutil ls -lh gs://YOUR_BUCKET/analytics-server.vmdk
```

You should see the file size listed. A 100 GB disk typically produces a VMDK around 40–50 GB (compressed).

* * *

## Step 7 — Install AWS CLI on the GCP VM

You'll use the GCP VM itself to transfer the file to AWS S3 — this avoids expensive egress fees by routing the data from GCP's network side.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

Configure it with your AWS credentials:

```bash
aws configure
# Enter: AWS Access Key ID, Secret Access Key, Region, Output format
```

* * *

## Step 8 — Set Up the AWS VM Import IAM Role

AWS VM Import/Export requires a specific IAM role called `vmimport`. This allows the EC2 service to read from your S3 bucket and register AMIs on your behalf.

**Create the trust policy:**

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "vmie.amazonaws.com" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "vmimport" }
    }
  }]
}
EOF
```

**Create the role:**

```bash
aws iam create-role \
  --role-name vmimport \
  --assume-role-policy-document file://trust-policy.json
```

> **Why?** Without this role, `aws ec2 import-image` will fail with an authorization error. The `vmie.amazonaws.com` service principal is the VM Import/Export service that does the actual AMI creation.

You also need to attach an inline policy that grants the `vmimport` role access to your S3 bucket. Create `role-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetBucketLocation", "s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::YOUR_S3_BUCKET", "arn:aws:s3:::YOUR_S3_BUCKET/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["ec2:ModifySnapshotAttribute", "ec2:CopySnapshot", "ec2:RegisterImage", "ec2:Describe*"],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam put-role-policy \
  --role-name vmimport \
  --policy-name vmimport \
  --policy-document file://role-policy.json
```

* * *

## Step 9 — Transfer VMDK from GCS to S3

The most cost-efficient way is to stream directly from GCS to S3 from within the GCP VM in the **same region as your bucket** — GCP doesn't charge egress within the same region.

```bash
gsutil cp gs://YOUR_BUCKET/analytics-server.vmdk - | \
  aws s3 cp - s3://YOUR_S3_BUCKET/migrations/analytics-server.vmdk \
  --expected-size 47244640256
```

> **Tip:** The `--expected-size` flag (in bytes) is required when piping stdin to S3 because S3 needs to know the content length upfront for multipart uploads. Get the exact size from `gsutil ls -lh`.

**Verify the upload:**

```bash
aws s3 ls s3://YOUR_S3_BUCKET/migrations/analytics-server.vmdk --human-readable
```

* * *

## Step 10 — Launch a Temporary EC2 for VMDK → RAW Conversion

> **Why convert?** GCP exports images in a **streamOptimized VMDK** format, which is incompatible with AWS VM Import. AWS requires either a **RAW**, OVA, or classic VMDK. We'll convert to RAW using `qemu-img`.

Launch a temporary EC2 instance with:

*   **Size:** At least **3× the disk size** (e.g., if your disk is 100 GB, use a 300 GB EBS volume). The RAW file will be the full virtual disk size.
    
*   **Instance type:** `t3.medium` or better — the conversion is CPU-bound.
    
*   **OS:** Ubuntu 22.04 LTS
    

SSH into it:

```bash
ssh -i YOUR_KEY.pem ubuntu@YOUR_EC2_IP
```

Install dependencies:

```bash
sudo apt-get update -y
sudo apt-get install -y qemu-utils awscli
```

* * *

## Step 11 — Download the VMDK from S3

```bash
aws s3 cp s3://YOUR_S3_BUCKET/migrations/analytics-server.vmdk . \
  --no-progress
```

* * *

## Step 12 — Inspect and Convert VMDK to RAW

**Check the format first:**

```bash
qemu-img info analytics-server.vmdk
```

**Convert to RAW** (~10–20 minutes for a 100 GB disk):

```bash
qemu-img convert -p \
  -f vmdk \
  -O raw \
  analytics-server.vmdk \
  analytics-server.raw
```

> **Theory:** RAW format is an uncompressed sector-by-sector copy of the disk. While larger in size, it is the most universally compatible format for VM Import and avoids any format-parsing issues.

**Verify the output:**

```bash
ls -lh analytics-server.raw
qemu-img info analytics-server.raw
```

The output file size should match the original virtual disk size (e.g., ~100 GB).

* * *

## Step 13 — Upload the RAW File to S3

```bash
aws s3 cp analytics-server.raw \
  s3://YOUR_S3_BUCKET/migrations/analytics-server.raw \
  --expected-size ACTUAL_SIZE_IN_BYTES \
  --no-progress
```

Replace `ACTUAL_SIZE_IN_BYTES` with the byte size shown by `ls -lh` or `stat analytics-server.raw`.

* * *

## Step 14 — Import the RAW Image as an AMI

Run this from your local machine or the EC2 converter (using the appropriate AWS profile):

```bash
aws ec2 import-image \
  --description "analytics-server-new migration" \
  --disk-containers '[
    {
      "Description": "analytics-server-new",
      "Format": "RAW",
      "UserBucket": {
        "S3Bucket": "YOUR_S3_BUCKET",
        "S3Key": "migrations/analytics-server.raw"
      }
    }
  ]' \
  --profile=company
```

* * *

## Step 15 — Track the Import Progress

The import process runs asynchronously. Poll it with:

```bash
aws ec2 describe-import-image-tasks \
  --import-task-ids import-ami-XXXXXXXX
```

You'll see the `Status` field cycle through:

*   `active` → `converting` → `updating` → `booting` → `completed`
    

This typically takes **20–60 minutes** depending on disk size.

* * *

## Step 16 — Handle Kernel Errors (If Any)

If your import completes but the instance **fails to boot** or you see console errors referencing the kernel, it's because GCP uses a custom GCP-optimized kernel that isn't compatible with the AWS Nitro/Xen hypervisor.

> 📖 **If you encounter kernel boot errors, follow this guide to fix it:**  
> **[How to Replace a GCP Kernel with a Generic Kernel for AWS Migration →](https://vishnunukala.hashnode.dev/fix-gcp-to-aws-kernel-boot-errors)**

* * *

## Step 17 — Upload the Fixed RAW and Re-Import

After fixing the kernel (see blog above), upload the corrected RAW file:

```bash
ls -lh ~/analytics-server.raw

aws s3 cp ~/analytics-server.raw \
  s3://YOUR_S3_BUCKET/migrations/analytics-server-fixed.raw \
  --no-progress
```

Re-run the import with the fixed image:

```bash
aws ec2 import-image \
  --description "analytics-server-new migration v2" \
  --disk-containers '[
    {
      "Description": "analytics-server-fixed",
      "Format": "RAW",
      "UserBucket": {
        "S3Bucket": "YOUR_S3_BUCKET",
        "S3Key": "migrations/analytics-server-fixed.raw"
      }
    }
  ]' \
  --profile=company
```

* * *

## Step 18 — Launch Your EC2 Instance from the AMI

Once `describe-import-image-tasks` returns `"Status": "completed"`, you'll get an AMI ID like `ami-0abc1234def56789`.

Go to **EC2 → AMIs → Launch Instance** and select your new AMI — or use the CLI:

```bash
aws ec2 run-instances \
  --image-id ami-XXXXXXXXXXXXXXXXX \
  --instance-type t3.medium \
  --key-name YOUR_KEY_PAIR \
  --subnet-id subnet-XXXXXXXX \
  --security-group-ids sg-XXXXXXXX
```

Your server is now running on AWS with all its data, configurations, and applications intact. 🎉

* * *

## Summary

| Step | What Happens |
| --- | --- |
| Snapshot GCP disk | Point-in-time copy of your VM disk |
| Create GCP image | Snapshot → portable GCP image |
| Export as VMDK | GCP image → VMDK in GCS bucket |
| Transfer to S3 | GCS → AWS S3 via GCP VM (free egress) |
| Convert VMDK → RAW | EC2 converter + qemu-img |
| Import as AMI | AWS VM Import registers the RAW as an AMI |
| Launch EC2 | Boot your migrated server on AWS |

* * *

## Common Pitfalls

*   **Wrong VMDK format:** Always convert to RAW before importing — streamOptimized VMDK from GCP will fail.
    
*   `vmimport` **role missing:** The import will silently fail or return an auth error. Double-check the role and policy.
    
*   **Disk too small on converter:** The RAW file equals the virtual disk size. Always provision 3× the original disk.
    
*   **Kernel mismatch:** GCP kernels don't boot on AWS. See the \[kernel fix guide → LINK\_TO\_BLOG\_2\].