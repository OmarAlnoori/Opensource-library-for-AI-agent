# 002 — n8n binaryDataMode: database Trap

> **Status:** Complete  
> **Stack:** n8n v2.15 self-hosted · Coolify · Code nodes · HTTP Request node  
> **Context:** DMS media generation pipeline (Layers 4 AR/EN) at Dot.ID

---

## The Goal

Read a binary asset (logo image) inside a Code node, combine it with a generated
AI image, and upload the result to WordPress. The images arrive from upstream HTTP
Request nodes as binary items in the n8n workflow.

---

## What We Tried

### Attempt 1 — Read binary via `item.binary.X.data`

**Why we tried it:** This is what the n8n docs and every tutorial show. Binary
data on an item is accessed via `item.binary.fieldName.data`, which is documented
as a base64 string.

**How we set it up:**

```js
// Code node — appeared to work
const items = $input.all();
const logoBase64 = items[0].binary.logo.data;
// ... composite image using logoBase64
```

**What happened:** No error was thrown. The workflow ran successfully. But the
downstream image upload produced a corrupted or zero-byte file — downloads were
~6 bytes instead of the expected ~200KB.

We spent multiple versions (v1.6.1 through v1.6.8 of the AR Media Generator)
adding logging, re-checking MIME types, and trying different compositing
approaches before isolating the real cause.

The root cause: our n8n instance runs with `binaryDataMode: database`
(visible in execution error stack traces). In this mode, `item.binary.X.data`
is **not** the actual base64 — it is a short opaque reference string (~8
characters) that points to a database record. Treating it as base64 silently
produces garbage bytes.

**Verdict:** ❌ Failed (silent — no error, corrupt output)

---

### Attempt 2 — `this.helpers.getBinaryDataBuffer()`

**Why we tried it:** After reading the n8n source and stack traces closely, we
found the `helpers` API documented for Community nodes, which includes
`getBinaryDataBuffer` — a method that resolves the database reference and
returns real bytes.

**How we set it up:**

```js
// Code node — correct approach for binaryDataMode: database
const buffer = await this.helpers.getBinaryDataBuffer(0, 'logo');
const base64 = buffer.toString('base64');
// buffer is a real Node.js Buffer with the actual image bytes
```

The first argument (`0`) is the input item index; the second is the binary
field name as it appears in `item.binary`.

**What happened:** Returned the actual image buffer. Downstream compositing
and upload worked correctly on the first try.

**Verdict:** ✅ Worked

---

## The Second Trap: HTTP Request Replaces Binary Collection

While fixing the above, we hit a related problem: a workflow that downloads
two assets in series (logo then template image) lost the first asset after the
second HTTP Request node ran.

The cause: n8n's HTTP Request node with `responseFormat: file` **replaces**
the entire binary collection on its output item. Input binaries don't pass
through. So after the second download, the item had only `template` — `logo`
was gone.

Combined with `binaryDataMode: database`, the only safe pattern is to extract
each binary to JSON (base64 string) immediately after it lands, before any
downstream HTTP Request node runs:

```js
// "Encode Logo Binary" — thin Code node immediately after logo download
const buffer = await this.helpers.getBinaryDataBuffer(0, 'logo');
return [{ json: { logo_b64: buffer.toString('base64') } }];
```

Then propagate `logo_b64` as a plain JSON field for the rest of the workflow.
Never rely on binary passthrough across HTTP Request nodes.

---

## What We Shipped

The pattern that landed in `dms-layer4-AR-media-generator v1.6.9`:

```
[Download Logo HTTP Request]
    → [Encode Logo Binary AR1]       ← extracts to JSON immediately
[Download Template HTTP Request]
    → [Encode Template Binary AR1]   ← same pattern
[Composite Code Node]
    ← reads logo_b64 + template_b64 from JSON, not from binary
    → [Upload to WordPress]
```

Each asset gets a dedicated "encode" Code node immediately after its download,
before any other node can touch the binary collection.

---

## Key Takeaway

In n8n with `binaryDataMode: database`, `item.binary.X.data` is a database
reference, not base64. Use `this.helpers.getBinaryDataBuffer(index, 'fieldName')`
to get real bytes. And always extract binaries to JSON immediately after
download — HTTP Request nodes silently wipe the binary collection on their output.

---

## Related

- [`tools/n8n-ce.md`](../tools/n8n-ce.md)
- [`lessons/n8n-binary-data.md`](../lessons/n8n-binary-data.md)
