# Security Report — logos-mcp-server

Audit date: 2026-03-24
Status: **Criticals resolved. Items below are open.**

---

## Already Resolved

| Issue | Resolution |
|-------|-----------|
| `.mcp.json` not gitignored | Already present in `.gitignore` — no action needed |
| API key in URL query params | Biblia API only supports `?key=...` authentication — this is a third-party API constraint, not fixable in our code. Documented below. |
| macOS-only data paths | Fixed: `config.ts` now detects `win32` and uses `%LOCALAPPDATA%\Logos4\...` |
| macOS-only `open` / `osascript` | Fixed: `logos-app.ts` now uses `cmd /c start` and `tasklist` on Windows |

---

## API Key in URL Query Parameters (Accepted Risk)

**File:** `logos-mcp-server/src/services/biblia-api.ts:10`

The Biblia API requires the key as `?key=<value>`. There is no `Authorization` header option. This means the key appears in:
- Server-side HTTP logs
- Any proxy or CDN access logs

**Mitigation in place:** The key is loaded from the environment (`.mcp.json` env block, never hardcoded), and `.mcp.json` is gitignored.
**Residual risk:** Low — this is a free-tier Bible text API. Treat the key as semi-public and rotate it if logs are ever compromised.

---

## Open Issues

### HIGH — LIKE Wildcard Injection

**Files:**
- `catalog-reader.ts:128-140`
- `sqlite-reader.ts:263-266`

User input passed to SQL `LIKE` patterns is not escaped, so `%`, `_`, and `\` in the input match beyond intent.

**Example affected code:**
```typescript
params.push(`%${options.type}%`);  // options.type can contain % or _
```

**Fix:**
```typescript
function escapeLike(s: string): string {
  return s.replace(/[%_\\]/g, '\\$&');
}
params.push(`%${escapeLike(options.type)}%`);
// Add ESCAPE '\\' to the SQL clause:
sql += " AND Type LIKE ? ESCAPE '\\'";
```

Apply to all four LIKE patterns in both files.

---

### MEDIUM — No Path Traversal Guard on `LOGOS_DATA_DIR`

**File:** `config.ts`

If `LOGOS_DATA_DIR` is set to an arbitrary path via environment variable (e.g. `../../sensitive`), it opens an unintended SQLite file.

**Fix:**
```typescript
import { resolve } from "path";

const resolved = resolve(process.env.LOGOS_DATA_DIR ?? getLogosBase());
const expectedBase = resolve(/* platform base */);
if (!resolved.startsWith(expectedBase)) {
  throw new Error(`LOGOS_DATA_DIR outside expected Logos directory: ${resolved}`);
}
export const LOGOS_DATA_DIR = resolved;
```

**Risk level:** Low in practice (env vars are operator-controlled), but worth adding before any broader distribution.

---

### MEDIUM — No JSON Schema Validation on DB-parsed JSON

**File:** `sqlite-reader.ts:122-126, 305-312`

JSON fields read from SQLite are `JSON.parse()`d without structural validation. A corrupt or crafted DB entry could produce unexpected object shapes or prototype pollution.

**Fix:** Add Zod schemas for the expected shapes, e.g.:
```typescript
import { z } from "zod";
const TemplateSchema = z.record(z.unknown());
const parsed = TemplateSchema.safeParse(JSON.parse(r.TemplateJson));
if (!parsed.success) continue;
```

---

### MEDIUM — No Fetch Timeout

**File:** `biblia-api.ts:15`

External fetch calls have no timeout, so a slow or hanging Biblia API response blocks the MCP server indefinitely.

**Fix:**
```typescript
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 30_000);
try {
  const res = await fetch(url.toString(), { signal: controller.signal });
  // ...
} finally {
  clearTimeout(timer);
}
```

---

### MEDIUM — No Rate Limiting on Biblia API

**File:** `biblia-api.ts`

No throttle or cache layer; a misbehaving client could exhaust the free API quota.

**Fix options:**
- Simple in-memory LRU cache for `getBibleText` results (passage+bible → text)
- Token-bucket rate limiter (e.g. 10 req/s)

---

### LOW — Regex ReDoS Potential in `stripRichText`

**File:** `utils/strip-markup.ts:43`

```typescript
const regex = /Text=["']([^"']*)["']/g;
```

The character class `[^"']*` is non-backtracking on most engines, so the practical risk is low — but adversarial XAML with very long unterminated attribute values could cause slow processing.

**Fix:** Add a maximum input length guard before calling the function:
```typescript
if (para.length > 500_000) return "";
```

---

### LOW — Verbose Error Messages Expose Filesystem Paths

**File:** `tools/user-data.ts:36-38`

```typescript
return { content: [{ type: "text", text: `Error reading notes: ${msg}` }], isError: true };
```

SQLite error messages can include the full filesystem path of the database file.

**Fix:** Return a generic message and log the full error to stderr:
```typescript
console.error("DB error:", msg);
return { content: [{ type: "text", text: "Error reading database." }], isError: true };
```

---

### LOW — Unpinned Dependency

**File:** `package.json`

```json
"better-sqlite3": "^11.8.1"
```

Minor/patch updates are applied automatically. For a local tool this is low risk, but worth pinning for reproducibility:
```json
"better-sqlite3": "11.8.1"
```
Then use `npm ci` instead of `npm install` for installs.

---

### LOW — No Input Length Limits

**Files:** All tool handlers (e.g. `tools/bible.ts`, `tools/navigation.ts`)

String parameters (passage references, search queries) have no max-length validation. Very long inputs are passed to Biblia API URLs or LIKE queries without truncation.

**Fix:** Add a simple guard at the start of each tool handler:
```typescript
if (input.length > 500) throw new Error("Input too long");
```

---

## Non-Issues (Noted for Completeness)

- **No CORS** — MCP server uses stdio transport, so CORS is not applicable.
- **No authentication on MCP server** — By design; security is handled by process isolation in Claude Code.
- **Certificate pinning** — Not needed for a local development tool using system CA roots.
