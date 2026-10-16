# TQNN DMM — Developer Integration Guide

**Toridion Ltd · API v1.1.5**

> **For coding agents and the developers working with them.** Give this guide to Claude or any LLM coding assistant, together with the `dmm-mcp-server` repository and its README, and say: *"Using this guide and the dmm-mcp-server README, build a [Python/JavaScript/PHP] script that does X with TQNN DMM."* This guide covers the REST API and client-side encoding. The MCP README covers the MCP tools, configuration and resolvers.

---

## What is TQNN DMM?

TQNN DMM (Document Meta Memory) is a sovereign AI memory appliance. It stores JSON metadata as a neural associative memory graph and enables probabilistic O(1) search across all stored associations — without SQL, schemas, or indexes.

**The mental model:**
- You **store** a document by sending its JSON metadata (plain, or hashed client-side) with a file reference you choose.
- You **search** by sending one query term, encoded the same way as the stored data, and asking the DMM which documents contain it.
- The DMM returns a list of **file references** — the identifiers you assigned at store time.
- The appliance stores tokens and file references, not documents. Your application (or the MCP resolver layer) turns a file reference back into content.

**Key facts for developers:**
- All API calls are form-encoded POST requests (`application/x-www-form-urlencoded` or `multipart/form-data`).
- Authentication is two fields on every call: `tqnnAPIKEY` and `tqnnAPISECRET`.
- Endpoints: `storeDoc`, `searchDoc`, `removeDoc`, `discoverDatasets`, `storePattern`, `searchPattern`, `authID`, `registerID`.
- For most applications you will use `storeDoc`, `searchDoc` and `removeDoc`.
- The `pattern` field is a JSON string of document metadata. It can be plain JSON, or hashed client-side (PQR, optionally HMAC-keyed). See [Protection Tiers](#protection-tiers).
- **Hashing is optional and entirely client-side.** The appliance accepts plain, PQR-hashed and HMAC-keyed patterns identically and has no setting for which was used. Store and search must use the same method — see [Protection Tiers](#protection-tiers).
- Matching is **case-sensitive**: `Gupta` and `gupta` are different tokens.

---

## Base URL and Credentials

```
Base URL:   https://<your-appliance-host>
            e.g. https://tqnn.local  (local appliance with self-signed cert)
            e.g. https://dmm.yourcompany.com  (hosted)
```

Credentials come as a single combined token. Split it at character position 37:

```python
combined = "TNNS<37-char-key><secret>"
api_key    = combined[:37]   # first 37 characters
api_secret = combined[37:]   # everything after
```

Contact `info@toridion.com` to request API keys.

---

## API Endpoints — Complete Reference

### POST /v1/authID

Authenticates a username/password credential pair against the TQNN graph.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `username` | yes | Username to authenticate |
| `password` | yes | Password to authenticate |
| `credential2`–`credential9` | no | Additional credential factors. Supply to require them at auth time |
| `multihash` | no | Integer. Enable multi-hash of credential pair for higher security |
| `returnauthtoken` | no | Integer. Set to `1` to receive an authentication token in the response |
| `dataset` | no | Target dataset/bucket name |

**Use case:** Storing and validating user login credentials inside the TQNN graph. The credential pair is hashed before storage so raw passwords are never held.

---

### POST /v1/registerID

Registers a new credential set (up to 10 credential fields) in the TQNN graph.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `credential0` | yes | First credential value |
| `credential1` | yes | Second credential value |
| `credential2`–`credential9` | no | Additional credential values |
| `multihash` | no | Integer. Enable multi-hash for higher security |
| `returnauthtoken` | no | Integer. Set to `1` to receive an auth token |
| `dataset` | no | Target dataset/bucket name |

**Use case:** Registering multi-factor identity patterns. Each credential field is hashed independently into the graph.

---

### POST /v1/storePattern

Stores a raw pattern string directly into the TQNN graph. Low-level — no document metadata structure required.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `pattern` | yes | A pattern string to store |
| `dataset` | no | Target dataset/bucket name |

**Use case:** Storing arbitrary patterns when you do not need document-level file references. For document ingestion use `storeDoc` instead.

---

### POST /v1/searchPattern

Searches for a raw pattern string. Low-level counterpart to `storePattern`.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `pattern` | yes | A pattern string to search for |
| `return_filelist` | no | Set to `1` to include matched file references in the response |
| `dataset` | no | Target dataset/bucket name |

---

### POST /v1/storeDoc

Indexes a document's metadata into the TQNN graph, associated with a file reference you provide. This is the primary ingestion endpoint.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `filereference` | yes | Your stable identifier for this document (see fileReference design below) |
| `pattern` | yes | JSON string of document metadata, plain or hashed (see [Protection Tiers](#protection-tiers)) |
| `dataset` | no | Target dataset/bucket name (default: `demoapp`) |
| `user_request_id` | no | Caller-supplied request identifier, echoed back in the response |
| `create_ots` | no | Set to `1` to fingerprint the stored document (SHA-256) and submit it to the OpenTimestamps Bitcoin calendar pool. Response includes `ots_finger` |

**Success response:**
```json
{ "tqnn_response": "STORE_OK", "function": "storeDoc" }
```

**Auth failure:**
```json
{ "tqnn_response": "AUTH_FAILED" }
```

---

### POST /v1/searchDoc

Searches the TQNN graph for documents matching a hashed pattern. Returns file references for all matching documents.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `pattern` | yes | Search token, encoded the same way as the stored data (plain word, PQR or HMAC token) — **one token per call** (see Search section) |
| `return_filelist` | no | Set to `1` to include matched file references in the response |
| `dataset` | no | Target dataset/bucket name |

**Match response:**
```json
{
  "function": "searchDoc",
  "request_id": 1780318542,
  "time_used": 0.086,
  "billing_units": 1,
  "matches": 1,
  "misses": 0,
  "fc": 58,
  "filelist": "records_0001.jsonl::line149::REC-00010149::fpd_ef99e90b::1780314679\nrecords_0001.jsonl::line302::...",
  "energy": {
    "energy_usage_kWh": "0.0000001677",
    "carbon_emissions_mg": "0.0390796365",
    "equivalent_meters_driven": "0.0003256636"
  },
  "tqnn_response": "MATCH"
}
```

**No match response:**
```json
{ "tqnn_response": "NO_MATCH", "fc": 0, "matches": 0, "misses": 1 }
```

**Response fields:**

| Field | Description |
|-------|-------------|
| `tqnn_response` | `MATCH`, `NO_MATCH`, `STORE_OK`, or `AUTH_FAILED` |
| `fc` | File count — number of matched file references |
| `filelist` | Newline-separated file references. Each line: `<your_ref>::<tqnn_timestamp>` |
| `matches` | Number of pattern tokens that matched |
| `misses` | Number of pattern tokens that did not match |
| `billing_units` | Units consumed (1 per call) |
| `energy` | Energy usage metadata for this query |

---

### POST /v1/removeDoc

Removes every entry linked to a file reference. Synchronous. Requires WRITE permission on the dataset.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `filereference` | yes | The file reference to remove, as you stored it (with the trailing `::`, without the appended timestamp) |
| `dataset` | no | Target dataset/bucket name |
| `user_request_id` | no | Caller-supplied request identifier, echoed back in the response |

**Success:** `tqnn_response` is `REMOVE_OK`, with `lines_removed`, `tuples_deleted` and `buckets_scanned`. A file reference with no entries returns `NOT_FOUND`.

**Errors:** HTTP 400 `MISSING_REQUIRED_PARAMETER: filereference` · 401 `AUTHENTICATION_ERROR` · 403 `DATASET_ACCESS_DENIED` (no WRITE permission on the dataset).

**To update a document:** `removeDoc` the old file reference, then `storeDoc` the new data.

---

### POST /v1/discoverDatasets

Lists the datasets your credentials can access. Fields: `tqnnAPIKEY`, `tqnnAPISECRET`.

| Response field | Description |
|----------------|-------------|
| `scope` | `owner` (all datasets in the namespace) or `sub_credential` (the ACL whitelist) |
| `permissions` | Permissions held, e.g. `READ`, `WRITE` |
| `datasets` / `dataset_count` | Accessible datasets |
| `default_dataset` | Set only when exactly one dataset is whitelisted |
| `tqnn_response` | `OK`. HTTP 401 = `INVALID_API_KEY` |

---

## Protection Tiers

The encoding of `pattern` is a client choice. Use one method per dataset, for every store and every search.

| Tier | `storeDoc` pattern | `searchDoc` pattern |
|------|--------------------|---------------------|
| **Plain** | The JSON record as-is | The plain word |
| **PQR** | `pqr_hash(record)` | `tqnn_token16(word)` |
| **PQR + HMAC** | `pqr_hash(record)` with `PQR_KEY` set | `tqnn_token16(word)` with `PQR_KEY` set |

- Plain: send `json.dumps(record)` instead of `json.dumps(pqr_hash(record))`, and search with the plain word. Skip the hashing sections below.
- The appliance cannot tell which tier was used. A mismatch between store and search (tier, HMAC key or FPD) returns `NO_MATCH` with no error.
- The appliance tokenises and hashes every field name and word in a stored pattern itself. Client-side hashing is an additional layer.
- On `searchDoc`, plain terms are reduced to letters, digits and underscore. Terms containing other characters (e.g. hyphens) cannot be matched in Plain; use PQR or HMAC.

---

## PQR Hashing (Optional Client-Side Layer)

PQR (Post-Quantum Resistant) hashing transforms every key and value in your JSON into SHA-256 hashes before sending to the appliance. The algorithm is deterministic — the same input always produces the same hash — which is what makes search work.

### The Algorithm

**Self-Salting PQR — V1.3.0+**

> This version number tracks the **client-side hashing convention**, not the appliance's API version (`API v1.1.5` at the top of this document). The appliance is hash-agnostic — it just stores/matches opaque pattern strings — so "PQR version" only ever describes which algorithm *your* client used. If you're integrating against a corpus ingested by another client (the Workbench, `dmm-mcp-server`, or an older custom integration), confirm it used this same V1.3.0+ scheme; a mismatch means store and search will silently never match, with no error from the appliance to tell you why.

The salt is derived from the input itself via a first SHA-256 pass, so no precomputed table covers it. Tokens are a deterministic function of the plaintext: anyone who can guess a term can compute its token. For tokens that cannot be computed without a secret, use HMAC (below).

```
tqnn_token16(s):
    input  = trim(s)
    h1     = SHA-256(input)              # 64 hex chars — endogenous salt
    mixed  = input + h1                  # append salt to input
    padded = mixed[:16]                  # take first 16 chars
    token  = SHA-256(padded)[:16]        # hash and truncate to 16-char token
    return token

For each key in a JSON object:
    hashed_key = tqnn_token16(key)

For each leaf value:
    tokens = split(value, on whitespace and punctuation: /[\s,.;:]+/)
    hashed_value = tqnn_token16(token1) + "," + tqnn_token16(token2) + ...
    (comma-separated, no spaces)

For nested objects and arrays: recurse.
```

### HMAC (optional keyed layer)

Setting `PQR_KEY` makes `tqnn_token16` keyed. Tokens cannot then be computed without the key.

```
tqnn_token16(s, key):
    input  = trim(s)
    h1     = HMAC-SHA256(key, input)          # hex
    padded = (input + h1)[:16]
    token  = HMAC-SHA256(key, padded)[:16]    # hex, truncated
```

- The key is a string of at least 32 characters, used as a literal UTF-8 string (not base64-decoded). Generate once: `node -e "console.log(require('crypto').randomBytes(48).toString('base64'))"`
- The key stays on the client. It is never sent to the appliance.
- Use the same key for every store and search on the dataset. Changing the key requires re-ingesting the data. Losing it makes the data unsearchable.
- `dmm-mcp-server` reads the key from `TQNN_PQR_KEY`. DMM Workbench stores it in the connection field `pqrKey`.
- `tqnn_token16`, `pqr_hash` and `pqr_hash_reversed` below use HMAC automatically when `PQR_KEY` is set.

**Example:** The record `{"name": "Aiden Liu", "fmt": "FHIR"}` becomes:
```json
{
  "<tqnn_token16('name')>": "<tqnn_token16('Aiden')>,<tqnn_token16('Liu')>",
  "<tqnn_token16('fmt')>":  "<tqnn_token16('FHIR')>"
}
```

Note that `"Aiden Liu"` splits into two tokens — `Aiden` and `Liu` — each hashed independently. This is what enables searching for either word individually.

### Python

```python
import hashlib
import hmac
import re
import json

PQR_KEY = None   # set to a string of 32+ chars to enable HMAC (keyed) tokens

def sha256(s: str) -> str:
    return hashlib.sha256(s.encode("utf-8")).hexdigest()

def tqnn_token16(s: str) -> str:
    """Self-salting PQR token — canonical implementation V1.3.0+."""
    inp = str(s).strip()
    if PQR_KEY:                       # HMAC mode
        k      = PQR_KEY.encode("utf-8")
        h1     = hmac.new(k, inp.encode("utf-8"), hashlib.sha256).hexdigest()
        padded = (inp + h1)[:16]
        return hmac.new(k, padded.encode("utf-8"), hashlib.sha256).hexdigest()[:16]
    h1     = sha256(inp)              # endogenous salt
    padded = (inp + h1)[:16]         # first 16 chars of input+salt
    return sha256(padded)[:16]       # hash and truncate

def split_tokens(s: str) -> list:
    parts = re.split(r"[\s,\.;:]+", str(s))
    return [p for p in parts if p]

def pqr_hash(obj):
    """Recursively PQR-hash any JSON-decoded Python object."""
    if obj is None:
        return None
    if isinstance(obj, dict):
        out = {}
        for k, v in obj.items():
            hk = tqnn_token16(str(k))
            if isinstance(v, (dict, list)):
                out[hk] = pqr_hash(v)
            else:
                tokens = split_tokens(str(v)) or [""]
                out[hk] = ",".join(tqnn_token16(t) for t in tokens)
        return out
    if isinstance(obj, list):
        return [pqr_hash(item) for item in obj]
    tokens = split_tokens(str(obj)) or [""]
    return ",".join(tqnn_token16(t) for t in tokens)
```

### JavaScript (browser — async Web Crypto)

```javascript
async function sha256(msg) {
    const buf  = new TextEncoder().encode(msg);
    const hash = await crypto.subtle.digest('SHA-256', buf);
    return Array.from(new Uint8Array(hash))
        .map(b => b.toString(16).padStart(2, '0')).join('');
}

let PQR_KEY = null;   // set to a string of 32+ chars to enable HMAC (keyed) tokens

async function hmacHex(key, msg) {
    const enc = new TextEncoder();
    const k   = await crypto.subtle.importKey('raw', enc.encode(key),
                    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
    const sig = await crypto.subtle.sign('HMAC', k, enc.encode(msg));
    return Array.from(new Uint8Array(sig))
        .map(b => b.toString(16).padStart(2, '0')).join('');
}

// Self-salting PQR token — V1.3.0+
// Mirrors PHP tqnnToken16() and Python tqnn_token16() exactly.
async function tqnnToken16(s) {
    const input  = String(s).trim();
    if (PQR_KEY) {                            // HMAC mode
        const hh     = await hmacHex(PQR_KEY, input);
        const padded = (input + hh).substring(0, 16);
        return (await hmacHex(PQR_KEY, padded)).substring(0, 16);
    }
    const h1     = await sha256(input);       // endogenous salt
    const padded = (input + h1).substring(0, 16);
    return (await sha256(padded)).substring(0, 16);
}

function splitTokens(s) {
    return String(s).split(/[\s,.;:]+/).filter(t => t.length > 0);
}

async function pqrHash(obj) {
    if (obj === null || obj === undefined) return obj;
    if (Array.isArray(obj)) return Promise.all(obj.map(pqrHash));
    if (typeof obj === 'object') {
        const out = {};
        for (const [k, v] of Object.entries(obj)) {
            const hk = await tqnnToken16(k);
            if (typeof v === 'object' && v !== null) {
                out[hk] = await pqrHash(v);
            } else {
                const tokens = splitTokens(String(v));
                const hashed = await Promise.all(
                    (tokens.length ? tokens : ['']).map(t => tqnnToken16(t))
                );
                out[hk] = hashed.join(',');
            }
        }
        return out;
    }
    const tokens = splitTokens(String(obj));
    const hashed = await Promise.all(
        (tokens.length ? tokens : ['']).map(t => tqnnToken16(t))
    );
    return hashed.join(',');
}
```

---

## Storing a Document

### Python example

```python
import urllib.parse
import urllib.request
import http.client
import json
import ssl

def store_doc(base_url, api_key, api_secret, dataset, file_ref, record):
    """Hash a record and store it in the DMM. Returns True on STORE_OK."""
    hashed = pqr_hash(record)
    body = urllib.parse.urlencode({
        "tqnnAPIKEY":    api_key,
        "tqnnAPISECRET": api_secret,
        "filereference": file_ref,
        "pattern":       json.dumps(hashed, separators=(",", ":")),
        "dataset":       dataset,
    }).encode("utf-8")
    ctx = ssl._create_unverified_context()   # remove for production with valid cert
    req = urllib.request.Request(
        f"{base_url}/v1/storeDoc", data=body, method="POST",
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
        result = json.loads(resp.read())
    return result.get("tqnn_response") == "STORE_OK"

# Usage
record = {
    "id": "REC-00010149",
    "name": "Sebastian Gupta",
    "nhs": "7416185009",
    "email": "sebastian_gupta@me.com",
    "dob": "1977-02-14",
    "diag": "13213009",
    "dlbl": "Congenital heart disease",
    "fmt": "FHIR"
}

ok = store_doc(
    base_url   = "https://tqnn.local",
    api_key    = "YOUR_API_KEY",
    api_secret = "YOUR_API_SECRET",
    dataset    = "demoapp",
    file_ref   = "records_0001.jsonl::line149::REC-00010149::",
    record     = record
)
```

### JavaScript example

```javascript
async function storeDoc(baseUrl, apiKey, apiSecret, dataset, fileRef, record) {
    const hashed = await pqrHash(record);
    const fd = new FormData();
    fd.append('tqnnAPIKEY',    apiKey);
    fd.append('tqnnAPISECRET', apiSecret);
    fd.append('filereference', fileRef);
    fd.append('pattern',       JSON.stringify(hashed));
    fd.append('dataset',       dataset);
    const res  = await fetch(`${baseUrl}/v1/storeDoc`, { method: 'POST', body: fd });
    const data = await res.json();
    return data.tqnn_response === 'STORE_OK';
}
```

---

## Removing a Document

```python
import urllib.error

def remove_doc(base_url, api_key, api_secret, dataset, file_ref):
    """Remove every entry linked to file_ref (as stored, with trailing ::). Returns the response dict."""
    body = urllib.parse.urlencode({
        "tqnnAPIKEY":    api_key,
        "tqnnAPISECRET": api_secret,
        "filereference": file_ref,
        "dataset":       dataset,
    }).encode("utf-8")
    ctx = ssl._create_unverified_context()
    req = urllib.request.Request(
        f"{base_url}/v1/removeDoc", data=body, method="POST",
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    try:
        with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as e:
        return json.loads(e.read())    # 400 / 401 / 403 carry a JSON body

# Update a record: remove, then store again
remove_doc(base_url, key, secret, dataset, file_ref)
store_doc(base_url, key, secret, dataset, file_ref, new_record)
```

```javascript
async function removeDoc(baseUrl, apiKey, apiSecret, dataset, fileRef) {
    const fd = new FormData();
    fd.append('tqnnAPIKEY',    apiKey);
    fd.append('tqnnAPISECRET', apiSecret);
    fd.append('filereference', fileRef);
    fd.append('dataset',       dataset);
    const res = await fetch(`${baseUrl}/v1/removeDoc`, { method: 'POST', body: fd });
    return res.json();   // success and 400/401/403 both return JSON
}
```

---

## Searching

### Single-token search

Hash one word and call `searchDoc`. Get back a list of file references for documents that contain that word anywhere in their stored metadata.

```python
def search_one_token(base_url, api_key, api_secret, dataset, word):
    """Search for a single word. Returns list of file reference strings."""
    pattern = tqnn_token16(word)   # Self-salting PQR — V1.3.0+
    body = urllib.parse.urlencode({
        "tqnnAPIKEY":      api_key,
        "tqnnAPISECRET":   api_secret,
        "pattern":         pattern,
        "return_filelist": 1,
        "dataset":         dataset,
    }).encode("utf-8")
    ctx = ssl._create_unverified_context()
    req = urllib.request.Request(
        f"{base_url}/v1/searchDoc", data=body, method="POST",
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
        result = json.loads(resp.read())
    if result.get("tqnn_response") == "MATCH":
        return [r for r in result.get("filelist", "").strip().split("\n") if r]
    return []
```

### Multi-word search (AND logic)

To find documents containing multiple words, search for each word separately and intersect the results. **Never join multiple words into a single pattern string** — that does not work.

```python
def search_phrase(base_url, api_key, api_secret, dataset, phrase):
    """
    Find documents containing ALL words in the phrase.
    "Aiden Liu" → search("Aiden") ∩ search("Liu")
    Returns only file references present in every token's result.
    """
    tokens = split_tokens(phrase)
    if not tokens:
        return []

    def stable_key(ref):
        # TQNN appends ::timestamp to every file reference.
        # Strip it for comparison — the same record's forward and reverse
        # stores get different timestamps, so comparing full strings fails.
        return ref.rsplit("::", 1)[0]

    master = None
    for token in tokens:
        refs = search_one_token(base_url, api_key, api_secret, dataset, token)
        token_set = set(stable_key(r) for r in refs)
        master = token_set if master is None else master & token_set
        if not master:
            break   # empty intersection — no point continuing

    return sorted(master) if master else []

# Usage
results = search_phrase(
    "https://tqnn.local", "YOUR_KEY", "YOUR_SECRET", "demoapp", "Aiden Liu"
)
# Returns: ["records_0001.jsonl::line3253::REC-00013253::fpd_f6c50cc7"]
```

**Why strip the timestamp?**

TQNN appends a Unix timestamp to every file reference at store time. The same record stored twice (e.g. forward + reverse hash for FPD) gets slightly different timestamps. Always strip the trailing `::timestamp` segment before comparing across calls:

```python
stable = ref.rsplit("::", 1)[0]
# "records.jsonl::line3::REC-003::fpd_abc123::1780315560"
# becomes:
# "records.jsonl::line3::REC-003::fpd_abc123"
```

> **Note:** `rsplit("::", 1)[0]` always strips whatever follows the last `::`, regardless of whether it's actually a timestamp. This is fine for every reference the appliance actually returns to you, since the last segment is always the appended timestamp. If you're writing code that also has to handle hand-built or third-party file references that might *not* end in a real timestamp, use a stricter check instead — only strip when the trailing segment is all-digits: `re.sub(r"::\d+$", "", ref)`.

---

## fileReference Design

The `filereference` is the handle returned in search results. Design it to carry enough context to retrieve your original source record without a secondary database lookup.

**Recommended format:**
```
<source_file>::line<N>::<record_id>::
```

**Rules:**
- Always end with `::` — TQNN appends its own Unix timestamp after the last `::`. Without the trailing `::` the timestamp fuses with your record ID, corrupting the reference.
- Use `::` as the delimiter throughout so segments are easy to split on.
- Keep it URL-safe — no spaces or special characters.
- To resolve references with `dmm-mcp-server`'s `tqnn_get`, begin with a namespace of letters/digits followed by `_` (e.g. `records_`, `invoices_`). Anything else goes to the `*` catch-all resolver.
- File references are stored and returned unhashed. Use opaque IDs; never include names, ID numbers or other personal data.

**Examples:**
```
invoices_2024.jsonl::line42::INV-00042::
records_0001.jsonl::line149::REC-00010149::
reports_q3::doc_0091::
```

When using FPD (see below), include the specialcode segment before the trailing `::`:
```
records_0001.jsonl::line149::REC-00010149::fpd_ef99e90b::
```

---

## False Positive Defence (FPD)

The TQNN memory graph uses shared hub tokens (common substrings like `com`, `uk`, `hl7`) that can cause false positive matches — documents returned that don't actually contain your search term. FPD eliminates these.

**FPD requires action at both ingest and search time.** If a dataset was ingested without FPD, enabling FPD at search will produce false negatives (missed genuine matches). The two must be consistent.

### How it works

At ingest, store each document **twice** with the same `filereference`:
1. **Forward store** — self-salting PQR token: `tqnn_token16(token)`
2. **Reverse-input store** — token string reversed before self-salting: `tqnn_token16(token[::-1])`

Note: reverse the **input string**, not the hash output.

At search, for each token run **two** `searchDoc` calls:
1. Forward: `tqnn_token16(token)`
2. Reverse: `tqnn_token16(token[::-1])`

AND the two result sets. Genuine matches appear in both. False positives from hub token traversal do not survive both calls.

### FPD specialcode

Because forward and reverse stores happen milliseconds apart, TQNN assigns them slightly different timestamps. Include a stable `fpd_XXXXXXXX` code in the fileReference so you can match them across calls by stripping the timestamp:

```python
def fpd_specialcode(rec_id: str) -> str:
    """Derive a stable 8-char code from the record ID. V1.3.0+ self-salting."""
    return "fpd_" + tqnn_token16(str(rec_id))[:8]

# Build file reference with FPD code
rec_id = "REC-00010149"
code   = fpd_specialcode(rec_id)   # → "fpd_XXXXXXXX" (self-salted)
file_ref = f"records_0001.jsonl::line149::{rec_id}::{code}::"
# → "records_0001.jsonl::line149::REC-00010149::fpd_XXXXXXXX::"
```

Both the forward and reverse `storeDoc` calls use this **identical fileReference**.

### Reverse-input hash (Python)

```python
def pqr_hash_reversed(obj):
    """
    Same as pqr_hash but every token string is reversed BEFORE tqnn_token16.
    Reverse the INPUT string — not the hash output. V1.3.0+ self-salting.
    """
    if obj is None:
        return None
    if isinstance(obj, dict):
        out = {}
        for k, v in obj.items():
            hk = tqnn_token16(str(k)[::-1])
            if isinstance(v, (dict, list)):
                out[hk] = pqr_hash_reversed(v)
            else:
                tokens = split_tokens(str(v)) or [""]
                out[hk] = ",".join(tqnn_token16(t[::-1]) for t in tokens)
        return out
    if isinstance(obj, list):
        return [pqr_hash_reversed(item) for item in obj]
    tokens = split_tokens(str(obj)) or [""]
    return ",".join(tqnn_token16(t[::-1]) for t in tokens)
```

### FPD ingest (Python)

```python
def store_doc_with_fpd(base_url, api_key, api_secret, dataset, file_ref, record):
    """
    Store a record twice — forward hash then reverse-input hash.
    Both calls use the same file_ref.
    Returns True only if both stores succeed.
    """
    hashed_fwd = pqr_hash(record)
    hashed_rev = pqr_hash_reversed(record)

    ok_fwd = _post_store(base_url, api_key, api_secret, dataset, file_ref, hashed_fwd)
    if not ok_fwd:
        return False
    ok_rev = _post_store(base_url, api_key, api_secret, dataset, file_ref, hashed_rev)
    return ok_rev

def _post_store(base_url, api_key, api_secret, dataset, file_ref, hashed):
    body = urllib.parse.urlencode({
        "tqnnAPIKEY":    api_key,
        "tqnnAPISECRET": api_secret,
        "filereference": file_ref,
        "pattern":       json.dumps(hashed, separators=(",", ":")),
        "dataset":       dataset,
    }).encode("utf-8")
    ctx = ssl._create_unverified_context()
    req = urllib.request.Request(
        f"{base_url}/v1/storeDoc", data=body, method="POST",
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
        return json.loads(resp.read()).get("tqnn_response") == "STORE_OK"
```

### FPD search (Python)

```python
def search_phrase_fpd(base_url, api_key, api_secret, dataset, phrase):
    """
    FPD-enabled multi-term search.
    Per token: forward search ∩ reverse search → AND across all tokens.
    Requires the corpus was ingested with FPD (double-store).
    """
    tokens = split_tokens(phrase)
    if not tokens:
        return []

    def stable_key(ref):
        return ref.rsplit("::", 1)[0]

    def search_raw(pattern):
        body = urllib.parse.urlencode({
            "tqnnAPIKEY":      api_key,
            "tqnnAPISECRET":   api_secret,
            "pattern":         pattern,
            "return_filelist": 1,
            "dataset":         dataset,
        }).encode("utf-8")
        ctx = ssl._create_unverified_context()
        req = urllib.request.Request(
            f"{base_url}/v1/searchDoc", data=body, method="POST",
            headers={"Content-Type": "application/x-www-form-urlencoded"}
        )
        with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
            result = json.loads(resp.read())
        lines = result.get("filelist", "").strip().split("\n")
        return set(stable_key(r) for r in lines if r)

    master = None
    for token in tokens:
        fwd_set = search_raw(tqnn_token16(token))           # V1.3.0+ self-salting
        rev_set = search_raw(tqnn_token16(token[::-1]))     # reverse input
        token_set = fwd_set & rev_set       # FPD: must appear in both
        master = token_set if master is None else master & token_set
        if not master:
            break

    return sorted(master) if master else []
```

---

## Bulk Ingest from JSONL

For ingesting large files, use a persistent HTTP connection and a thread pool. The appliance processes requests synchronously — threading improves throughput proportionally up to the appliance's CPU ceiling (4 threads is a safe default on a Raspberry Pi 5; `storeDoc` throughput plateaus at approximately 16 threads and `searchDoc` scales to approximately 64).

```python
import http.client
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed

class TQNNConnection:
    """Persistent HTTP connection — one instance per thread."""
    def __init__(self, base_url, no_verify=False):
        parsed         = urllib.parse.urlparse(base_url)
        self.host      = parsed.netloc
        self.scheme    = parsed.scheme
        self.no_verify = no_verify
        self._conn     = None
        self._connect()

    def _connect(self):
        if self.scheme == "https":
            ctx = ssl._create_unverified_context() if self.no_verify \
                  else ssl.create_default_context()
            self._conn = http.client.HTTPSConnection(self.host, context=ctx, timeout=30)
        else:
            self._conn = http.client.HTTPConnection(self.host, timeout=30)

    def post(self, endpoint, body):
        headers = {"Content-Type": "application/x-www-form-urlencoded",
                   "Connection":   "keep-alive"}
        for attempt in range(2):
            try:
                self._conn.request("POST", endpoint, body=body, headers=headers)
                resp = self._conn.getresponse()
                return json.loads(resp.read().decode("utf-8"))
            except (http.client.RemoteDisconnected, BrokenPipeError,
                    ConnectionResetError, http.client.CannotSendRequest):
                self._connect()
        return {"tqnn_response": "CONNECTION_ERROR"}

# Thread-local connection — one per worker thread
_thread_local = threading.local()

def get_conn(base_url, no_verify=True):
    if not hasattr(_thread_local, "conn"):
        _thread_local.conn = TQNNConnection(base_url, no_verify=no_verify)
    return _thread_local.conn


def ingest_jsonl(path, base_url, api_key, api_secret, dataset,
                 fpd=True, threads=4):
    """
    Ingest a JSONL file into the DMM.
    - fpd=True: double-store each record (forward + reverse hash)
    - threads: worker pool size
    - Resume: completed records logged to <path>.tqnn_resume; safe to restart
    """
    import os
    basename    = os.path.basename(path)
    resume_path = path + ".tqnn_resume"
    resume_lock = threading.Lock()

    # Load resume log (file references already stored)
    resume_set = set()
    if os.path.exists(resume_path):
        with open(resume_path) as f:
            resume_set = {l.strip() for l in f if l.strip()}

    # Build task list
    tasks = []
    with open(path) as f:
        for line_num, raw_line in enumerate(f, 1):
            raw_line = raw_line.strip()
            if not raw_line:
                continue
            record  = json.loads(raw_line)
            rec_id  = record.get("id") or f"REC-{line_num:08d}"
            scode   = fpd_specialcode(rec_id) if fpd else ""
            file_ref = (
                f"{basename}::line{line_num}::{rec_id}::{scode}::"
                if scode else
                f"{basename}::line{line_num}::{rec_id}::"
            )
            if file_ref in resume_set:
                continue
            tasks.append({"record": record, "file_ref": file_ref})

    def run_task(task):
        conn = get_conn(base_url, no_verify=True)
        record   = task["record"]
        file_ref = task["file_ref"]

        def post_store(hashed):
            body = urllib.parse.urlencode({
                "tqnnAPIKEY":    api_key,
                "tqnnAPISECRET": api_secret,
                "filereference": file_ref,
                "pattern":       json.dumps(hashed, separators=(",", ":")),
                "dataset":       dataset,
            }).encode("utf-8")
            return conn.post("/v1/storeDoc", body).get("tqnn_response") == "STORE_OK"

        # Forward store (always)
        ok = post_store(pqr_hash(record))
        # Reverse-input store (FPD only)
        if ok and fpd:
            ok = post_store(pqr_hash_reversed(record))

        if ok:
            with resume_lock:
                with open(resume_path, "a") as rf:
                    rf.write(file_ref + "\n")
        return ok

    ok_count  = 0
    err_count = 0
    with ThreadPoolExecutor(max_workers=threads) as pool:
        futures = {pool.submit(run_task, t): t for t in tasks}
        for future in as_completed(futures):
            if future.result():
                ok_count += 1
            else:
                err_count += 1

    print(f"Done. Stored: {ok_count}  Errors: {err_count}  Skipped: {len(resume_set)}")
```

---

## Common Patterns

### Pattern 1 — Store and search any JSON document

```python
# Store
doc = {"title": "Q3 Revenue Report", "author": "finance", "status": "approved", "year": "2024"}
store_doc(base_url, key, secret, "reports", "reports::doc_001::", doc)

# Search — find all approved documents
results = search_phrase(base_url, key, secret, "reports", "approved")

# Search — find all finance documents from 2024
results = search_phrase(base_url, key, secret, "reports", "finance 2024")
```

### Pattern 2 — Search by any field value

Any field stored at ingest time is searchable. Email domains, diagnosis codes, record formats — all tokenised and indexed automatically.

```python
# Find all records with an icloud.com email
results = search_phrase(base_url, key, secret, "demoapp", "icloud")

# Find all FHIR-format records with Type 2 diabetes
results = search_phrase(base_url, key, secret, "demoapp", "FHIR diabetes")

# Find all records with a specific NHS number
results = search_phrase(base_url, key, secret, "demoapp", "7416185009")
```

### Pattern 3 — Credential storage with authID / registerID

```python
import urllib.request

def register_user(base_url, api_key, api_secret, username, password, dataset="users"):
    body = urllib.parse.urlencode({
        "tqnnAPIKEY":    api_key,
        "tqnnAPISECRET": api_secret,
        "credential0":   username,
        "credential1":   password,
        "multihash":     1,
        "dataset":       dataset,
    }).encode("utf-8")
    ctx = ssl._create_unverified_context()
    req = urllib.request.Request(
        f"{base_url}/v1/registerID", data=body, method="POST",
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
        return json.loads(resp.read())

def authenticate_user(base_url, api_key, api_secret, username, password, dataset="users"):
    body = urllib.parse.urlencode({
        "tqnnAPIKEY":       api_key,
        "tqnnAPISECRET":    api_secret,
        "username":         username,
        "password":         password,
        "multihash":        1,
        "returnauthtoken":  1,
        "dataset":          dataset,
    }).encode("utf-8")
    ctx = ssl._create_unverified_context()
    req = urllib.request.Request(
        f"{base_url}/v1/authID", data=body, method="POST",
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    with urllib.request.urlopen(req, context=ctx, timeout=30) as resp:
        return json.loads(resp.read())
```

---

## Security Best Practice

Hashing and HMAC run on the client. The appliance never receives the HMAC key.

- **Plain** — the appliance hashes every stored word and field name itself. Use when index contents are not sensitive.
- **PQR** — adds a client-side hash. Tokens are a deterministic function of the plaintext, so anyone who can guess a term can compute its token.
- **PQR + HMAC** — tokens cannot be computed without the key. Use for sensitive datasets and for guessable values (names, ID numbers, dates).
- **File references are not hashed.** Use opaque IDs; look the record up in your system of record.
- Keep the HMAC key (`PQR_KEY` / `TQNN_PQR_KEY`) and API credentials in environment variables or a secret store. Do not commit, log or transmit the HMAC key.
- Use sub-credentials with the minimum dataset permissions (READ for search-only clients).
- Verify TLS in production. `ssl._create_unverified_context()` is for self-signed local appliances only.

---

## Gotchas — Read These Before You Start

**1. Multi-word values become multiple search tokens**
`"name": "Aiden Liu"` stores two tokens — `Aiden` and `Liu`. Searching for `"Aiden Liu"` as a single string will not work. Make two separate `searchDoc` calls and AND the results.

**2. The trailing `::` on fileReference is mandatory**
Without it, TQNN's appended timestamp fuses with your record ID: `REC-00042001780314679`. Always end your fileReference with `::`.

**3. Strip the TQNN timestamp before set intersections**
`searchDoc` returns full references including a TQNN-appended timestamp. The same record stored twice (FPD) gets two different timestamps. Always strip with `ref.rsplit("::", 1)[0]` before comparing.

**4. FPD at search requires FPD at ingest**
If a corpus was ingested without the reverse-input double-store, enabling FPD at search will drop genuine matches. The two must be consistent.

**5. One token per searchDoc call**
Never send `tqnn_token16("Aiden") + "," + tqnn_token16("Liu")` as a single pattern. Each token is one call. The comma-separated format is only used inside stored document patterns to represent multi-token field values.

**6. SSL verification on local appliances**
Use `ssl._create_unverified_context()` for `tqnn.local` with self-signed certificates. Remove this for production deployments with valid certificates.

**7. dataset is a bucket**
Different datasets are isolated from each other. Searching `dataset=demoapp` will not find records stored in `dataset=production`. Always specify the correct dataset.

**8. Matching is case-sensitive**
`Gupta` and `gupta` produce different tokens. Use the same casing at store and search.

**9. Tokenisation**
Stored values split on whitespace, `,`, `.`, `;` and `:`. Other characters (`-`, `@`, `/`) stay inside the token. Use the same split on the search side.

**10. Error responses carry a JSON body**
400 / 401 / 403 return JSON. With `urllib`, read it from `HTTPError.read()`.

---

## Quick Checklist for a New Integration

- [ ] Split combined credential token at position 37 to get `api_key` and `api_secret`
- [ ] Choose a protection tier (Plain, PQR, PQR + HMAC) and use it for every store and search
- [ ] If hashing: implement `sha256`, `tqnn_token16`, `split_tokens`, `pqr_hash`. For HMAC also set `PQR_KEY` (32+ chars, client-side only)
- [ ] Design `fileReference` with `::` delimiters and a trailing `::`
- [ ] Encode each record per your tier (`pqr_hash` for PQR / HMAC) before calling `storeDoc`
- [ ] If using FPD: also implement `pqr_hash_reversed` and `fpd_specialcode`; call `storeDoc` twice per record with the same fileReference
- [ ] For search: encode one word at a time (`tqnn_token16(word)` for PQR / HMAC)
- [ ] For multi-word search: one `searchDoc` call per word, AND the result sets
- [ ] Strip TQNN timestamp before set operations: `ref.rsplit("::", 1)[0]`
- [ ] For FPD search: per token, also run reverse hash search and intersect before ANDing across tokens
- [ ] Use `return_filelist=1` on `searchDoc` to get file references back
- [ ] To change a record: `removeDoc`, then `storeDoc`

---

## See Also: Content Resolution and MCP Tools

Everything above gets you `storeDoc`/`searchDoc` file references — stable identifiers, but *your* application is responsible for turning a file reference back into actual content (typically your own database lookup by `record_id`).

If you'd rather not build and maintain that lookup layer yourself, or you want Claude (or another MCP-compatible LLM) to call DMM directly as a tool rather than going through your own API, Toridion also ships **`dmm-mcp-server`** — an MCP server that wraps this same appliance and adds:

- A **resolver layer** (`tqnn_get`) that maps a file reference's namespace prefix (e.g. `records_`, `invoices_`) to wherever that content actually lives — local file, S3, SharePoint, cold storage, or a custom handler you write — so file references resolve to real content without you hand-rolling the lookup.
- MCP tools (`tqnn_search`, `tqnn_similarity`, `tqnn_similarity_plain`, `tqnn_store`, `tqnn_get`, `tqnn_discover_datasets`, `tqnn_status`) that let Claude call DMM directly during a conversation, including IDF-weighted similarity search on top of the raw token matching this guide covers. The `hmac` option uses `TQNN_PQR_KEY` (32+ characters) from the server environment.
- OAuth 2.1 auth for exposing DMM to claude.ai as a remote connector, with optional per-employee accounts and per-employee DMM sub-credentials.

The two are complementary, not competing: `dmm-mcp-server` calls the exact same `storeDoc`/`searchDoc` endpoints documented above, using the same encoding options (Plain, PQR, HMAC) — it just adds the content-resolution and LLM-tool-calling layers on top. `removeDoc` is called directly on the REST API. See the `dmm-mcp-server` repo's README and `docs/resolvers.md` for details.

---

## Contact

**Toridion Ltd**
Email: `info@toridion.com`
Contact: `scot.forshaw@toridion.com`

*TQNN DMM · tqnn_dmm_integration_guide.md · © 2026 Toridion Ltd*