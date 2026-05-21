# Examples

Curl examples and sample payloads for every endpoint in `hush-api.yaml`.

Currently a placeholder. Examples will be added in **Phase 1** (see [`../PLAN.md`](../PLAN.md)). One file per endpoint, named `<operationId>.md`, containing:

1. A short description of when the call is made.
2. A request example (curl + raw HTTP).
3. A success-response example.
4. The most common error-response example.

Example skeleton:

```markdown
# registerDevice

Called on first boot, after WiFi credentials are provisioned.

## Request

\`\`\`bash
curl -X POST https://api.open-hush.com/v1/device/register \
  -H "Authorization: HMAC keyId=...,signature=...,ts=..." \
  -H "Content-Type: application/json" \
  -d '{"serial":"HUSH-2026-0001","firmwareVersion":"0.1.0"}'
\`\`\`

## Success — 200

\`\`\`json
{ "device": { "id": "...", "serial": "HUSH-2026-0001", "state": "unclaimed" } }
\`\`\`
```
