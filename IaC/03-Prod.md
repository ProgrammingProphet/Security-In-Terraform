# Production-Grade Setup

## Vault + AWS STS + Terraform + CI/CD (Recommended Architecture)

This setup uses:

* HashiCorp [Vault](https://developer.hashicorp.com/vault?utm_source=chatgpt.com)
* Amazon Web Services IAM Roles + STS
* Dynamic temporary credentials
* Terraform
* CI/CD pipelines securely

NO long-lived AWS keys inside:

* GitHub Secrets
* Jenkins
* GitLab
* Terraform code

---

# Final Architecture

```text id="8m4b1z"
CI/CD
  |
  v
Authenticate to Vault
  |
  v
Vault AWS Secrets Engine
  |
  v
AWS STS AssumeRole
  |
  v
Temporary Credentials
  |
  v
Terraform Apply
```

---

# STEP 1 — Create AWS IAM Role

Go to:

```text id="x1xg9g"
AWS Console → IAM → Roles → Create Role
```

Select:

```text id="vqpyh3"
AWS Account
```

---

## Create Trust Policy

Create role:

```text id="lcfht3"
vault-terraform-role
```

Trust policy:

```json id="7d06yo"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Replace:

```text id="89fdv8"
<ACCOUNT_ID>
```

with your AWS account ID.

---

# STEP 2 — Attach Terraform Permissions

Attach policy to the role.

Example Admin-like policy for learning:

```json id="t5ppw8"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

Production should use least privilege.

---

# STEP 3 — Copy Role ARN

Example:

```text id="8y5n7v"
arn:aws:iam::123456789012:role/vault-terraform-role
```

Save this.

---

# STEP 4 — Install Vault

On Ubuntu:

```bash id="axruxc"
sudo apt update
sudo apt install gpg -y
```

Add HashiCorp repo:

```bash id="v3j54j"
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

```bash id="xj4w2s"
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
```

Install Vault:

```bash id="hfhj7m"
sudo apt update
sudo apt install vault -y
```

Check:

```bash id="f0v8ms"
vault version
```

---

# STEP 5 — Start Vault Dev Server (Learning)

```bash id="a3f6jq"
vault server -dev -dev-root-token-id=root
```

Export variables:

```bash id="bsjlwm"
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'
```

Verify:

```bash id="0xjkz5"
vault status
```

---

# STEP 6 — Enable AWS Secrets Engine

```bash id="sn7e8y"
vault secrets enable aws
```

Verify:

```bash id="7nqu8n"
vault secrets list
```

You should see:

```text id="7j56j8"
aws/    aws
```

---

# STEP 7 — Configure Vault AWS Root Access

Vault needs permissions to call AWS STS.

Use AWS access keys temporarily:

```bash id="b2x5cb"
vault write aws/config/root \
    access_key=YOUR_ACCESS_KEY \
    secret_key=YOUR_SECRET_KEY \
    region=ap-south-1
```

Verify:

```bash id="mo3pl9"
vault read aws/config/root
```

---

# STEP 8 — Create Vault AWS Role

THIS is the important production-grade part.

Run:

```bash id="bz09g4"
vault write aws/roles/terraform-role \
    credential_type=assumed_role \
    role_arns=arn:aws:iam::123456789012:role/vault-terraform-role \
    default_sts_ttl=1h \
    max_sts_ttl=2h
```

Explanation:

| Parameter                    | Meaning                      |
| ---------------------------- | ---------------------------- |
| credential_type=assumed_role | Generate temporary STS creds |
| role_arns                    | AWS role to assume           |
| default_sts_ttl              | Default expiry               |
| max_sts_ttl                  | Max expiry                   |

---

# STEP 9 — Generate Temporary Credentials

Now test:

```bash id="r4gslm"
vault read aws/sts/terraform-role
```

You will get:

```text id="c7bjlwm"
access_key
secret_key
security_token
lease_duration
```

These credentials are temporary.

---

# STEP 10 — Export Credentials

```bash id="e0md4r"
export AWS_ACCESS_KEY_ID=<ACCESS_KEY>
export AWS_SECRET_ACCESS_KEY=<SECRET_KEY>
export AWS_SESSION_TOKEN=<TOKEN>
```

Verify:

```bash id="xv1goh"
aws sts get-caller-identity
```

---

# STEP 11 — Terraform Configuration

Install Terraform:

```bash id="vffk2m"
sudo apt install unzip -y
```

```bash id="90eozv"
wget https://releases.hashicorp.com/terraform/1.8.5/terraform_1.8.5_linux_amd64.zip
```

```bash id="9vjlwm"
unzip terraform_1.8.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

Verify:

```bash id="a3ffgm"
terraform version
```

---

# STEP 12 — Terraform Example

Create:

```text id="r6n49o"
main.tf
```

Example:

```hcl id="vq4l63"
provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "demo" {
  bucket = "vikram-demo-bucket-123456"
}
```

Run:

```bash id="i4gj3k"
terraform init
terraform apply
```

Terraform automatically uses:

* AWS_ACCESS_KEY_ID
* AWS_SECRET_ACCESS_KEY
* AWS_SESSION_TOKEN

---

# STEP 13 — CI/CD Integration (GitHub Actions)

Use:

* GitHub OIDC
* Vault JWT auth
* Dynamic creds

---

## Enable JWT Auth

```bash id="6r6e7p"
vault auth enable jwt
```

---

## Configure GitHub OIDC

```bash id="f6dqme"
vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"
```

---

## Create Vault Policy

Create:

```text id="lgw1p4"
terraform-policy.hcl
```

Content:

```hcl id="v0cgj5"
path "aws/sts/terraform-role" {
  capabilities = ["read"]
}
```

Apply:

```bash id="vjlwm1"
vault policy write terraform-policy terraform-policy.hcl
```

---

## Create JWT Role

```bash id="4ddw9v"
vault write auth/jwt/role/github-actions \
    role_type="jwt" \
    bound_audiences="https://github.com/<ORG>" \
    user_claim="sub" \
    policies="terraform-policy" \
    ttl="1h"
```

---

# STEP 14 — GitHub Actions Workflow

```yaml id="txhjlwm"
name: Terraform

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Import Secrets
        uses: hashicorp/vault-action@v2
        with:
          url: https://vault.example.com
          method: jwt
          role: github-actions
          secrets: |
            aws/sts/terraform-role access_key | AWS_ACCESS_KEY_ID ;
            aws/sts/terraform-role secret_key | AWS_SECRET_ACCESS_KEY ;
            aws/sts/terraform-role security_token | AWS_SESSION_TOKEN

      - uses: hashicorp/setup-terraform@v3

      - run: terraform init

      - run: terraform apply -auto-approve
```

---

# Security Benefits

| Traditional        | Vault STS            |
| ------------------ | -------------------- |
| Static IAM users   | Temporary creds      |
| Secrets in GitHub  | No long-term secrets |
| Manual rotation    | Auto-expiry          |
| High blast radius  | Minimal risk         |
| Difficult auditing | Better traceability  |

---

# Production Improvements

After learning phase:

| Improvement             | Purpose            |
| ----------------------- | ------------------ |
| Vault HA                | High availability  |
| Integrated Storage/Raft | Production backend |
| Auto Unseal with KMS    | Secure unseal      |
| TLS certificates        | Encryption         |
| Vault Agent             | Auto auth          |
| Kubernetes Auth         | Pod identity       |
| AWS IAM Auth            | EC2 auth           |
| AppRole Auth            | Machine auth       |

---

# Enterprise-Level Pattern

Large companies typically use:

```text id="yjlwmv"
GitHub Actions
   ↓
Vault JWT Auth
   ↓
Vault AWS Secrets Engine
   ↓
AWS STS AssumeRole
   ↓
Terraform
```

This is one of the core DevSecOps industry patterns.
