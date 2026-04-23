# API Requests Timing Out (Upstream Delay)

## Context
External clients are experiencing HTTP 504 (Gateway Timeout) errors when calling the `/v1/process-data` endpoint.

## Symptoms
- Clients report connection timeouts after 30 seconds.
- Load Balancer (ALB) metrics show an increase in `HTTPCode_ELB_504_Count`.
- Backend logs show many requests reaching the handler but not completing.

## Initial Triage
1. Check backend logs: `grep "process-data" /var/log/app/access.log | tail -n 50`. Result: `processing_time=30.001s`.
2. Check internal metrics for the process: Backend is waiting for a response from an upstream third-party service (Payment Gateway).
3. Test connectivity to upstream: `curl -I https://api.thirdparty.com/v1/ping`. Result: `HTTP/1.1 200 OK`, but latency is >10s.

## Investigation
1. Log analysis: Trace IDs reveal the process flow hangs at the `POST /v1/process-payment` call to the upstream provider.
2. Verify timeout settings:
   - Load Balancer idle timeout: 60s (Sufficient).
   - Application-level HTTP client timeout: 30s (Matches the symptom).
3. Check for external provider status: Visit `status.thirdparty.com`. Result: "Investigating slow response times in the US-EAST region".

## Root Cause
A regional service degradation in the third-party Payment Gateway provider is causing long processing times (>30s), which exceeds the application's internal HTTP client timeout, leading to 504 errors being propagated back to the client.

## Resolution
1. Temporarily increase the internal application timeout to 45s to allow for slower processing:
   ```bash
   sed -i 's/UPSTREAM_TIMEOUT=30/UPSTREAM_TIMEOUT=45/' /etc/app/config.env
   systemctl restart application
   ```
2. Implement an immediate "Circuit Breaker" to fail-fast if the third-party continues to degrade, preventing resource exhaustion on the backend instances.
3. Communicate the external provider's status to clients via the public status page.

## Validation
- Monitor ELB metrics: 504 count decreases; latency increases but remains under 45s.
- Manual test: `curl -w "%{time_total}\n" -I https://api.corp.com/v1/process-data` -> `38.450s`. Success.

## Prevention
- Implement the "Circuit Breaker" pattern (e.g., using Hystrix or Resilience4j) to handle upstream failures gracefully.
- Add asynchronous processing (Queue-based) for non-immediate operations to avoid blocking synchronous API calls.
- Monitor upstream latency specifically and set up alerts for when P99 exceeds baseline (>2s).
- Maintain multi-region or secondary provider availability for critical upstream dependencies.