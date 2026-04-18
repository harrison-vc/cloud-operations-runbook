# Systemd Service Not Running (Crash Loop)

## Context
A mission-critical Node.js microservice (`order-processor`) is failing to start on a fleet of Linux VMs (Ubuntu 22.04).

## Symptoms
- `systemctl status order-processor` shows `failed (Result: exit-code)`.
- Rapid restart attempts (Flapping).
- Service is completely unavailable.

## Initial Triage
1. Check service status: `systemctl status order-processor`. Result: `Active: failed (Result: exit-code) since...`.
2. Inspect latest logs: `journalctl -u order-processor -n 100 --no-pager`.
3. Check process tree: `ps aux | grep node`. Result: No process found.

## Investigation
1. Log analysis: `journalctl -u order-processor` reveals `Error: Cannot find module '../config/prod.json'`.
2. Verify environment files: `ls -l /etc/order-processor/config/`. Result: `prod.json` is missing.
3. Check for recent deployments: `history | grep deploy`. Result: Service was updated via a script 10 minutes ago.
4. Verify file permissions: `ls -ld /etc/order-processor/config/` is owned by `root:root`, but service runs as `nodeuser`.

## Root Cause
The deployment script failed to copy the configuration file `prod.json` to the target directory on the new instances, and permissions were not correctly set for the `nodeuser` service account.

## Resolution
1. Manually restore the missing config file from the artifacts repository:
   ```bash
   cp /tmp/prod.json /etc/order-processor/config/
   ```
2. Correct the ownership and permissions:
   ```bash
   chown -R nodeuser:nodeuser /etc/order-processor/
   chmod 600 /etc/order-processor/config/prod.json
   ```
3. Restart the service:
   ```bash
   systemctl restart order-processor
   ```

## Validation
- Check status: `systemctl status order-processor` -> `active (running)`.
- Verify listener: `ss -tulpn | grep :3000` -> Process is bound to port 3000.
- Health check: `curl localhost:3000/health` -> `{"status":"ok"}`.

## Prevention
- Enhance deployment script to include verification steps (e.g., `test -f /etc/order-processor/config/prod.json`).
- Implement health check validation post-deployment (Wait for status 200 before finalizing rollout).
- Use configuration management (Ansible/Chef) or containerization (Docker) to ensure environmental consistency.
- Add systemd `StartLimitIntervalSec` and `StartLimitBurst` to prevent excessive flapping.