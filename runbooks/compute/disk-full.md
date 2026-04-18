# Disk Space Exhaustion (Root Partition Full)

## Context
A data processing server is unresponsive, and SSH attempts are failing or very slow.

## Symptoms
- App logs show: `Error: No space left on device`.
- Cron jobs and system utilities are failing.
- CloudWatch metrics show root EBS volume at 100% usage.

## Initial Triage
1. Check disk usage: `df -h`. Result: `/` is at 100%.
2. Identify largest directories: `du -ah / | sort -rh | head -n 20`.
3. Verify free inodes: `df -i`. Result: Inodes are 85% free (Space is the issue, not inodes).

## Investigation
1. Log investigation: `du -sh /var/log/*`. Result: `/var/log/syslog` and `/var/log/journal/` are very large (45GB+).
2. Check for deleted but open files: `lsof +L1`. Result: Several log files were deleted but are still held open by the `rsyslog` process.
3. Identify top growing files: `find / -type f -size +1G`. Result: A custom application log file `/opt/app/data.log` has grown to 30GB.

## Root Cause
The `logrotate` configuration for the custom application was misconfigured, leading to logs not being rotated or compressed correctly. Simultaneously, `journald` logs were not capped, consuming all remaining free space.

## Resolution
1. Immediately clear old journal logs to regain emergency space:
   ```bash
   journalctl --vacuum-time=1h
   ```
2. Truncate the massive application log (Only if not critical):
   ```bash
   truncate -s 0 /opt/app/data.log
   ```
3. Restart `rsyslog` to release deleted file handles:
   ```bash
   systemctl restart rsyslog
   ```

## Validation
- Re-run `df -h` -> `/` should show available space.
- Verify application can write logs again: `touch /opt/app/test_write`.
- Check app health: `systemctl status application`.

## Prevention
- Correct the `/etc/logrotate.d/application` configuration with `maxsize` and `rotate` limits.
- Set `SystemMaxUse=500M` in `/etc/systemd/journald.conf` to cap journal size.
- Set up CloudWatch Alarms for EBS volume usage > 80%.
- Implement a centralized logging solution (CloudWatch Logs, ELK) to reduce local storage reliance.