# MCP Server Setup

External setup required for each server in `servers.json`. Complete these steps before activating a server.

---

## io.github.awslabs/log-analyzer-cloudwatch

**Server:** `io.github.awslabs/log-analyzer-cloudwatch`
**Repository:** [awslabs/Log-Analyzer-with-MCP](https://github.com/awslabs/Log-Analyzer-with-MCP)

### Prerequisites

- **Python 3.12+** and [uv](https://docs.astral.sh/uv/getting-started/installation/) (provides the `uvx` command)
- An **AWS account** with CloudWatch Logs data

### AWS credentials setup

The server needs AWS credentials with CloudWatch Logs read access. Choose one approach:

1. **AWS CLI profiles** (recommended for local dev):
   - Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
   - Run `aws configure` or `aws configure sso` to set up a named profile
   - Set `AWS_PROFILE` to your profile name

2. **Static credentials** (for CI or environments without the CLI):
   - Create an IAM user or role with `CloudWatchLogsReadOnlyAccess`
   - Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`

3. **Instance/container roles** (for EC2, ECS, Lambda):
   - Attach a role with `CloudWatchLogsReadOnlyAccess` — no env vars needed

### Required IAM permissions

At minimum, the credentials need these CloudWatch Logs actions:

- `logs:DescribeLogGroups`
- `logs:DescribeLogStreams`
- `logs:GetLogEvents`
- `logs:FilterLogEvents`
- `logs:StartQuery`
- `logs:GetQueryResults`
- `logs:StopQuery`

The managed policy `CloudWatchLogsReadOnlyAccess` covers all of these.

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `AWS_PROFILE` | No | Named AWS CLI profile to use |
| `AWS_DEFAULT_REGION` | No | AWS region (e.g., `us-east-1`). Defaults to profile/CLI default. |
| `AWS_ACCESS_KEY_ID` | No | Access key for static credential auth |
| `AWS_SECRET_ACCESS_KEY` | No | Secret key for static credential auth |
| `AWS_SESSION_TOKEN` | No | Session token for temporary credentials |

### Verify setup

```bash
# Confirm AWS credentials are working and can access CloudWatch Logs
aws logs describe-log-groups --max-items 1
```

If this returns a log group, the server will work. See the [AWS config guide](https://github.com/awslabs/Log-Analyzer-with-MCP/blob/main/docs/aws-config.md) for more details.

---

## io.github.crystaldba/postgres-mcp

**Server:** `io.github.crystaldba/postgres-mcp`
**Repository:** [crystaldba/postgres-mcp](https://github.com/crystaldba/postgres-mcp)

### Prerequisites

- **Python 3.12+** and [uv](https://docs.astral.sh/uv/getting-started/installation/) (provides the `uvx` command)
- A **PostgreSQL** database you want to connect to

### Optional PostgreSQL extensions

For full functionality (index tuning and hypothetical index analysis), install these extensions on the target database:

- **pg_stat_statements** — query execution statistics (`CREATE EXTENSION IF NOT EXISTS pg_stat_statements;`)
- **hypopg** — hypothetical index simulation (`CREATE EXTENSION IF NOT EXISTS hypopg;`)

These are optional; the server works without them but some analysis features will be limited.

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `PG_USER` | Yes | PostgreSQL username |
| `PG_PASSWORD` | Yes | PostgreSQL password |
| `PG_HOST` | Yes | PostgreSQL hostname (e.g., `localhost`) |
| `PG_PORT` | Yes | PostgreSQL port (e.g., `5432`) |
| `PG_DATABASE` | Yes | PostgreSQL database name (e.g., `postgres`) |

These are composed into `DATABASE_URI` as `postgresql://$PG_USER:$PG_PASSWORD@$PG_HOST:$PG_PORT/$PG_DATABASE`.

### Access modes

The `--access-mode` flag controls what the server can do:

- `restricted` (default) — read-only queries with resource limits, safe for production
- `unrestricted` — full read/write access, intended for development

### Verify setup

```bash
# Confirm the database is reachable
psql "postgresql://$PG_USER:$PG_PASSWORD@$PG_HOST:$PG_PORT/$PG_DATABASE" -c "SELECT 1;"
```

If this returns a row, the server will be able to connect.

---

## io.github.awslabs/aws-iac-mcp-server

**Server:** `io.github.awslabs/aws-iac-mcp-server`
**Repository:** [awslabs/mcp](https://github.com/awslabs/mcp) (subfolder `src/aws-iac-mcp-server`)

### Prerequisites

- **Python 3.10+** and [uv](https://docs.astral.sh/uv/getting-started/installation/) (provides the `uvx` command)
- An **AWS account** (for CloudFormation validation and deployment troubleshooting)

### AWS credentials setup

The server needs AWS credentials for CloudFormation and CDK operations. Choose one approach:

1. **AWS CLI profiles** (recommended for local dev):
   - Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
   - Run `aws configure` or `aws configure sso` to set up a named profile
   - Set `AWS_PROFILE` to your profile name

2. **Static credentials** (for CI or environments without the CLI):
   - Create an IAM user or role with appropriate CloudFormation permissions
   - Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`

3. **Instance/container roles** (for EC2, ECS, Lambda):
   - Attach a role with appropriate CloudFormation permissions — no env vars needed

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `AWS_PROFILE` | No | Named AWS CLI profile to use |
| `AWS_DEFAULT_REGION` | No | AWS region (e.g., `us-east-1`). Defaults to profile/CLI default. |
| `AWS_ACCESS_KEY_ID` | No | Access key for static credential auth |
| `AWS_SECRET_ACCESS_KEY` | No | Secret key for static credential auth |
| `AWS_SESSION_TOKEN` | No | Session token for temporary credentials |
| `FASTMCP_LOG_LEVEL` | No | Logging level (default: `ERROR`) |

### Verify setup

```bash
# Confirm AWS credentials are working
aws sts get-caller-identity
```

If this returns your account info, the server will work.

---

## io.github.xing5/mcp-google-sheets

**Server:** `io.github.xing5/mcp-google-sheets`
**Repository:** [xing5/mcp-google-sheets](https://github.com/xing5/mcp-google-sheets)

### Prerequisites

- **Python 3.10+** and [uv](https://docs.astral.sh/uv/getting-started/installation/) (provides the `uvx` command)
- A **Google Cloud Platform** project

### Google Cloud setup

1. **Create or select a GCP project** at [console.cloud.google.com](https://console.cloud.google.com)

2. **Enable APIs** — go to **APIs & Services > Library** and enable:
   - Google Sheets API
   - Google Drive API

3. **Create a Service Account:**
   - Go to **APIs & Services > Credentials**
   - Click **+ Create Credentials > Service account**
   - Name it (e.g. `mcp-sheets-sa`), click **Create and Continue**
   - Skip the optional role/access steps, click **Done**
   - Click the new service account, go to the **Keys** tab
   - Click **Add Key > Create new key > JSON > Create** — a `.json` key file downloads

4. **Base64-encode the key** (no key file needed on disk at runtime):
   ```bash
   base64 -i /path/to/downloaded-key.json
   ```
   Copy the output — this is your `GOOGLE_SHEETS_CREDENTIALS_CONFIG` value.

5. **Create a shared Drive folder:**
   - Go to [drive.google.com](https://drive.google.com) and create a new folder (e.g. `MCP Sheets`)
   - Right-click the folder > **Share**
   - Paste the service account email (found in GCP console under **IAM & Admin > Service Accounts**, looks like `mcp-sheets-sa@your-project.iam.gserviceaccount.com`)
   - Give it **Editor** access, click **Send**
   - Open the folder — the folder ID is in the URL: `https://drive.google.com/drive/folders/<FOLDER_ID>`

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `GOOGLE_SHEETS_CREDENTIALS_CONFIG` | Yes | Base64-encoded Service Account JSON key |
| `GOOGLE_SHEETS_DRIVE_FOLDER_ID` | Yes | Google Drive folder ID shared with the service account |

### Verify setup

```bash
CREDENTIALS_CONFIG="$(cat <<< "$GOOGLE_SHEETS_CREDENTIALS_CONFIG")" \
DRIVE_FOLDER_ID="$GOOGLE_SHEETS_DRIVE_FOLDER_ID" \
uvx mcp-google-sheets==0.6.0
```

If the server starts without errors, the credentials and folder access are working.

---

## io.github.pulsemcp/playwright-stealth

**Server:** `io.github.pulsemcp/playwright-stealth`
**Repository:** [pulsemcp/mcp-servers](https://github.com/pulsemcp/mcp-servers) (subfolder `experimental/playwright-stealth`)

### Prerequisites

- **Node.js 18+** and npm (provides the `npx` command)
- **Playwright browsers** — installed automatically on first run, or manually with:
  ```bash
  npx playwright install chromium
  ```

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `STEALTH_MODE` | No | Enable anti-detection measures (default: `false`) |
| `HEADLESS` | No | Run browser without visible window (default: `true`) |
| `TIMEOUT` | No | Default action timeout in ms (default: `30000`) |
| `NAVIGATION_TIMEOUT` | No | Page navigation timeout in ms (default: `60000`) |
| `SCREENSHOT_STORAGE_PATH` | No | Directory for screenshots (default: `/tmp/playwright-screenshots`) |
| `PROXY_URL` | No | Proxy server URL |
| `PROXY_USERNAME` | No | Proxy auth username |
| `PROXY_PASSWORD` | No | Proxy auth password |

### Verify setup

```bash
npx -y playwright-stealth-mcp-server@0.0.9
```

If the server starts without errors, Playwright is installed and working.

---

## com.amazonaws/ecs-mcp

**Server:** `com.amazonaws/ecs-mcp`
**Docs:** [Amazon ECS MCP Server](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-mcp-getting-started.html)
**Proxy:** [aws/mcp-proxy-for-aws](https://github.com/aws/mcp-proxy-for-aws) (PyPI: `mcp-proxy-for-aws`)

### How it works

The ECS MCP server is a remote AWS-hosted service at `https://ecs-mcp.{region}.api.aws/mcp`. You connect to it through the `mcp-proxy-for-aws` local proxy, which handles AWS SigV4 request signing using your local AWS credentials.

### Prerequisites

- **Python 3.10+** and [uv](https://docs.astral.sh/uv/getting-started/installation/) (provides the `uvx` command)
- **AWS CLI** configured with credentials that have ECS access
- An **AWS account** with ECS clusters you want to monitor

### IAM permissions

The credentials need the following permissions:

**MCP-specific:**
- `ecs-mcp:InvokeReadOnlyTools`
- `ecs-mcp:UseMcp`

**ECS read-only:**
- `ecs:List*` and `ecs:Describe*` actions

**Supporting services (for full troubleshooting):**
- `logs:GetLogEvents`, `logs:FilterLogEvents` (CloudWatch Logs)
- `elasticloadbalancing:Describe*` (ELB)
- `ec2:Describe*` (VPC/networking)
- `ecr:Describe*` (container images)

### AWS credentials setup

Choose one approach:

1. **AWS CLI profiles** (recommended for local dev):
   - Run `aws configure` or `aws configure sso` to set up a profile
   - Set `AWS_PROFILE` to the profile name

2. **Static credentials** (for CI):
   - Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `AWS_PROFILE` | No | AWS CLI profile name |
| `AWS_DEFAULT_REGION` | No | AWS region (default: `us-east-1`) |
| `AWS_ACCESS_KEY_ID` | No | Access key for static credential auth |
| `AWS_SECRET_ACCESS_KEY` | No | Secret key for static credential auth |
| `AWS_SESSION_TOKEN` | No | Session token for temporary credentials |

At least one authentication method must be configured.

### Verify setup

```bash
# Confirm AWS credentials are working
aws sts get-caller-identity

# Confirm ECS access
aws ecs list-clusters --region us-east-1
```

If both commands succeed, the server will work.

---

## io.github.pulsemcp/dynamodb

**Server:** `io.github.pulsemcp/dynamodb`
**Repository:** [pulsemcp/mcp-servers](https://github.com/pulsemcp/mcp-servers) (subfolder `experimental/dynamodb`)
**Package:** [`aws-dynamodb-mcp-server`](https://www.npmjs.com/package/aws-dynamodb-mcp-server) (npm)

### Prerequisites

- **Node.js 18+** and npm (provides the `npx` command)
- An **AWS account** with DynamoDB access

### AWS credentials setup

Choose one approach:

1. **AWS CLI profiles** (recommended for local dev):
   - Run `aws configure` or `aws configure sso` to set up a profile
   - Set `AWS_PROFILE` to the profile name

2. **Static credentials** (for CI):
   - Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`

### IAM permissions

The credentials need DynamoDB access. At minimum:

- **Readonly tools**: `dynamodb:ListTables`, `dynamodb:DescribeTable`, `dynamodb:GetItem`, `dynamodb:Query`, `dynamodb:Scan`, `dynamodb:BatchGetItem`
- **ReadWrite tools**: above + `dynamodb:PutItem`, `dynamodb:UpdateItem`, `dynamodb:DeleteItem`, `dynamodb:BatchWriteItem`
- **Admin tools**: above + `dynamodb:CreateTable`, `dynamodb:DeleteTable`, `dynamodb:UpdateTable`

The managed policy `AmazonDynamoDBFullAccess` covers all of these.

### Tool access control

Fine-grained control over which tools are exposed:

| Variable | Description |
|---|---|
| `DYNAMODB_ENABLED_TOOL_GROUPS` | Comma-separated groups: `readonly`, `readwrite`, `admin` |
| `DYNAMODB_ENABLED_TOOLS` | Whitelist specific tools (overrides groups and disabled) |
| `DYNAMODB_DISABLED_TOOLS` | Blacklist specific tools |

Priority: `ENABLED_TOOLS` > `DISABLED_TOOLS` > `ENABLED_TOOL_GROUPS`

### Table access control

Restrict which tables the server can access:

| Variable | Description |
|---|---|
| `DYNAMODB_ALLOWED_TABLES` | Comma-separated list of table names. If unset, all tables accessible. |

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `AWS_REGION` | No | AWS region (default: `us-east-1`) |
| `AWS_ACCESS_KEY_ID` | No | Access key for static credential auth |
| `AWS_SECRET_ACCESS_KEY` | No | Secret key for static credential auth |
| `AWS_SESSION_TOKEN` | No | Session token for temporary credentials |
| `AWS_PROFILE` | No | AWS CLI profile name |
| `DYNAMODB_ENDPOINT` | No | Custom endpoint URL (for DynamoDB Local / LocalStack) |
| `DYNAMODB_ENABLED_TOOL_GROUPS` | No | Tool groups to enable |
| `DYNAMODB_ENABLED_TOOLS` | No | Specific tools to whitelist |
| `DYNAMODB_DISABLED_TOOLS` | No | Specific tools to blacklist |
| `DYNAMODB_ALLOWED_TABLES` | No | Tables to restrict access to |
| `SKIP_HEALTH_CHECKS` | No | Skip startup connectivity check (default: `false`) |

At least one authentication method must be configured.

### Verify setup

```bash
# Confirm AWS credentials are working
aws sts get-caller-identity

# Confirm DynamoDB access
aws dynamodb list-tables --region us-east-1
```

If both commands succeed, the server will work.

---

## io.github.pulsemcp/s3

**Server:** `io.github.pulsemcp/s3`
**Repository:** [pulsemcp/mcp-servers](https://github.com/pulsemcp/mcp-servers) (subfolder `experimental/s3`)
**Package:** [`s3-aws-mcp-server`](https://www.npmjs.com/package/s3-aws-mcp-server) (npm)

### Prerequisites

- **Node.js 18+** and npm (provides the `npx` command)
- An **AWS account** with S3 access

### AWS credentials setup

Choose one approach:

1. **AWS CLI profiles** (recommended for local dev):
   - Run `aws configure` or `aws configure sso` to set up a profile
   - Set `AWS_PROFILE` to the profile name

2. **Static credentials** (for CI):
   - Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`

### IAM permissions

The credentials need S3 access. At minimum:

- **Readonly tools**: `s3:ListAllMyBuckets`, `s3:ListBucket`, `s3:GetObject`, `s3:HeadBucket`
- **ReadWrite tools**: above + `s3:PutObject`, `s3:DeleteObject`, `s3:CreateBucket`, `s3:DeleteBucket`

The managed policy `AmazonS3FullAccess` covers all of these.

### Tool access control

Fine-grained control over which tools are exposed:

| Variable | Description |
|---|---|
| `S3_ENABLED_TOOLGROUPS` | Comma-separated groups: `readonly`, `readwrite` |
| `S3_ENABLED_TOOLS` | Whitelist specific tools (overrides groups) |
| `S3_DISABLED_TOOLS` | Blacklist specific tools |

### Single-bucket mode

Set `S3_BUCKET` to constrain all operations to one bucket. When set:
- Bucket-level tools (`list_buckets`, `create_bucket`, `delete_bucket`, `head_bucket`) are hidden
- Object-level tools automatically inject the bucket parameter
- Startup health check validates the constrained bucket exists

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Yes | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Yes | AWS secret key |
| `AWS_REGION` | No | AWS region (default: `us-east-1`) |
| `AWS_ENDPOINT_URL` | No | Custom S3-compatible endpoint (MinIO, LocalStack) |
| `S3_BUCKET` | No | Constrain to a single bucket |
| `S3_ENABLED_TOOLGROUPS` | No | Tool groups to enable |
| `S3_ENABLED_TOOLS` | No | Specific tools to whitelist |
| `S3_DISABLED_TOOLS` | No | Specific tools to blacklist |
| `SKIP_HEALTH_CHECKS` | No | Skip startup validation (default: `false`) |

### Verify setup

```bash
# Confirm AWS credentials are working
aws sts get-caller-identity

# Confirm S3 access
aws s3 ls
```

If both commands succeed, the server will work.

---

## io.github.chromedevtools/chrome-devtools-mcp

**Server:** `io.github.chromedevtools/chrome-devtools-mcp`
**Repository:** [ChromeDevTools/chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)
**Package:** [`chrome-devtools-mcp`](https://www.npmjs.com/package/chrome-devtools-mcp) (npm)

### Prerequisites

- **Node.js v20.19+** or **v22.12+** (LTS releases) and npm (provides the `npx` command)
- **Chrome** current stable version or newer

No credentials or API keys are required — the server controls a local Chrome browser. See `servers.json` for the full list of environment variables and CLI flags.

### Remote debugging setup (for `chrome-devtools-remote`)

The `chrome-devtools-remote` mcp.json entry uses `--autoConnect` to attach to a Chrome instance that is already running with remote debugging enabled.

**Launching Chrome with remote debugging:**

```bash
# macOS
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222

# Linux
google-chrome --remote-debugging-port=9222

# Windows
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222
```

On Chrome 144+, you can also enable remote debugging by navigating to `chrome://inspect/#remote-debugging` within Chrome. Alternatively, on any Chrome version, you can add `--remote-debugging-port=9222` to your Chrome desktop shortcut.

Once Chrome is running with remote debugging, the `--autoConnect` flag will automatically discover and connect to it (requires Chrome 144+). For older Chrome versions, use `--browserUrl http://localhost:9222` instead.

### Puppeteer dependencies (headless / Linux servers)

The `chrome-devtools-headless` entry launches its own Chrome instance via Puppeteer. On headless Linux servers or CI environments, Chromium may need system-level dependencies:

```bash
# Debian/Ubuntu — install Chromium dependencies
sudo apt-get update && sudo apt-get install -y \
  ca-certificates fonts-liberation libasound2 libatk-bridge2.0-0 \
  libatk1.0-0 libcups2 libdbus-1-3 libdrm2 libgbm1 libgtk-3-0 \
  libnspr4 libnss3 libx11-xcb1 libxcomposite1 libxdamage1 \
  libxrandr2 xdg-utils

# Or install Chromium directly (npx will download its own, but system deps are still needed)
sudo apt-get install -y chromium-browser
```

On macOS and Windows with Chrome already installed, no extra dependencies are needed.

### Verify setup

```bash
# Confirm Node.js version is sufficient
node --version  # should be v20.19.0 or newer

# Confirm Chrome is installed
google-chrome --version || chromium-browser --version
```

If both commands return versions, the server will work.

---

## io.github.neondatabase/mcp-server-neon

**Server:** `io.github.neondatabase/mcp-server-neon`
**Repository:** [neondatabase/mcp-server-neon](https://github.com/neondatabase/mcp-server-neon)
**Package:** [`@neondatabase/mcp-server-neon`](https://www.npmjs.com/package/@neondatabase/mcp-server-neon) (npm)
**Docs:** [Neon MCP Server](https://neon.com/docs/ai/neon-mcp-server)

### Prerequisites

- **Node.js 18+** and npm (provides the `npx` command) — only needed for the local stdio package
- A **Neon account** at [neon.tech](https://neon.tech)

### Connection modes

The Neon MCP server can be used in two ways:

1. **Remote (recommended)** — connect to the hosted server at `https://mcp.neon.tech/mcp` via Streamable HTTP (or `https://mcp.neon.tech/sse` via SSE, deprecated). Supports OAuth browser-based authorization or API key auth via the `Authorization` header.

2. **Local** — run the npm package `@neondatabase/mcp-server-neon` locally via `npx`. Requires a Neon API key passed as a CLI argument. Set `NEON_API_KEY` in your shell so `mcp.json` can substitute it into the args array.

### API key setup

1. Log in to the [Neon Console](https://console.neon.tech)
2. Go to **Account Settings > API Keys**
3. Click **Generate new API key**, copy the value
4. Set it as `NEON_API_KEY` in your environment or pass it directly to the CLI

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `NEON_API_KEY` | Yes (local) / No (remote with OAuth) | Neon API key. The local server reads this as a CLI argument, not an env var — set it in your shell so `mcp.json` can substitute it into the args array. |

### Read-only mode

Add the `x-read-only: true` header (remote) to restrict the server to safe, read-only operations. This is useful for production environments where you want to prevent accidental modifications.

### Verify setup

```bash
# Confirm Node.js is available (for local mode)
node --version  # should be v18 or newer

# Confirm your API key works
curl -s -H "Authorization: Bearer $NEON_API_KEY" https://console.neon.tech/api/v2/projects | head -c 200
```

If the curl command returns project data, the API key is valid and the server will work.

---

## com.langfuse/langfuse

**Server:** `com.langfuse/langfuse`
**Docs:** [Langfuse MCP Server](https://langfuse.com/docs/api-and-data-platform/features/mcp-server)
**Repository:** [langfuse/langfuse](https://github.com/langfuse/langfuse)

### How it works

The Langfuse MCP server is a remote cloud-hosted service. You connect to it directly via streamable HTTP at `https://<host>/api/public/mcp`, authenticating with a base64-encoded API key pair in the `Authorization` header.

### Prerequisites

- A **Langfuse** account (cloud or self-hosted)
- A **project-scoped API key pair** (public key `pk-lf-...` and secret key `sk-lf-...`)

### API key setup

1. Log in to your Langfuse instance
2. Go to **Settings > API Keys**
3. Create a new API key pair — you'll get a public key (`pk-lf-...`) and secret key (`sk-lf-...`)
4. Base64-encode the key pair:
   ```bash
   echo -n 'pk-lf-YOUR_PUBLIC_KEY:sk-lf-YOUR_SECRET_KEY' | base64
   ```
5. Use the base64 output as `LANGFUSE_API_CREDENTIALS`

### Endpoints by region

| Region | Host |
|---|---|
| EU (default) | `cloud.langfuse.com` |
| US | `us.cloud.langfuse.com` |
| HIPAA | `hipaa.cloud.langfuse.com` |
| Self-hosted | Your domain (e.g., `langfuse.example.com`) |

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `LANGFUSE_API_CREDENTIALS` | Yes | Base64-encoded API key pair (`pk-lf-...:sk-lf-...`) |
| `LANGFUSE_HOST` | No | Langfuse hostname (default: `cloud.langfuse.com`) |

### Changing regions

The `mcp.json` entry defaults to the EU endpoint (`cloud.langfuse.com`). To use a different region or a self-hosted instance, change the `url` in `mcp-servers/mcp.json` to match your host (see the table above).

### Verify setup

```bash
# Confirm API credentials are valid
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Basic $LANGFUSE_API_CREDENTIALS" \
  "https://${LANGFUSE_HOST:-cloud.langfuse.com}/api/public/mcp"
```

A `200` or `405` response confirms the credentials are valid and the endpoint is reachable.

---

## io.github.pulsemcp/vercel

**Server:** `io.github.pulsemcp/vercel`
**Repository:** [pulsemcp/mcp-servers](https://github.com/pulsemcp/mcp-servers) (subfolder `experimental/vercel`)
**Package:** [`vercel-platform-mcp-server`](https://www.npmjs.com/package/vercel-platform-mcp-server) (npm)

### Prerequisites

- **Node.js 18+** and npm (provides the `npx` command)
- A **Vercel account** at [vercel.com](https://vercel.com)

### API token setup

1. Log in to the [Vercel Dashboard](https://vercel.com/dashboard)
2. Go to **Account Settings > Tokens** (or visit [vercel.com/account/tokens](https://vercel.com/account/tokens))
3. Click **Create**, give the token a name, select the scope (team or personal), and set an expiration
4. Copy the token — this is your `VERCEL_TOKEN` value

### Team-scoped access

If you want to scope operations to a specific team:

- **VERCEL_TEAM_ID** — find it in your team's **Settings > General** page in the Vercel Dashboard
- **VERCEL_TEAM_SLUG** — the URL slug for your team (e.g., `my-team` from `vercel.com/my-team`)

Either `VERCEL_TEAM_ID` or `VERCEL_TEAM_SLUG` can be used; both are optional for personal account access.

### Tool access control

Fine-grained control over which tools are exposed:

| Variable | Description |
|---|---|
| `VERCEL_ENABLED_TOOLGROUPS` | Comma-separated groups: `readonly`, `readwrite` |

- **readonly**: list deployments, get deployment details, list projects, view build events, view runtime logs
- **readwrite**: above + create, cancel, delete, promote, and rollback deployments

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `VERCEL_TOKEN` | Yes | Vercel API token |
| `VERCEL_TEAM_ID` | No | Team ID for team-scoped operations |
| `VERCEL_TEAM_SLUG` | No | Team URL slug for team-scoped operations |
| `VERCEL_ENABLED_TOOLGROUPS` | No | Tool groups to enable (default: all) |

### Verify setup

```bash
# Confirm your Vercel token works
curl -s -H "Authorization: Bearer $VERCEL_TOKEN" https://api.vercel.com/v9/projects | head -c 200
```

If the curl command returns project data, the token is valid and the server will work.

---

## io.github.pulsemcp/langfuse-observability

**Server:** `io.github.pulsemcp/langfuse-observability`
**Repository:** [pulsemcp/mcp-servers](https://github.com/pulsemcp/mcp-servers) (subfolder `experimental/langfuse`)
**Package:** [`langfuse-observability-mcp-server`](https://www.npmjs.com/package/langfuse-observability-mcp-server) (npm)

### How it works

This is a **local** MCP server that connects to the Langfuse API to provide readonly access to LLM traces and observations. It complements the remote `com.langfuse/langfuse` server (which manages prompts) by focusing on observability and trace analysis.

### Prerequisites

- **Node.js 18+** and npm (provides the `npx` command)
- A **Langfuse** account (cloud or self-hosted) with tracing data

### API key setup

1. Log in to your Langfuse instance
2. Go to **Settings > API Keys**
3. Create a new API key pair — you'll get a public key (`pk-lf-...`) and secret key (`sk-lf-...`)
4. Set `LANGFUSE_PUBLIC_KEY` to the public key and `LANGFUSE_SECRET_KEY` to the secret key

### Endpoints by region

| Region | Base URL |
|---|---|
| EU (default) | `https://cloud.langfuse.com` |
| US | `https://us.cloud.langfuse.com` |
| Self-hosted | Your instance URL (e.g., `https://langfuse.example.com`) |

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `LANGFUSE_SECRET_KEY` | Yes | Langfuse secret key (`sk-lf-...`) |
| `LANGFUSE_PUBLIC_KEY` | Yes | Langfuse public key (`pk-lf-...`) |
| `LANGFUSE_BASE_URL` | No | Langfuse instance base URL (default: `https://cloud.langfuse.com`) |

### Verify setup

```bash
# Confirm API credentials are valid
curl -s -o /dev/null -w "%{http_code}" \
  -u "$LANGFUSE_PUBLIC_KEY:$LANGFUSE_SECRET_KEY" \
  "${LANGFUSE_BASE_URL:-https://cloud.langfuse.com}/api/public/traces?limit=1"
```

A `200` response confirms the credentials are valid and the server will work.
