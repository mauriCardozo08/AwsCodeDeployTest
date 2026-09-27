# AwsCodeDeployTest

Continuous deployment of a static website from GitHub to an **on-premises Linux server** (nginx) using **GitHub Actions**, **Amazon S3** and **AWS CodeDeploy**.

Every push to `main` that changes `site/` or `appspec.yml` packages the site, uploads it to S3 and triggers a CodeDeploy deployment. The CodeDeploy agent running on the on-premises server picks up the deployment, downloads the artifact from S3 and copies the files into the nginx web root.

---

## Table of contents

1. [Architecture](#architecture)
2. [Repository structure](#repository-structure)
3. [Resources used](#resources-used)
4. [Part 1 – AWS base resources](#part-1--aws-base-resources)
5. [Part 2 – On-premises server setup](#part-2--on-premises-server-setup)
6. [Part 3 – CodeDeploy application and deployment group](#part-3--codedeploy-application-and-deployment-group)
7. [Part 4 – GitHub Actions with OIDC](#part-4--github-actions-with-oidc)
8. [How a deployment works end to end](#how-a-deployment-works-end-to-end)
9. [Troubleshooting](#troubleshooting)

---

## Architecture

```
 ┌──────────────┐   push to main   ┌──────────────────┐
 │  Developer   │ ───────────────▶ │  GitHub Actions  │
 └──────────────┘                  └───┬──────────┬───┘
                                       │          │ 1. OIDC token → STS AssumeRoleWithWebIdentity
                 3. create-deployment  │          │    (temporary credentials, no stored secrets)
          ┌────────────────────────────┘          │ 2. zip + upload artifact
          ▼                                       ▼
 ┌──────────────────┐                    ┌──────────────────┐
 │  AWS CodeDeploy  │                    │    Amazon S3     │
 └────────▲─────────┘                    └────────▲─────────┘
          │ 4. agent polls for commands           │ 5. agent downloads artifact
          │    (outbound HTTPS only)              │
 ┌────────┴───────────────────────────────────────┴──┐
 │  On-premises server (Ubuntu + nginx)              │
 │  CodeDeploy agent → copies site/ to web root      │
 └───────────────────────────────────────────────────┘
```

CodeDeploy never pushes anything to the server. The agent on the server **polls** CodeDeploy for pending work, which means the server only needs outbound HTTPS access to AWS.

### CodeDeploy concepts

| Concept | What it is | In this project |
|---|---|---|
| **Application** | A named container for what you deploy | `code-deploy-test-app` |
| **Deployment group** | *Where* and *how* to deploy: target servers (selected by tag), service role, deployment strategy | `onprem-dg` |
| **Revision** | *Which version*: a zip in S3 containing the files and `appspec.yml` | `s3://<bucket>/code-deploy-test-app/<commit-sha>.zip` |
| **Deployment** | One execution: application + deployment group + revision | Created by the workflow on every push that changes `site/` or `appspec.yml` |

Instances are never referenced directly in a deployment. The deployment group selects them by tag, so adding a new server only requires registering it with the same tag.

---

## Repository structure

```
.
├── appspec.yml                 # CodeDeploy instructions (must be at the zip root)
├── site/                       # Files published to the nginx web root
│   └── index.html
└── .github/
    └── workflows/
        └── deploy.yml          # CI/CD pipeline
```

Only the contents of `site/` are copied to the server, so `appspec.yml` and the workflow are never exposed on the website.

---

## Resources used

| Resource | Name / value |
|---|---|
| AWS region | `us-east-1` |
| S3 bucket (private) | `code-deploy-test-mc-975630231455` |
| CodeDeploy service role | `CodeDeployTest` (policy `AWSCodeDeployRole`) |
| IAM user for the on-prem agent | `codedeploy-onprem-mcardozo` |
| On-prem instance name | `onprem-mcardozo` |
| On-prem instance tag | `Name=onprem-mcardozo` |
| CodeDeploy application | `code-deploy-test-app` |
| Deployment group | `onprem-dg` |
| IAM role for GitHub Actions | `github-actions-codedeploy` |
| nginx web root | `/var/www/html` |

> Replace these values if you reproduce this setup in another account. The AWS account ID and role ARNs are identifiers, not secrets, but **never commit access keys**.

---

## Part 1 – AWS base resources

### 1.1 S3 bucket

A private bucket that stores deployment artifacts. It must be in the **same region** as the CodeDeploy application.

### 1.2 CodeDeploy service role

Role `CodeDeployTest` with the AWS managed policy `AWSCodeDeployRole`. This is a *service role*: it is assumed by the CodeDeploy service itself, not by an instance, so it has no instance profile.

Verify its trust policy allows CodeDeploy:

```bash
aws iam get-role --role-name CodeDeployTest \
  --query 'Role.AssumeRolePolicyDocument.Statement[].Principal'
# Expected: "Service": "codedeploy.amazonaws.com"
```

### 1.3 IAM user for the on-premises agent

On-premises servers cannot use instance roles, so the agent authenticates with an IAM user. User `codedeploy-onprem-mcardozo` has this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::code-deploy-test-mc-975630231455"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::code-deploy-test-mc-975630231455/*"
    },
    {
      "Effect": "Allow",
      "Action": ["codedeploy:*"],
      "Resource": "*"
    }
  ]
}
```

Create an access key for it (IAM → Users → Security credentials → Create access key). It will be stored only on the on-premises server.

---

## Part 2 – On-premises server setup

Tested on Ubuntu 26.04 (running under WSL).

### 2.1 Agent configuration file

Create it **before** installing the agent:

```bash
sudo mkdir -p /etc/codedeploy-agent/conf
sudo nano /etc/codedeploy-agent/conf/codedeploy.onpremises.yml
```

```yaml
---
aws_access_key_id: AKIA...
aws_secret_access_key: <secret>
iam_user_arn: arn:aws:iam::975630231455:user/codedeploy-onprem-mcardozo
region: us-east-1
```

The `iam_user_arn` must match the IAM user **exactly** (see [Troubleshooting](#agent-log-shows-internalfailure-on-poll_host_command)).

### 2.2 Install Ruby

The CodeDeploy agent runs on Ruby, so install it before the agent:

```bash
sudo apt update
sudo apt install -y ruby-full wget
```

### 2.3 Install the agent

```bash
cd /tmp
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x install
sudo AWS_REGION=us-east-1 ./install auto
sudo systemctl status codedeploy-agent
```

`install` is AWS's installer script. `auto` detects the package type (`.deb`/`.rpm`). Setting `AWS_REGION` avoids a long wait while the script tries to reach the EC2 metadata service, which does not exist on-premises.

### 2.4 Register the server in CodeDeploy

Run with administrator credentials (e.g. from AWS CloudShell):

```bash
aws deploy register-on-premises-instance \
  --instance-name onprem-mcardozo \
  --iam-user-arn arn:aws:iam::975630231455:user/codedeploy-onprem-mcardozo \
  --region us-east-1

aws deploy add-tags-to-on-premises-instances \
  --instance-names onprem-mcardozo \
  --tags Key=Name,Value=onprem-mcardozo \
  --region us-east-1

aws deploy get-on-premises-instance --instance-name onprem-mcardozo --region us-east-1
```

### 2.5 Verify the agent is connected

Enable verbose logging temporarily in `/etc/codedeploy-agent/conf/codedeployagent.yml` (`:verbose: true`), then:

```bash
sudo systemctl restart codedeploy-agent
sudo tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

Healthy output shows periodic `poll_host_command` calls with HTTP **200** and no `ERROR` lines. Set `:verbose: false` afterwards.

---

## Part 3 – CodeDeploy application and deployment group

```bash
aws deploy create-application \
  --application-name code-deploy-test-app \
  --compute-platform Server \
  --region us-east-1

aws deploy create-deployment-group \
  --application-name code-deploy-test-app \
  --deployment-group-name onprem-dg \
  --service-role-arn arn:aws:iam::975630231455:role/CodeDeployTest \
  --on-premises-instance-tag-filters Key=Name,Value=onprem-mcardozo,Type=KEY_AND_VALUE \
  --deployment-config-name CodeDeployDefault.AllAtOnce \
  --region us-east-1
```

Confirm the tag filter finds the server:

```bash
aws deploy list-on-premises-instances \
  --tag-filters Key=Name,Value=onprem-mcardozo,Type=KEY_AND_VALUE \
  --region us-east-1
```

### appspec.yml

```yaml
version: 0.0
os: linux
files:
  - source: site
    destination: /var/www/html
file_exists_behavior: OVERWRITE
```

- `appspec.yml` must be at the **root of the zip** (zip the folder contents, not the folder).
- `source: site` publishes only the website files.
- `file_exists_behavior: OVERWRITE` is required because the web root already contained files not created by CodeDeploy.

### Manual deployment (optional sanity check)

Before automating, a deployment can be tested from the console: zip `appspec.yml` + `site/`, upload the zip to the bucket, then in **CodeDeploy → Applications → code-deploy-test-app → onprem-dg → Create deployment** choose the S3 location, file type `.zip`, and under *Content options* select **Overwrite the content** (this setting takes precedence over the appspec).

---

## Part 4 – GitHub Actions with OIDC

GitHub Actions authenticates to AWS using **OpenID Connect**: no AWS keys are stored in GitHub. Nothing is configured in the GitHub repository settings; all trust configuration lives in AWS IAM.

### 4.1 OIDC identity provider (once per AWS account)

IAM → **Identity providers** → Add provider:

- Provider type: **OpenID Connect**
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### 4.2 IAM role for GitHub Actions

Role `github-actions-codedeploy` with this **trust policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::975630231455:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:mauriCardozo08@87872544/AwsCodeDeployTest@1368954251:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

The `sub` condition restricts the role to the `main` branch of this repository. Note that GitHub includes the **immutable numeric IDs** of the owner and repository in the `sub` claim (`owner@ID/repo@ID`). This is safer than name-only matching because IDs are never reused, but it means the policy must be updated if the repository is deleted and recreated.

And this **permissions policy** (least privilege):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UploadArtifact",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::code-deploy-test-mc-975630231455/*"
    },
    {
      "Sid": "RegisterRevision",
      "Effect": "Allow",
      "Action": [
        "codedeploy:RegisterApplicationRevision",
        "codedeploy:GetApplicationRevision"
      ],
      "Resource": "arn:aws:codedeploy:us-east-1:975630231455:application:code-deploy-test-app"
    },
    {
      "Sid": "CreateAndTrackDeployment",
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetDeployment"
      ],
      "Resource": "arn:aws:codedeploy:us-east-1:975630231455:deploymentgroup:code-deploy-test-app/onprem-dg"
    },
    {
      "Sid": "ReadDeploymentConfig",
      "Effect": "Allow",
      "Action": "codedeploy:GetDeploymentConfig",
      "Resource": "arn:aws:codedeploy:us-east-1:975630231455:deploymentconfig:*"
    }
  ]
}
```

### 4.3 How the OIDC authentication works

1. The workflow requests an identity token from GitHub (enabled by `permissions: id-token: write`). GitHub signs it; its claims include the repository and branch (`sub`) and the audience (`aud`).
2. `aws-actions/configure-aws-credentials` sends the token and the role ARN to AWS STS (`AssumeRoleWithWebIdentity`). The role ARN already contains the account ID, so nothing else is needed.
3. STS verifies the token signature against GitHub's published public keys and checks `aud` and `sub` against the role's trust policy.
4. If everything matches, STS returns temporary credentials (1 hour by default) that the remaining steps use automatically.

The role ARN is not a secret: knowing it does not allow anyone to assume the role, because only tokens from this repository and branch satisfy the trust policy.

### 4.4 Workflow

`.github/workflows/deploy.yml`:

```yaml
name: Deploy to on-premise

on:
  push:
    branches: [main]
    paths:             # only deploy when the site or the deployment spec changes
      - 'site/**'
      - 'appspec.yml'
  workflow_dispatch:   # allows manual runs from the Actions tab

permissions:
  id-token: write      # required for OIDC
  contents: read

env:
  AWS_REGION: us-east-1
  BUCKET: code-deploy-test-mc-975630231455
  APP_NAME: code-deploy-test-app
  DEPLOY_GROUP: onprem-dg
  ROLE_ARN: arn:aws:iam::975630231455:role/github-actions-codedeploy

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Build artifact
        run: zip -r deploy.zip appspec.yml site

      - name: Upload to S3
        run: aws s3 cp deploy.zip s3://$BUCKET/$APP_NAME/${{ github.sha }}.zip

      - name: Create deployment and wait for result
        run: |
          DEPLOYMENT_ID=$(aws deploy create-deployment \
            --application-name $APP_NAME \
            --deployment-group-name $DEPLOY_GROUP \
            --s3-location bucket=$BUCKET,key=$APP_NAME/${{ github.sha }}.zip,bundleType=zip \
            --file-exists-behavior OVERWRITE \
            --description "Commit ${{ github.sha }}" \
            --query deploymentId --output text)
          echo "Deployment created: $DEPLOYMENT_ID"
          aws deploy wait deployment-successful --deployment-id $DEPLOYMENT_ID
          echo "Deployment completed successfully"
```

Each commit produces its own artifact named after the commit SHA, which makes it easy to trace what is deployed and to roll back by redeploying an older zip. The last step waits for the deployment, so a failure on the server turns the workflow red in GitHub.

### Running the workflow

- **Automatically:** push to `main`.
- **Manually:** Actions → *Deploy to on-premise* → **Run workflow** → branch `main`.
- **Re-run:** open a run → **Re-run jobs**. Note this reuses the same commit, so workflow changes are not picked up.

---

## How a deployment works end to end

1. A commit that changes `site/` or `appspec.yml` is pushed to `main` (other changes, like README edits, do not trigger the workflow).
2. GitHub Actions assumes `github-actions-codedeploy` via OIDC.
3. The workflow zips `appspec.yml` + `site/` and uploads it to S3.
4. The workflow calls `create-deployment` for `code-deploy-test-app` / `onprem-dg`.
5. CodeDeploy resolves the targets through the tag `Name=onprem-mcardozo`.
6. The agent on the server, which polls CodeDeploy, receives the command, downloads the zip from S3 using the IAM user credentials, and copies `site/` into `/var/www/html`.
7. nginx serves the new files immediately (no reload needed for static content).

---

## Troubleshooting

Issues found while building this setup, and how they were fixed.

### Agent log shows `InternalFailure` on `poll_host_command`

```
[Aws::CodeDeployCommand::Client 500 ...] poll_host_command(host_identifier:"arn:aws:iam::...:user/...")
Aws::CodeDeployCommand::Errors::InternalFailure
```

The network is fine (AWS answers), but CodeDeploy cannot map the agent identity to a registered instance. Check that:

- The instance is registered in the same region as the agent config (`get-on-premises-instance`, no `deregisterTime`).
- `iam_user_arn` in `codedeploy.onpremises.yml` matches the real IAM user **character by character**. In this project the cause was a typo (`on-prem` vs `onprem`).
- The access key in the config belongs to that user and is `Active` (`aws iam list-access-keys --user-name ...`).

The registered ARN cannot be edited: deregister and register the instance again, fix the config file and restart the agent.

### Deployment stuck in *In progress*

The agent never picked up the command (see the previous item). If all lifecycle events stay *Pending*, check the agent log. Deployments that are never picked up eventually time out and fail.

### Agent is slow / ~1 minute gap between polls

The agent tries to reach the EC2 metadata service (`169.254.169.254`), which does not exist on-premises, and waits for a timeout. Optionally make the request fail immediately:

```bash
sudo iptables -A OUTPUT -d 169.254.169.254 -j REJECT
sudo systemctl restart codedeploy-agent
```

### Deployment fails because files already exist

Files in the destination that were not created by CodeDeploy make the deployment fail by default. Use `file_exists_behavior: OVERWRITE` in `appspec.yml`, `--file-exists-behavior OVERWRITE` in the CLI, or *Overwrite the content* in the console.

### Deployment fails: appspec not found

`appspec.yml` must be at the root of the zip. On Windows, select the files inside the folder and compress them (not the folder itself). Prefer the Explorer's *Send to → Compressed folder* over PowerShell 5's `Compress-Archive`.

### GitHub Actions: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

The role's trust policy rejected the GitHub token. Check, in order:

1. The OIDC provider exists with URL `token.actions.githubusercontent.com` and audience `sts.amazonaws.com`.
2. The role name matches `ROLE_ARN` in the workflow.
3. The workflow ran from `main`.
4. The `sub` claim matches the trust policy exactly.

To see the real `sub`, open **CloudTrail → Event history**, filter by event name `AssumeRoleWithWebIdentity`, and look at `userIdentity.userName` of the failed event. In this project the token used the ID-based format `repo:owner@ID/repo@ID:ref:refs/heads/main`, while the policy expected `repo:owner/repo:...`.

Note: this error happens *before* the permissions policy is evaluated, so editing the permissions policy never fixes it.

### Useful log locations on the server

| Log | Path |
|---|---|
| Agent | `/var/log/aws/codedeploy-agent/codedeploy-agent.log` |
| Deployments (hooks, scripts) | `/opt/codedeploy-agent/deployment-root/deployment-logs/codedeploy-agent-deployments.log` |
