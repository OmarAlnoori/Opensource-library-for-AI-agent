# Lesson: n8n Binary Data Doesn't Travel the Way You Think

**Applies to:** Any n8n self-hosted workflow that reads, passes, or transforms
binary data (images, videos, files) inside Code nodes.

---

## The Two Traps

### Trap 1 — `item.binary.X.data` is not base64 in database mode

When n8n runs with `binaryDataMode: database` (common in self-hosted / Coolify
deployments), the `.data` property on a binary field is an opaque reference
string (~8 characters), not actual base64.

This fails silently: no error is thrown, but the decoded bytes are garbage.

```js
// WRONG — looks right, produces garbage in database mode
const b64 = item.binary.logo.data;
const buffer = Buffer.from(b64, 'base64'); // ~6 bytes of garbage
```

```js
// CORRECT — resolves the reference to real bytes
const buffer = await this.helpers.getBinaryDataBuffer(0, 'logo');
const b64 = buffer.toString('base64'); // actual image bytes
```

Check whether you're in database mode: look for `binaryDataMode` in n8n stack
traces or your Coolify environment variables.

### Trap 2 — HTTP Request node wipes the binary collection

An HTTP Request node with `responseFormat: file` replaces the entire `binary`
collection on its output item. Binaries from upstream don't pass through.

If your workflow downloads two files in series:

```
[Download logo]   → binary.logo = ✅
[Download image]  → binary.logo = ❌ gone, binary.data = ✅ (only the new one)
```

---

## The Safe Pattern

Extract every binary to JSON immediately after it arrives, before any downstream
HTTP Request node or Code node that might not pass it through.

```
[HTTP Request — Download Logo]
    ↓
[Code — Encode Logo]          ← extract to JSON here
    return [{ json: { logo_b64: buffer.toString('base64') } }]
[HTTP Request — Download Template]
    ↓
[Code — Encode Template]      ← same pattern
[Code — Composite / Process]  ← reads logo_b64 + template_b64 from JSON
[HTTP Request — Upload]
```

Key rules:
1. Call `this.helpers.getBinaryDataBuffer(itemIndex, 'fieldName')` — not `.data`.
2. Extract to JSON **at the node where the binary lands** — not several nodes later.
3. If you need binary from two upstream paths, use the "extract to JSON first"
   pattern on each path independently.
4. `helpers.getBinaryDataBuffer` only works on the **current node's input item**,
   not on `$('OtherNode').first()` references across the graph.

---

## How to Check Your n8n Mode

Open any failed execution, expand the error details, and look for a stack trace.
If you see `binaryDataMode: database` or `filesystem`, that tells you which mode
is active. Alternatively, check the `N8N_BINARY_DATA_MODE` env var in your
Coolify or Docker configuration.

To avoid the trap entirely: set `N8N_BINARY_DATA_MODE=filesystem` and mount a
persistent volume. Binaries are then stored as files on disk, and `.data` contains
actual base64. This trades the database mode quirk for a volume management
requirement.

---

## Related

- [Case Study 002 — n8n binaryDataMode Trap (full debug story)](../case-studies/002-n8n-binary-data-mode.md)
- [Tool: n8n Community Edition](../tools/n8n-ce.md)
