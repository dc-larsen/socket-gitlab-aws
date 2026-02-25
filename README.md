# Socket GitLab AWS Deployment

CloudFormation template for deploying the [Socket GitLab integration](https://github.com/SocketDev/socket-gitlab) on AWS. Deploys a single EC2 instance running Docker Compose with the application and PostgreSQL.

## Prerequisites

Before deploying, you need:

1. **A Socket organization** dedicated to this integration. Create one at https://socket.dev/dashboard/create-organization
2. **A Socket API token** with scopes: `full-scans` (all), `diff-scans` (all), `repo` (all). Create under **Settings > Integrations > API Tokens** in your Socket org.
3. **Your GitLab group ID**. Find it on the group page: click `...` > "Copy group ID"
4. **A GitLab access token** (group or personal) with scopes: `api`, `read_repository`. For group tokens: **Settings > Access Tokens**. Role: Developer or higher.
5. **An EC2 key pair** in the target AWS region. Create one in the [EC2 console](https://console.aws.amazon.com/ec2/home#KeyPairs) if you don't have one.

## Deploy

### Option A: AWS Console

1. Open [CloudFormation > Create Stack](https://console.aws.amazon.com/cloudformation/home#/stacks/create)
2. Select **Upload a template file**, upload `cloudformation.yaml`
3. Fill in the parameters:

| Parameter | Description |
|-----------|-------------|
| **SocketOrg** | Your Socket organization slug |
| **SocketApiKey** | Socket API token |
| **GitlabToken** | GitLab access token |
| **GitlabGroupId** | GitLab group ID to scan |
| **GitlabInstance** | GitLab URL (default: `https://gitlab.com`, change for self-hosted) |
| **KeyPairName** | EC2 key pair for SSH access |
| **InstanceType** | EC2 instance size (default: `t3.small`) |
| **SSHCidrIp** | IP range allowed to SSH in (restrict to your IP, e.g. `203.0.113.5/32`) |
| **WebhookCidrIp** | IP range allowed to reach the webhook port (restrict to GitLab IPs if possible) |

4. Click through to **Create stack**
5. Wait for status: `CREATE_COMPLETE` (takes 3-5 minutes)

### Option B: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name socket-gitlab \
  --template-body file://cloudformation.yaml \
  --parameters \
    ParameterKey=SocketOrg,ParameterValue=your-org \
    ParameterKey=SocketApiKey,ParameterValue=sktsec_yourtoken \
    ParameterKey=GitlabToken,ParameterValue=glpat-yourtoken \
    ParameterKey=GitlabGroupId,ParameterValue=123456 \
    ParameterKey=KeyPairName,ParameterValue=your-keypair

aws cloudformation wait stack-create-complete --stack-name socket-gitlab
aws cloudformation describe-stacks --stack-name socket-gitlab --query 'Stacks[0].Outputs'
```

## After Deployment

### 1. Get the webhook URL

The stack outputs include **WebhookURL**. Find it in the CloudFormation console under **Outputs**, or:

```bash
aws cloudformation describe-stacks --stack-name socket-gitlab \
  --query 'Stacks[0].Outputs[?OutputKey==`WebhookURL`].OutputValue' --output text
```

### 2. Configure the GitLab webhook

The application creates a group webhook automatically on startup. If your GitLab instance can reach the EC2 instance on port 5050, no manual webhook setup is needed.

If you're using **GitLab Free tier** (no group webhooks), create a project-level webhook for each project:

1. SSH into the instance and get the webhook secret:
   ```bash
   ssh ec2-user@<PublicIP>
   docker exec socket-gitlab-db-1 psql -U socket socket-gitlab \
     -c "SELECT token FROM gitlab_webhook_configs LIMIT 1;"
   ```
2. In each GitLab project, go to **Settings > Webhooks > Add new webhook**
   - URL: the **WebhookURL** from stack outputs
   - Secret token: the value from step 1
   - Triggers: Push events, Merge request events

### 3. Verify

```bash
# Health check
curl http://<PublicIP>:5050/health
# Expected: {"statusCode":200,"status":"ok"}

# Check logs
ssh ec2-user@<PublicIP>
cd /opt/socket-gitlab
docker compose logs -f app
```

Look for these log messages on startup:
- `pg-boss started` (database connected)
- `GitLab webhook configured` (webhook created on GitLab)
- `Server listening at ...` (ready)
- `SYNCED PROJECTS` (GitLab projects mirrored to Socket)

### 4. Test

Push a commit to any project in the configured GitLab group. Within a few seconds, the logs should show `Found N manifest files` and the scan will appear in your Socket dashboard under **Scans**.

## Architecture

```
┌─────────────────────────────────────┐
│         EC2 Instance (t3.small)     │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  socket-gitlab container    │    │◄── GitLab webhooks (port 5050)
│  │  - Webhook server           │    │──► api.socket.dev
│  │  - Background worker        │    │──► GitLab API
│  └──────────┬──────────────────┘    │
│             │                       │
│  ┌──────────▼──────────────────┐    │
│  │  PostgreSQL 17 container    │    │
│  │  - Job queue state          │    │
│  │  - Project/scan mappings    │    │
│  └─────────────────────────────┘    │
│                                     │
│  EBS volume (20GB gp3)             │
└─────────────────────────────────────┘
         │
    Elastic IP (stable address)
```

The database stores only identifiers and mapping state (project IDs, git hashes, scan IDs). No source code or dependency data is persisted locally.

## Updating

To pull the latest Socket GitLab image:

```bash
ssh ec2-user@<PublicIP>
cd /opt/socket-gitlab
docker compose pull app
docker compose up -d app
```

## Tearing Down

```bash
aws cloudformation delete-stack --stack-name socket-gitlab
```

This removes all AWS resources created by the stack, including the EBS volume and its data.

## Troubleshooting

**Stack creation fails at Instance resource**
- Check that the key pair exists in the target region
- Check CloudFormation Events tab for the specific error

**Health check returns connection refused**
- Allow a few minutes for Docker to pull images and start containers on first boot
- SSH in and check: `docker compose -f /opt/socket-gitlab/docker-compose.yml ps`

**No scans after pushing**
- Verify GitLab can reach the webhook URL. The EC2 security group must allow inbound on port 5050 from GitLab's IP range.
- Check app logs: `docker compose -f /opt/socket-gitlab/docker-compose.yml logs app`

**"No valid x-gitlab-token" in logs**
- The webhook secret doesn't match. Restart the app container: `docker compose -f /opt/socket-gitlab/docker-compose.yml restart app`

**Scans have fewer alerts than expected**
- Check if your repos have lockfiles committed (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`). Without lockfiles, only direct dependencies are scanned.
