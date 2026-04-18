# DNS Resolution Failure (NXDOMAIN)

## Context
Applications in an AWS VPC are failing to reach an internal API service at `api.internal.corp`.

## Symptoms
- App logs report: `Could not resolve host: api.internal.corp`.
- Internal services report timeouts during DNS lookups.
- `ping` and `curl` to the target domain fail immediately with "name or service not known".

## Initial Triage
1. Test local DNS resolution: `dig api.internal.corp`. Result: `NXDOMAIN`.
2. Verify connectivity to resolver: `nc -zv 169.254.169.253 53` (VPC DNS). Result: `Succeeded`.
3. Check external resolution: `dig google.com`. Result: `Succeeded`.

## Investigation
1. Inspect Resolver rules:
   ```bash
   aws route53resolver list-resolver-rule-associations --filters "Name=VPCId,Values=vpc-01234567"
   ```
2. Verify Private Hosted Zone (PHZ) association:
   ```bash
   aws route53 list-hosted-zones-by-vpc --vpc-id vpc-01234567 --vpc-region us-east-1
   ```
3. Analyze PHZ records: PHZ `internal.corp` is correctly associated, but the record `api` is missing from the record set.

## Root Cause
The record `api.internal.corp` was accidentally deleted during a Route 53 cleanup task that targeted stale environments.

## Resolution
1. Re-create the DNS record in the Private Hosted Zone:
   ```bash
   aws route53 change-resource-record-sets --hosted-zone-id Z1234567 --change-batch '{"Changes":[{"Action":"CREATE","ResourceRecordSet":{"Name":"api.internal.corp","Type":"A","AliasTarget":{"HostedZoneId":"Z9876543","DNSName":"internal-lb.us-east-1.elb.amazonaws.com","EvaluateTargetHealth":false}}}]}'
   ```
2. Verify record propagation across VPC instances.

## Validation
- DNS lookup: `dig api.internal.corp +short` -> Returns the correct Load Balancer alias IP.
- Connectivity test: `curl -I http://api.internal.corp/health` -> `HTTP/1.1 200 OK`.

## Prevention
- Enable Route 53 Query Logging to track deletion events.
- Implement resource tagging and "protected" flags in IaC (e.g., `lifecycle { prevent_destroy = true }` in Terraform).
- Monitor `NXDOMAIN` count in CloudWatch to detect mass deletion events early.