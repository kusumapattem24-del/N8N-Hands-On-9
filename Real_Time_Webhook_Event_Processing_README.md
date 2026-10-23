# Build & Validate a Real-Time Webhook Event Processing Workflow

## Assessment overview
This Topic 9 n8n assessment designs a POST webhook to receive user events, inspect the incoming body, headers, and query parameters, validate an `X-Webhook-Token` header, normalize authorized user data, and return HTTP 200 or HTTP 403 using dedicated Respond to Webhook nodes.

**Status:** Implementation guide and evidence template. Actual workflow execution, exported JSON, screenshots, public repository, and video must be supplied after testing. The assessment does not specify the literal required token value; choose a test token privately and do not commit it.

## Workflow architecture
```text
Webhook_NewUser (POST /new-user; Response: Using Respond to Webhook Node)
    |
IF_ValidWebhookToken (compare headers['x-webhook-token'] with protected expected token)
    | true                                  | false
EDIT_NormalizeUser                       RESPOND_Forbidden (403)
    |
RESPOND_Success (200)
```

## Node configuration

### 1. Webhook_NewUser
- Node: Webhook; HTTP Method: `POST`; Path: `new-user`.
- Respond: `Using 'Respond to Webhook' Node`.
- Read body fields via `{{$json.body.name}}` and `{{$json.body.email}}`.
- Read query parameters via `{{$json.query}}` and the token header via `{{$json.headers['x-webhook-token']}}` (n8n typically lowercases header names; inspect the actual output).
- For development, click **Listen for test event** and send a request to `/webhook-test/new-user`.

### 2. IF_ValidWebhookToken
- Condition: String equality, incoming header expression `{{$json.headers['x-webhook-token'] || ''}}` against the expected token.
- Use a protected environment variable or another securely managed server-side secret where supported; do not embed a production token in workflow JSON or screenshots. For example, on instances that permit environment expressions, `{{$env.WEBHOOK_TOKEN}}` can supply the expected value. Confirm instance access restrictions and supported configuration before use.
- True -> normalization; False -> forbidden response. A missing header must fail validation.

### 3. EDIT_NormalizeUser
Use Edit Fields (Set), keep only required output fields. Example expressions:
- `name` (String): `{{ String($json.body.name || '').trim().replace(/\s+/g, ' ').toLowerCase().replace(/\b\w/g, c => c.toUpperCase()) }}`
- `email` (String): `{{ String($json.body.email || '').trim().toLowerCase() }}`
- `receivedAt` (String): `{{ $now.toISO() }}`
- `status` (String): `accepted`
- Optional: add an IF node before normalization to reject malformed or missing name/email (400). The assessment explicitly requires 200/403; 400 is a suggested enhancement.

### 4. RESPOND_Success
- Node: Respond to Webhook.
- Respond With: JSON.
- Response Code: `200`.
- Example response body expression: `{{ { success: true, message: 'User event accepted', user: { name: $json.name, email: $json.email }, receivedAt: $json.receivedAt } }}`.

### 5. RESPOND_Forbidden
- Node: Respond to Webhook.
- Respond With: JSON.
- Response Code: `403`.
- Example body: `{ "success": false, "error": "Forbidden: invalid or missing webhook token" }`.
- Do not echo the submitted token in errors or logs.

## Example test requests
Replace `YOUR_N8N_HOST` with your actual n8n base URL. Supply `WEBHOOK_TOKEN` from a local environment variable; do not paste a real token into public documentation.

Authorized request (test URL):
```bash
curl -i -X POST 'https://YOUR_N8N_HOST/webhook-test/new-user?source=assessment' \
  -H 'Content-Type: application/json' \
  -H "X-Webhook-Token: ${WEBHOOK_TOKEN}" \
  -d '{"name":"  aLiCe   JOHNSON ","email":" ALICE@EXAMPLE.COM "}'
```
Expected: HTTP 200, normalized name `Alice Johnson`, email `alice@example.com`, timestamp.

Unauthorized request:
```bash
curl -i -X POST 'https://YOUR_N8N_HOST/webhook-test/new-user' \
  -H 'Content-Type: application/json' \
  -H 'X-Webhook-Token: invalid-test-token' \
  -d '{"name":"Alice","email":"alice@example.com"}'
```
Expected: HTTP 403, safe error message.

Missing-token request:
```bash
curl -i -X POST 'https://YOUR_N8N_HOST/webhook-test/new-user' \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice","email":"alice@example.com"}'
```
Expected: HTTP 403.

## Test versus production
- Test endpoint `/webhook-test/new-user` is registered while the editor listens for a test event and is intended for interactive development.
- Production endpoint `/webhook/new-user` is available after saving and activating/publishing the workflow, according to the n8n version. Use the production URL for external integrations.
- If a webhook is not registered, confirm the HTTP method, exact path, listening/active status, and correct base URL. Check proxy routing and whether another workflow uses the same method/path.

## Test matrix (record actual results)
| ID | Test | Expected | Actual | Evidence |
|---|---|---|---|---|
| T01 | Valid token and mixed-case email | 200; normalized profile | Pending | Screenshot pending |
| T02 | Invalid token | 403 | Pending | Screenshot pending |
| T03 | Missing token | 403 | Pending | Screenshot pending |
| T04 | Query parameter `source=assessment` | Query accessible in trigger | Pending | Screenshot pending |
| T05 | Test endpoint while listening | Webhook receives request | Pending | Screenshot pending |
| T06 | Production endpoint when active | Webhook receives request | Pending | Screenshot pending |
| T07 | Whitespace in name and email | Trimmed and standardized | Pending | Screenshot pending |

## Security review
- Use HTTPS; keep token secret and rotate it when exposed.
- Never publish real credentials, authorization headers, or raw sensitive payloads in GitHub/screenshots.
- A static token is a basic shared-secret check, not full message authenticity or replay protection. Consider HMAC signatures, timestamps, replay defense, IP controls, rate limiting, and payload validation for production.
- Do not assume headers are case-sensitive; inspect n8n's normalized incoming headers.
- Return generic 403 errors and avoid revealing expected tokens.

## Suggested repository layout
```text
real-time-webhook-event-processing/
  README.md
  workflow/
    real-time-webhook-event-processing.json
  docs/
    Real_Time_Webhook_Event_Processing_Documentation.pdf
  screenshots/
    webhook-trigger.png
    authorized-200.png
    unauthorized-403.png
    normalized-output.png
    production-test.png
  tests/
    curl-examples.md
    test-results.md
```

## Demonstration video outline (5-8 minutes)
1. Introduce event-driven webhooks and assessment objective.
2. Show Webhook POST path and response mode.
3. Demonstrate incoming body, headers, and query parameters (redact tokens).
4. Show IF authorization rule and both branches.
5. Show normalization expressions and successful 200 response.
6. Send invalid/missing token and show 403 response.
7. Explain test vs production endpoint behavior and troubleshooting.
8. Show workflow export, documentation, evidence, and public repository.

## LMS submission checklist
- [ ] Workflow configured and tested in n8n
- [ ] Authorized 200 and unauthorized 403 confirmed
- [ ] Test and production URLs verified
- [ ] Exported workflow JSON included
- [ ] Screenshots and actual test results included
- [ ] PDF documentation included
- [ ] Public GitHub repository accessible
- [ ] Loom/YouTube link accessible
- [ ] Description includes challenges, fixes, assumptions, and limitations

## Assumptions and limitations
This document is a proposed implementation based on the supplied assessment, not proof of successful execution. The token value, instance settings, final screenshots, and GitHub/video URLs are not provided. Actual results must be captured before submission.
