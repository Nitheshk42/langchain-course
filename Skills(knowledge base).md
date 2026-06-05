# Webhook Implementation Skill

## Overview

This skill enables Claude (or any LLM) to understand, explain, and recreate a production-ready webhook system for secure inter-application communication using HMAC-SHA256 signature verification.

**Use this skill when:**
- Building webhook systems from scratch
- Implementing event-driven architecture
- Adding webhook security (signature verification)
- Explaining webhook concepts to others
- Debugging webhook authentication issues
- Deploying webhooks to production

---

## Core Concepts

### What is a Webhook?

A webhook is an HTTP callback—a way for one application (sender) to push real-time notifications to another application (receiver) when something happens.

**Key characteristics:**
- Event-driven (not polling)
- Real-time delivery
- Requires public URL on receiver
- Must include authentication (signatures)

### HMAC-SHA256 Authentication

The security mechanism that proves webhook authenticity:

```
Secret (known to both) + Event Data → HMAC Algorithm → Signature
Both apps verify: Does received_signature == calculated_signature?
```

**Why HMAC?**
- Symmetric (both apps use same secret)
- One-way (can't reverse to get secret)
- Tamper-proof (any change invalidates signature)
- Efficient (no public-key cryptography needed)

---

## Architecture Pattern

### Components

| Component | Role | Technology |
|-----------|------|-----------|
| **App A (Receiver)** | Listens for webhooks, verifies signatures | Flask (Python) |
| **App B (Sender)** | Creates events, generates signatures, sends webhooks | Flask (Python) |
| **Shared Secret** | Pre-shared authentication key | Environment variable |
| **Transport** | HTTP POST with signature header | HTTPS (production) |
| **Deployment** | Cloud hosting for public URL | Railway, Heroku, etc |

### Event Flow

```
1. Event triggers in App B (user input, database change, etc)
2. App B creates JSON payload
3. App B calculates HMAC-SHA256(SECRET, payload)
4. App B sends: POST /webhook + X-Signature header
5. App A receives request
6. App A calculates HMAC-SHA256(SECRET, received_payload)
7. App A compares signatures:
   ✅ Match → Process event (200 OK)
   ❌ No match → Reject (401 Unauthorized)
```

---

## Implementation Patterns

### Receiver (App A) Pattern

**Architectural Requirements:**

The webhook receiver must:
1. **Accept HTTP POST requests** on a publicly accessible endpoint
2. **Extract the signature** from the `X-Signature` header
3. **Retrieve the request body** without parsing (maintain raw bytes for signature verification)
4. **Calculate expected signature** using HMAC-SHA256 algorithm with the pre-shared SECRET
5. **Compare signatures** using constant-time comparison (prevents timing attacks)
6. **Return 200 OK** if signatures match (event is authentic)
7. **Return 401 Unauthorized** if signatures don't match (reject forged requests)
8. **Process authenticated events** asynchronously (don't block the HTTP response)
9. **Log all verification attempts** including failures, timestamps, and source IPs

**Key Implementation Considerations:**
- Must handle raw request body before JSON parsing
- Signature comparison must be constant-time (not early-exit)
- Should implement request timeouts (5-10 seconds)
- Must not expose implementation details in error responses
- Should queue events for async processing (don't process in HTTP handler)
- Must maintain audit logs of all signature failures

### Sender (App B) Pattern

**Architectural Requirements:**

The webhook sender must:
1. **Create event payload** in JSON format with relevant data
2. **Serialize to string** before cryptographic processing
3. **Generate HMAC-SHA256 signature** using event data + pre-shared SECRET
4. **Convert signature to hexadecimal** format for transmission
5. **Send HTTP POST request** with:
   - JSON event in request body
   - Signature in `X-Signature` header
   - Appropriate `Content-Type` header
6. **Implement connection handling** with timeouts and retries
7. **Log all webhook sends** including payload, signature, and response status
8. **Handle delivery failures** gracefully (exponential backoff, max retries)
9. **Avoid hardcoding secrets** in any form

**Key Implementation Considerations:**
- Must use HTTPS in production (no HTTP)
- Should implement exponential backoff for retries (1s, 2s, 4s, 8s, 16s)
- Must not expose secrets in logs or error messages
- Should track webhook delivery status for monitoring
- Must handle network timeouts gracefully
- Should implement idempotency keys for safe retries

---

## Security Model

### Threat Mitigations

| Threat | Attack Vector | Mitigation |
|--------|----------------|-----------|
| **Unauthorized webhooks** | Attacker sends to public URL | Signature verification fails |
| **Payload tampering** | Attacker modifies event data | Hash changes, detected |
| **Replay attacks** | Send same event multiple times | Add timestamp validation (future) |
| **Man-in-the-middle** | Intercept & modify request | HTTPS + signature verification |
| **Secret exposure** | Git history, logs | Store in environment variables |

### Best Practices

**DO:**
- ✅ Store SECRET in environment variables
- ✅ Use HTTPS in production
- ✅ Log all verification failures
- ✅ Implement request timeouts
- ✅ Add rate limiting
- ✅ Rotate secrets periodically

**DON'T:**
- ❌ Hardcode secrets in source code
- ❌ Log request bodies containing sensitive data
- ❌ Trust the URL alone (always verify signature)
- ❌ Process webhooks synchronously (use queue)
- ❌ Return detailed error messages (reveals implementation)

---

## Testing Patterns

### Unit Test Pattern: Valid Signature Verification

**Test Objective:** Verify that the receiver correctly accepts webhooks with valid signatures.

**Prerequisites:**
- Test SECRET key known to both sender and receiver
- Sample event payload in JSON format
- Method to generate HMAC-SHA256 signatures

**Test Steps:**
1. Create event payload with known content
2. Generate expected signature using HMAC-SHA256(SECRET, payload)
3. Send HTTP POST request with signature in X-Signature header
4. Assert response status is 200 OK
5. Verify event was processed (check database, logs, or queue)

**Success Criteria:**
- Server accepts request without errors
- Event is marked as processed
- No security warnings in logs
- Response includes confirmation of receipt

### Unit Test Pattern: Invalid Signature Rejection

**Test Objective:** Verify that the receiver rejects webhooks with invalid/missing signatures.

**Test Scenarios:**
1. Missing X-Signature header entirely
2. X-Signature with incorrect hash value
3. X-Signature with incomplete hash
4. Payload modified after signature generation

**Test Steps (for each scenario):**
1. Prepare event payload
2. Send HTTP POST request with invalid/missing signature
3. Assert response status is 401 Unauthorized
4. Verify event was NOT processed (not in database/queue)
5. Verify failure was logged

**Success Criteria:**
- Server rejects all invalid signatures consistently
- No event processing occurs
- Security failure is properly logged
- Response is generic (doesn't leak implementation details)

### Integration Test Pattern: End-to-End Webhook Flow

**Test Objective:** Verify complete webhook flow from sender through receiver.

**Test Setup:**
1. Deploy both sender and receiver applications
2. Configure both with same SECRET
3. Establish connectivity between components

**Test Execution:**
1. Trigger event in sender application
2. Sender generates signature and sends webhook
3. Receiver accepts request
4. Receiver verifies signature
5. Receiver processes event
6. Verify end-to-end result (email sent, database updated, etc.)

**Success Criteria:**
- Event successfully delivered from sender to receiver
- Signature verification passes
- Event processing completes without errors
- Results visible in downstream systems
- Execution time within SLA

### Load Testing Pattern

**Test Objective:** Verify webhook system can handle production traffic volumes.

**Test Scenarios:**
- Normal load: 100 webhooks per minute
- Peak load: 1000 webhooks per minute
- Sustained load: Maintain peak for 1 hour

**Metrics to Monitor:**
- Response time (p50, p95, p99)
- Error rate (5xx, 401s, timeouts)
- CPU and memory utilization
- Database connection pool status
- Queue depth if asynchronous

---

## Deployment Patterns

### Local Development

```bash
# Terminal 1 - Receiver
python app_a.py  # Runs on http://localhost:3000

# Terminal 2 - Sender
python app_b.py  # Sends to http://localhost:3000
```

### Production Deployment (Railway)

1. **Prepare files:**
   - `app_a.py` (receiver)
   - `requirements.txt` with dependencies
   - `Procfile` with `web: gunicorn app_a:app`

2. **Deploy:**
   ```bash
   git push  # Railway auto-detects Procfile
   ```

3. **Configure environment:**
   - Railway Dashboard → Settings → Variables
   - Add: `WEBHOOK_SECRET=<production-key>`

4. **Update sender:**
   ```python
   url = "https://app-name.up.railway.app/webhook"
   ```

---

## Common Patterns

### Pattern 1: Dynamic Event Generation

**Architectural Pattern:** Event Factory Pattern

**Overview:**
Rather than hardcoding individual webhook payloads, implement a generalized event generation mechanism that supports multiple event types.

**Components:**
- Event registry (defines all possible event types)
- Event builder (constructs events from domain logic)
- Signature generator (consistent signing across all events)
- Dispatcher (routes events to appropriate webhooks)

**Implementation Considerations:**
- Event versioning (handle schema changes over time)
- Event validation (ensure required fields before sending)
- Event enrichment (add metadata: timestamp, request ID, source)
- Event deduplication (prevent duplicate sends)

**Benefits:**
- Scales to many event types without code changes
- Consistent signature generation
- Auditable event history
- Easy to add new webhook consumers

### Pattern 2: Resilient Delivery with Exponential Backoff

**Architectural Pattern:** Retry with Exponential Backoff

**Overview:**
When webhook delivery fails (network error, receiver down), automatically retry with increasing delays to avoid overwhelming the receiver.

**Key Requirements:**
- Track delivery attempts (attempt count, timestamp)
- Calculate backoff duration (exponential: 1s, 2s, 4s, 8s, 16s)
- Maximum retry limit (typically 5 attempts)
- Dead letter queue for permanently failed deliveries
- Monitoring and alerting on failed deliveries

**Implementation Considerations:**
- Only retry on transient failures (connection timeout, 5xx)
- Don't retry on client errors (401, 400, 403)
- Track which events are in retry queue
- Implement jitter (add randomness to prevent thundering herd)
- Maintain retry history for debugging

**Benefits:**
- Handles temporary receiver outages
- Reduces load on receiver during recovery
- Completes successful deliveries automatically
- Tracks permanently failed events for investigation

### Pattern 3: Idempotent Event Processing

**Architectural Pattern:** Request ID + Deduplication

**Overview:**
Prevent duplicate processing when the same webhook is delivered multiple times (due to retries or network issues).

**Key Requirements:**
- Sender includes unique Request-ID header
- Receiver stores processed Request-IDs
- Check if Request-ID already processed before accepting
- Return success (200) for duplicate attempts (idempotent)
- Clean up old Request-IDs after retention period (e.g., 30 days)

**Implementation Considerations:**
- Request-ID format (UUID recommended)
- Storage mechanism for processed IDs (cache, database)
- TTL/cleanup strategy for old IDs
- Logging to track duplicate attempts
- Monitoring on deduplication rate

**Benefits:**
- Safe retries (can retry without side effects)
- Handles receiver restarts gracefully
- Reduces downstream duplicate events
- Improves reliability in failure scenarios

---

## Real-World Examples

### GitHub Webhooks

When code is pushed:
```
1. Developer: git push
2. GitHub: Sends webhook to your CI/CD URL
3. Your CI Server: Receives verified webhook
4. Runs: Tests, build, deploy
```

### Stripe Payment Webhooks

When payment succeeds:
```
1. Customer: Makes payment
2. Stripe: Sends webhook to your app
3. Your App: Verifies signature (payment is real!)
4. Updates: Order status, inventory, sends receipt
```

### AWS SNS Webhooks

When infrastructure event happens:
```
1. AWS: Detects server down
2. AWS: Sends webhook to your monitoring app
3. Your App: Verifies (trusted source)
4. Alerts: PagerDuty, Slack, email
```

---

## Troubleshooting

### 401 Unauthorized (Signature Mismatch)

**Causes:**
- SECRET doesn't match between apps
- Payload was modified in transit
- Request body encoding issue (raw vs JSON)

**Debug:**
```python
print(f"Received signature: {signature}")
print(f"Received payload: {payload}")
print(f"Calculated signature: {expected}")
```

### Connection Refused

**Causes:**
- Receiver not running
- Wrong port/URL
- Firewall blocking

**Fix:**
```bash
# Verify receiver is running
netstat -an | grep 3000

# Test connection
curl http://localhost:3000/webhook
```

### Slow Response

**Causes:**
- Processing synchronously (should queue)
- Network latency
- Overloaded receiver

**Solution:**
- Return 200 immediately
- Process in background worker
- Implement load balancing

---

## Production Checklist

Before deploying webhooks to production:

- [ ] SECRET stored in environment variables
- [ ] HTTPS only (no HTTP)
- [ ] Signature verification implemented
- [ ] Error handling for edge cases
- [ ] Timeout configured (5-10 seconds)
- [ ] Rate limiting implemented
- [ ] Retry logic with exponential backoff
- [ ] Idempotency keys tracked
- [ ] Monitoring and alerting set up
- [ ] Security audit completed
- [ ] Documentation written
- [ ] Test coverage >80%

---

## References & Resources

- **HMAC Spec:** RFC 4868
- **Webhook Security:** GitHub docs on securing webhooks
- **Flask:** Official documentation
- **Railway:** Deployment guide
- **HTTP Status Codes:** 200 (OK), 401 (Unauthorized), 5xx (Error)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-06-04 | Initial skill creation |

---

## Skill Metadata

**Applicable to:**
- Building webhook systems
- Teaching event-driven architecture
- Implementing secure APIs
- Creating notification systems
- Building integrations between apps

**Assumes knowledge of:**
- Python basics
- HTTP/REST concepts
- JSON format
- Cryptography fundamentals (HMAC)

**Teaches:**
- Webhook architecture
- HMAC-SHA256 signatures
- Event-driven design
- Security best practices
- Production deployment

**Real-world value:**
- Every modern SaaS uses webhooks
- Understanding webhooks = understanding modern backend
- Skills directly applicable to production systems
