# High CPU Utilization (Hung Process)

## Context
Production API performance is degraded, with high latency and increased error rates.

## Symptoms
- CloudWatch CPU metrics show consistent 100% usage for multiple instances in the ASG.
- `top` or `htop` reveals a single process consuming all CPU.
- Slow response times reported by customers.

## Initial Triage
1. Identify offending process: `top -b -n 1 | head -n 20`. Result: `java` process is at 99.8% CPU.
2. Check thread activity: `top -H -p <pid>`. Result: Multiple threads are in high-activity states.
3. Inspect application throughput: `tail -f /var/log/app/access.log`. Result: High volume of normal traffic, but slower processing time per request.

## Investigation
1. Thread dump for analysis:
   ```bash
   jstack <pid> > thread_dump.txt
   ```
2. Process tracing: `strace -p <pid> -c`. Result: High number of system calls, particularly `futex` and `sched_yield`.
3. Resource limits check: `ulimit -a`. Result: Limits are within standard ranges.
4. Recent changes: Check the `git log` of the application. Result: A new regex validation was added 1 hour ago.

## Root Cause
A poorly optimized regular expression in the input validation layer was susceptible to "Regular Expression Denial of Service" (ReDoS), causing the CPU to hang during high-traffic periods with complex inputs.

## Resolution
1. Temporarily scale out the ASG to provide additional capacity and reduce the load per instance:
   ```bash
   aws autoscaling update-auto-scaling-group --auto-scaling-group-name api-asg --desired-capacity 6
   ```
2. Deploy a hotfix with the simplified regular expression.
3. Once the hotfix is verified, scale back to original capacity.

## Validation
- Monitor CPU: `top` should show the application process at expected baseline (e.g., 20-30%).
- Check latency: CloudWatch P99 response time returns to normal baseline (<200ms).

## Prevention
- Implement linting rules (e.g., `eslint-plugin-security`) to detect unsafe regex.
- Use regex timeout settings in the application code.
- Introduce load testing during CI to simulate high-traffic scenarios before deployment.
- Set up CPU alarms (>80% for 5 mins) to alert the engineering team immediately.