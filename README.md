# TQNN DMM — Developer Integration Guide

**Toridion Ltd · API v1.0.1**

> Give this document to Claude or any LLM coding assistant and say: *"Using this guide, help me build a [Python/JavaScript/PHP] application that does X with TQNN DMM."* The document is self-contained — no other reference is needed.

---

## What is TQNN DMM?

TQNN DMM (Document Meta Memory) is a sovereign AI memory appliance. It stores JSON metadata as a neural associative memory graph and enables probabilistic O(1) search across all stored associations — without SQL, schemas, or indexes.

**The mental model:**
- You **store** a document by hashing its metadata fields and indexing them into the DMM graph.
- You **search** by hashing a plain-text query term and asking the DMM which documents contain that pattern.
- The DMM returns a list of **file references** — the identifiers you assigned at store time.
- You never store raw data values. Everything is SHA-256 hashed before it reaches the appliance. The original values cannot be recovered from the DMM.

**Key facts for developers:**
- All API calls are `multipart/form-data` POST requests.
- Authentication is two fields on every call: `tqnnAPIKEY` and `tqnnAPISECRET`.
- There are five endpoints: `authID`, `registerID`, `storePattern`, `storeDoc`, `searchDoc`.
- For most applications you will only use `storeDoc` and `searchDoc`.
- The `pattern` field in every call is a JSON string of SHA-256 hashed values — never plaintext.

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

### POST /v1/storeDoc

Indexes a document's metadata into the TQNN graph, associated with a file reference you provide. This is the primary ingestion endpoint.

| Field | Required | Description |
|-------|----------|-------------|
| `tqnnAPIKEY` | yes | API key |
| `tqnnAPISECRET` | yes | API secret |
| `filereference` | yes | Your stable identifier for this document (see fileReference design below) |
| `pattern` | yes | JSON string of PQR-hashed document metadata (see PQR Hash section) |
| `dataset` | no | Target dataset/bucket name (default: `demoapp`) |

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
| `pattern` | yes | PQR-hashed search token — **one token per call** (see Search section) |
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

## The PQR Hash — The Most Important Concept

**Everything stored in the DMM must be PQR-hashed. Searches must hash the query identically. If the hash doesn't match, nothing is found.**

PQR (Post-Quantum Resistant) hashing transforms every key and value in your JSON into SHA-256 hashes before sending to the appliance. The algorithm is deterministic — the same input always produces the same hash — which is what makes search work.

### The Algorithm

```
padTo16(s):
    if len(s) >= 16  →  return s[:16]          (truncate to 16 chars)
    else             →  return s.ljust(16, '*') (right-pad with '*')

For each key in a JSON object:
    hashed_key = SHA-256( padTo16(key) )

For each leaf value:
    tokens = split(value, on whitespace and punctuation: /[\s,.;:]+/)
    hashed_value = SHA-256( padTo16(token1) ) + "," + SHA-256( padTo16(token2) ) + ...
    (comma-separated, no spaces)

For nested objects and arrays: recurse.
```

**Example:** The record `{"name": "Aiden Liu", "fmt": "FHIR"}` becomes:
```json
{
  "<sha256 of 'name************'>": "<sha256 of 'Aiden***********'>,<sha256 of 'Liu*************'>",
  "<sha256 of 'fmt*************'>": "<sha256 of 'FHIR************'>"
}
```

Note that `"Aiden Liu"` splits into two tokens — `Aiden` and `Liu` — each hashed independently. This is what enables searching for either word individually.

### Python

```python
import hashlib
import re
import json

def sha256(s: str) -> str:
    return hashlib.sha256(s.encode("utf-8")).hexdigest()

def pad_to_16(s: str) -> str:
    s = str(s)
    return s[:16] if len(s) >= 16 else s.ljust(16, "*")

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
            hk = sha256(pad_to_16(str(k)))
            if isinstance(v, (dict, list)):
                out[hk] = pqr_hash(v)
            else:
                tokens = split_tokens(str(v)) or [""]
                out[hk] = ",".join(sha256(pad_to_16(t)) for t in tokens)
        return out
    if isinstance(obj, list):
        return [pqr_hash(item) for item in obj]
    tokens = split_tokens(str(obj)) or [""]
    return ",".join(sha256(pad_to_16(t)) for t in tokens)
```

### JavaScript (browser — async Web Crypto)

```javascript
async function sha256(msg) {
    const buf  = new TextEncoder().encode(msg);
    const hash = await crypto.subtle.digest('SHA-256', buf);
    return Array.from(new Uint8Array(hash))
        .map(b => b.toString(16).padStart(2, '0')).join('');
}

function padTo16(s) {
    const str = String(s);
    return str.length >= 16 ? str.substring(0, 16) : str.padEnd(16, '*');
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
            const hk = await sha256(padTo16(k));
            if (typeof v === 'object' && v !== null) {
                out[hk] = await pqrHash(v);
            } else {
                const tokens = splitTokens(String(v));
                const hashed = await Promise.all(
                    (tokens.length ? tokens : ['']).map(t => sha256(padTo16(t)))
                );
                out[hk] = hashed.join(',');
            }
        }
        return out;
    }
    const tokens = splitTokens(String(obj));
    const hashed = await Promise.all(
        (tokens.length ? tokens : ['']).map(t => sha256(padTo16(t)))
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

## Searching

### Single-token search

Hash one word and call `searchDoc`. Get back a list of file references for documents that contain that word anywhere in their stored metadata.

```python
def search_one_token(base_url, api_key, api_secret, dataset, word):
    """Search for a single word. Returns list of file reference strings."""
    pattern = sha256(pad_to_16(word))
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

