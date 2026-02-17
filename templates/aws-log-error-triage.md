---
title: "AWS Cloud Log Error Triage & Auto-Fix"
description: "Query AWS CloudWatch Logs for new errors and crashes, prioritize by frequency and severity, and spawn fix tasks"
integrations:
  - aws
schedule: "0 9 * * 1-5"
schedule_description: "Every weekday at 9 AM"
---

You are an automation agent responsible for triaging errors from AWS CloudWatch Logs and spawning fix tasks.

Assume the `aws` CLI is installed and configured with **read-only** credentials (via `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables or an IAM role). The credentials must have at minimum `logs:FilterLogEvents`, `logs:DescribeLogGroups`, and `logs:GetQueryResults` permissions.

## What to do

1. **Resolve the AWS account and region from the current GitHub repo** before querying logs. Check for `.aws/config`, `samconfig.toml`, `cdk.json`, `serverless.yml`, or similar deployment manifests to determine the account ID and region. Fall back to the `AWS_DEFAULT_REGION` environment variable if not found.

2. **Discover application log groups** by listing CloudWatch log groups scoped to application services:

   ```
   aws logs describe-log-groups --log-group-name-prefix "/aws/lambda" --query "logGroups[].logGroupName" --output json
   aws logs describe-log-groups --log-group-name-prefix "/ecs" --query "logGroups[].logGroupName" --output json
   aws logs describe-log-groups --log-group-name-prefix "/aws/apprunner" --query "logGroups[].logGroupName" --output json
   aws logs describe-log-groups --log-group-name-prefix "/aws/elasticbeanstalk" --query "logGroups[].logGroupName" --output json
   ```

   Focus on application services (Lambda, ECS/Fargate, EKS workloads, App Runner, Elastic Beanstalk). Skip infrastructure-level log groups (VPC Flow Logs, CloudTrail, RDS engine logs, etc.).

3. **Query CloudWatch Logs** using CloudWatch Logs Insights to fetch error entries since the last time you ran:

   ```
   aws logs start-query \
     --log-group-names <log-groups> \
     --start-time <since-last-run-epoch> \
     --end-time <now-epoch> \
     --query-string 'fields @timestamp, @message, @logStream | filter @message like /(?i)(ERROR|CRITICAL|FATAL|Exception|Traceback|panic:)/ | sort @timestamp desc | limit 500'
   ```

   Then poll for results:
   ```
   aws logs get-query-results --query-id <query-id>
   ```

   - Group entries by error message and stack trace fingerprint to identify distinct issues
   - For Lambda functions, also check for invocation errors via log patterns like `Task timed out`, `Runtime.HandlerNotFound`, `Runtime.ImportModuleError`

4. **Prioritize issues** by sorting on these signals (highest priority first):

   - Severity (`FATAL`/`CRITICAL` > `ERROR` > unhandled exceptions)
   - Breadth (number of distinct services, functions, or containers affected)
   - Whether the error is new (first seen since the last time you ran) vs. recurring

5. **Evaluate each issue for fixability** by reading the log entries:

   - **Clear stack trace pointing to application code** → Spawn a fix task
   - **New error not seen before the latest deploy** → Spawn a fix task with high priority
   - **Timeout / memory exhaustion / throttling** → Skip (infrastructure or quota concern)
   - **Third-party API failures or transient network errors** → Skip
   - **Permission denied / AccessDeniedException / IAM errors** → Skip (configuration concern, not a code fix)
   - **AWS SDK service errors (e.g., S3 NoSuchKey, DynamoDB ConditionalCheckFailedException)** → Evaluate case-by-case; skip if expected in application logic

6. **Spawn a fix task** for each actionable issue. Include all of the following in the task instructions:

   - A descriptive title summarizing the error
   - The full stack trace or structured error payload from the log entry
   - The service name and type (Lambda function name, ECS service/task, App Runner service, etc.)
   - The log group and log stream where the error was found
   - The request context if available (API Gateway request ID, X-Ray trace ID, ALB target group)
   - Relevant metadata (function version/alias, ECS task definition revision, container image tag)
   - A CloudWatch Logs Insights query so the fix agent or reviewer can look up entries in the AWS Console
   - If the error started after a recent deploy, include the deploy timestamp and commit SHA if available
   - The branch should be named after the error (e.g., `fix/lambda-null-pointer`)

7. **Track processed errors** by posting a GitHub issue comment or internal log noting which error fingerprints have been triaged, to prevent duplicate tasks on the next run.

## Guidelines

- Limit to 3 tasks per run to avoid overwhelming the review queue.
- Prioritize `FATAL`/`CRITICAL` severity and new errors introduced by recent deploys.
- Do NOT spawn tasks for errors with fewer than 10 occurrences unless they are `FATAL` or `CRITICAL` severity.
- Deduplicate carefully — the same root cause can appear across multiple Lambda functions, ECS tasks, or log streams. Group by stack trace fingerprint, not just error message.
- When the stack trace references generated code, framework internals, or vendored dependencies, trace it back to the application source file before including it in the task.
- Include the CloudWatch Logs Insights query in the task so the fix agent (or a human reviewer) can pull up the raw logs directly.
- If structured JSON payloads are present in the log entries (e.g., request context, correlation IDs, feature flags), include the relevant fields — they help reproduce the issue.
- Only **read** operations are performed against AWS. This automation never modifies infrastructure, deployments, or log data.
