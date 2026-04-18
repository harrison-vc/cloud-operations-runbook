# Security Group Blocking Inbound Traffic

## Context
Production web server (EC2 instance) sitting in a public subnet. Security Group `web-sg-prod` is attached.

## Symptoms
- Connection timed out when attempting to reach the server on port 443.
- Application logs show zero incoming traffic during the outage.
- Server is pingable (ICMP allowed), but TCP connection fails.

## Initial Triage
1. Check instance health: `gh api repos/:owner/:repo/actions/runs` (N/A) -> Check AWS CLI: `aws ec2 describe-instance-status --instance-ids i-0123456789abcdef0`. Status is `running`.
2. Verify application listener: `ss -tulpn | grep 443` (Done on instance). Result: Process is listening on `0.0.0.0:443`.
3. Test connectivity locally: `curl -k https://localhost` (Done on instance). Result: Success.

## Investigation & Triage Details
1. Confirm external blocking: `nc -zv <public-ip> 443`. Result: `Connection timed out`.
2. Inspect Security Group rules:
   ```bash
   aws ec2 describe-security-groups --group-ids sg-0123456789abcdef0 --query 'SecurityGroups[0].IpPermissions'
   ```
3. Analyze rules: Inbound rules show Port 80 is open to `0.0.0.0/0`, but Port 443 is missing from the whitelist.

## Root Cause
A recent Infrastructure as Code (Terraform) change omitted the port 443 ingress rule during a refactor of the `web-sg-prod` module.

## Resolution
1. Manually add the missing ingress rule via CLI for immediate restoration:
   ```bash
   aws ec2 authorize-security-group-ingress --group-id sg-0123456789abcdef0 --protocol tcp --port 443 --cidr 0.0.0.0/0
   ```
2. Re-apply Terraform state with the corrected module configuration to ensure persistence.

## Validation
- External check: `nc -zv <public-ip> 443` -> `Connection to <public-ip> 443 port [tcp/https] succeeded!`.
- Browser check: Application is reachable and TLS handshake completes.

## Prevention
- Implement Terraform `tflint` and `checkov` in the CI pipeline to catch missing mandatory ports.
- Add an automated probe (e.g., CloudWatch Synthetics) to alert immediately on port 443 availability.