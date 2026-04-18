[![Lint](https://github.com/harrison-vc/cloud-support-runbook/actions/workflows/lint.yml/badge.svg)](https://github.com/harrison-vc/cloud-support-runbook/actions/workflows/lint.yml)

# cloud-support-runbook

A structured collection of cloud support runbooks for common infrastructure and application issues. This repository demonstrates a methodological approach to troubleshooting, root cause analysis, and incident prevention.

## Structure

```text
runbooks/
├── networking/   # DNS, Security Groups, VPC, Load Balancers
├── compute/      # VM management, Linux systems, Disk, CPU
├── identity/     # IAM, Permissions, Service Accounts
└── application/  # API failures, Environment config, Timeouts
```

## Methodology

Each runbook follows a standard, production-grade format:

- **Context**: The environment and architecture involved.
- **Symptoms**: Observed errors, metrics, or reported behavior.
- **Initial Triage**: Rapid checks to isolate the fault domain.
- **Investigation**: Step-by-step diagnostic process with specific commands.
- **Root Cause**: The underlying failure mechanism.
- **Resolution**: Steps taken to restore service.
- **Validation**: How to confirm the fix was successful.
- **Prevention**: Changes implemented to avoid recurrence.

## Usage

These runbooks are intended for internal support engineering and SRE teams to standardize incident response and reduce Mean Time To Resolution (MTTR).
