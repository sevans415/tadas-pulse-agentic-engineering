# MCP Server Setup

External setup required for each server in `servers.json`. Complete these steps before activating a server.

---

## Log Analyzer with CloudWatch Logs

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