**Examples:**
```
invoices.jsonl::line42::INV-00042::
patient_records.jsonl::line149::REC-00010149::
documents/2024/q3::doc_0091::
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
1. **Forward store** — normal PQR hash: `sha256(pad_to_16(token))`
2. **Reverse-input store** — token string reversed before hashing: `sha256(pad_to_16(token[::-1]))`

Note: reverse the **input string**, not the hash output.

At search, for each token run **two** `searchDoc` calls:
1. Forward: `sha256(pad_to_16(token))`
2. Reverse: `sha256(pad_to_16(token[::-1]))`

AND the two result sets. Genuine matches appear in both (the record contains the token in both its forward and reverse-hashed form). False positives from hub token traversal do not survive both calls.

### FPD specialcode

Because forward and reverse stores happen milliseconds apart, TQNN assigns them slightly different timestamps. Include a stable `fpd_XXXXXXXX` code in the fileReference so you can match them across calls by stripping the timestamp:

```python
def fpd_specialcode(rec_id: str) -> str:
    """Derive a stable 8-char code from the record ID. Zero extra cost."""
    return "fpd_" + sha256(pad_to_16(str(rec_id)))[:8]

# Build file reference with FPD code
rec_id = "REC-00010149"
code   = fpd_specialcode(rec_id)   # → "fpd_ef99e90b"
file_ref = f"records_0001.jsonl::line149::{rec_id}::{code}::"
# → "records_0001.jsonl::line149::REC-00010149::fpd_ef99e90b::"
```

Both the forward and reverse `storeDoc` calls use this **identical fileReference**.

### Reverse-input hash (Python)

```python
def pqr_hash_reversed(obj):
    """
    Same as pqr_hash but every token string is reversed BEFORE pad_to_16 + sha256.
    Reverse the INPUT string — not the hash output.
    """
    if obj is None:
        return None
    if isinstance(obj, dict):
        out = {}
        for k, v in obj.items():
            hk = sha256(pad_to_16(str(k)[::-1]))
            if isinstance(v, (dict, list)):
                out[hk] = pqr_hash_reversed(v)
            else:
                tokens = split_tokens(str(v)) or [""]
                out[hk] = ",".join(sha256(pad_to_16(t[::-1])) for t in tokens)
        return out
    if isinstance(obj, list):
        return [pqr_hash_reversed(item) for item in obj]
    tokens = split_tokens(str(obj)) or [""]
    return ",".join(sha256(pad_to_16(t[::-1])) for t in tokens)
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
        fwd_set = search_raw(sha256(pad_to_16(token)))
        rev_set = search_raw(sha256(pad_to_16(token[::-1])))
        token_set = fwd_set & rev_set       # FPD: must appear in both
        master = token_set if master is None else master & token_set
        if not master:
            break

    return sorted(master) if master else []
```

---

## Bulk Ingest from JSONL

For ingesting large files, use a persistent HTTP connection and a thread pool. The appliance processes requests synchronously — threading improves throughput proportionally up to the appliance's CPU ceiling (4 threads is a safe default on a Raspberry Pi 5).

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
Never send `sha256(padTo16("Aiden")) + "," + sha256(padTo16("Liu"))` as a single pattern. Each token is one call. The comma-separated format is only used inside stored document patterns to represent multi-token field values.

**6. SSL verification on local appliances**
Use `ssl._create_unverified_context()` for `tqnn.local` with self-signed certificates. Remove this for production deployments with valid certificates.

**7. dataset is a bucket**
Different datasets are isolated from each other. Searching `dataset=demoapp` will not find records stored in `dataset=production`. Always specify the correct dataset.

---

## Quick Checklist for a New Integration

- [ ] Split combined credential token at position 37 to get `api_key` and `api_secret`
- [ ] Implement `sha256`, `pad_to_16`, `split_tokens`, `pqr_hash`
- [ ] Design `fileReference` with `::` delimiters and a trailing `::`
- [ ] Hash every record with `pqr_hash` before calling `storeDoc`
- [ ] If using FPD: also implement `pqr_hash_reversed` and `fpd_specialcode`; call `storeDoc` twice per record with the same fileReference
- [ ] For search: hash one word at a time with `sha256(pad_to_16(word))`
- [ ] For multi-word search: one `searchDoc` call per word, AND the result sets
- [ ] Strip TQNN timestamp before set operations: `ref.rsplit("::", 1)[0]`
- [ ] For FPD search: per token, also run reverse hash search and intersect before ANDing across tokens
- [ ] Use `return_filelist=1` on `searchDoc` to get file references back

---

## Contact

**Toridion Ltd**
Email: `info@toridion.com`
Contact: `scot.forshaw@toridion.com`

*TQNN DMM · tqnn_dmm_integration_guide.md · © 2026 Toridion Ltd*