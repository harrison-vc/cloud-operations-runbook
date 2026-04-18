# Application Environment Misconfiguration

## Context
A backend API service (`billing-service`) is failing to connect to its PostgreSQL database after a secret rotation.

## Symptoms
- App logs report: `FATAL: password authentication failed for user "billing_user"`.
- All requests requiring database access fail with HTTP 500.
- Database logs confirm failed login attempts from the application.

## Initial Triage
1. Check app environment variables: `env | grep DB_`. Result: `DB_PASSWORD` is set.
2. Verify connectivity to DB: `nc -zv db-prod.cluster-xyz.us-east-1.rds.amazonaws.com 5432`. Result: `Succeeded`.
3. Manually test DB connection with credentials from environment: `psql -h $DB_HOST -U $DB_USER -d $DB_NAME`. Result: `authentication failed`.

## Investigation
1. Check secret source (AWS Secrets Manager):
   ```bash
   aws secretsmanager get-secret-value --secret-id prod/billing/db-creds
   ```
   Compare the password in Secrets Manager to the value in the application environment. Result: They do NOT match.
2. Verify Secret rotation status:
   ```bash
   aws secretsmanager describe-secret --secret-id prod/billing/db-creds
   ```
   Last rotated: 2 hours ago.
3. Check the application's secret-loading logic. Application only loads secrets during startup.

## Root Cause
The database credentials were rotated, but the application (running on EC2) did not pick up the change because it only reads environment variables from Secrets Manager at launch. No rolling restart was triggered after the rotation.

## Resolution
1. Force a rolling restart of the application to refresh the environment:
   ```bash
   aws autoscaling start-instance-refresh --auto-scaling-group-name billing-asg
   ```
2. Verify new instances have the correct password.

## Validation
- DB connection test on new instance: `psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT 1"` -> Returns `1`.
- API health check: `curl https://api.corp.com/v1/billing/status` -> `{"database": "connected"}`.

## Prevention
- Implement dynamic secret fetching in the application code (e.g., fetch from Secrets Manager on connection failure or periodically).
- Automate a rolling restart via Lambda hook triggered by Secrets Manager rotation event.
- Use HashiCorp Vault or AWS AppConfig for dynamic configuration management.