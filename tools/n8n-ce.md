# n8n Community Edition (self-hosted)

**Verdict:** ✅ Worked — with several non-obvious gotchas documented below

**Version tested:** n8n 2.15.0, self-hosted via Coolify on Ubuntu VPS

---

## What It Is

n8n is an open-source workflow automation platform. The Community Edition (CE)
is free and self-hostable. It covers most automation needs but has meaningful
differences from the Cloud and Enterprise editions — differences that are easy
to miss and hard to debug when you hit them.

---

## Gotcha 1: Webhook Node typeVersion 2 Wraps the Body

**Symptom:** `$json.field_name` returns `undefined` even though the request body
clearly contains the field.

**Cause:** Webhook node typeVersion 2 (the default in newer n8n) wraps the entire
request as:

```json
{
  "body": { "your": "fields" },
  "headers": { ... },
  "params": { ... },
  "query": { ... }
}
```

Every downstream reference must use `$json.body.field_name`, not `$json.field_name`.

**Fix:**

```js
// Wrong (typeVersion 2)
const value = $json.field_name;

// Correct (typeVersion 2)
const value = $json.body.field_name;

// Or in expressions:
{{ $json.body.field_name }}
{{ $('Webhook').item.json.body.field_name }}
```

typeVersion 1 outputs the body flat — but don't downgrade an existing workflow
that already uses the wrapped shape, or all downstream references break.

---

## Gotcha 2: No Variables in CE 2.15 (`$vars` is Enterprise-only)

**Symptom:** `$vars.MY_VAR` returns undefined or throws. The left sidebar has
no "Variables" section. Setting `N8N_VAR_*` environment variables in Coolify
has no effect.

**Cause:** Workflow Variables (`$vars.X`) are an Enterprise feature. They are
not available in Community Edition regardless of env vars set.

**Fix:** Two alternatives depending on sensitivity:

```js
// For non-sensitive config — hardcode in the workflow JSON
const baseUrl = 'https://yoursite.com';

// For sensitive config — set a raw env var on the n8n container in Coolify
// N8N_MY_SECRET=value
// then in a Code node:
const secret = $env.N8N_MY_SECRET;
```

---

## Gotcha 3: `binaryDataMode: database` — `.data` Is Not Base64

**Symptom:** Code node reads `item.binary.fieldName.data`, treats it as base64,
produces a ~6-byte corrupted file with no error thrown.

**Cause:** When n8n runs with `binaryDataMode: database` (common in self-hosted
setups), `item.binary.X.data` is an opaque reference string (~8 chars), not the
actual base64. It looks like base64 but decodes to garbage.

**Fix:**

```js
// Wrong — returns a reference string in database mode
const b64 = item.binary.logo.data;

// Correct — resolves the reference to real bytes
const buffer = await this.helpers.getBinaryDataBuffer(0, 'logo');
const b64 = buffer.toString('base64');
```

The helper only works on the current node's input item — not on
`$('OtherNode').first()` references. See
[Case Study 002](../case-studies/002-n8n-binary-data-mode.md) for the full story.

---

## Gotcha 4: HTTP Request Node Replaces the Binary Collection

**Symptom:** A binary field that existed before an HTTP Request node is gone
after it runs. Downloading two files in series results in only the second one
surviving.

**Cause:** HTTP Request node with `responseFormat: file` replaces the entire
`binary` collection on its output item. Input binaries are not passed through.

**Fix:** Extract any binary you need to keep into a JSON field (base64 string)
before the next HTTP Request node runs:

```js
// "Encode Asset" Code node — immediately after the first HTTP Request
const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');
return [{ json: { asset_b64: buffer.toString('base64') } }];
```

Then propagate `asset_b64` as plain JSON. The second HTTP Request node can now
replace the binary collection without losing the first asset.

---

## Gotcha 5: Postgres Table Names Cannot Start with a Digit

**Symptom:** `ERROR: trailing junk after numeric literal` when the Chat Memory
node or a PGVector operation references a table.

**Cause:** Postgres parses a leading digit as a numeric literal, not as an
identifier. A table name like `20_vectors` fails.

**Fix:** Prefix with a letter:

```
# Wrong
20_vectors
20u_vectors

# Correct
u20_vectors
acct20_vectors
```

Applies to: Chat Memory `tableName`, PGVector table names, and any
`knowledge_namespace` derived from a numeric account ID.

---

## Gotcha 6: IF/Switch Strict Type Validation

**Symptom:** A condition like `status == true` always evaluates to false even
when `status` is `1`.

**Cause:** Default `typeValidation: "strict"` in IF and Switch nodes rejects
comparisons across types (`1` is not `true`). PHP backends often encode booleans
as integers in JSON.

**Fix:** Set `typeValidation: "loose"` on the condition, or cast explicitly in
a Code node before the IF node:

```js
// Normalize PHP-style booleans before IF node
return [{ json: { ...item.json, is_active: Boolean(item.json.is_active) } }];
```

---

## Deployment Notes (Coolify + Ubuntu)

- Set `N8N_BINARY_DATA_MODE=filesystem` if you want binaries on disk rather than
  in the database (avoids Gotcha 3 entirely, but requires a persistent volume).
- The default `database` mode works fine as long as you always use
  `helpers.getBinaryDataBuffer`.
- Expose the webhook port (default 5678) and set `N8N_HOST` + `WEBHOOK_URL` env
  vars in Coolify to get correct webhook URLs.
- Community Edition does not support SAML/SSO or Workflow History beyond 3 days.

---

## See Also

- [Case Study 002 — n8n Binary Data Mode Trap](../case-studies/002-n8n-binary-data-mode.md)
- [Lesson: n8n Binary Data Gotchas](../lessons/n8n-binary-data.md)
- [n8n CE vs Enterprise feature comparison](https://n8n.io/pricing/)
