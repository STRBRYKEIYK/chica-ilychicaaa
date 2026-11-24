**No blocking findings.** I found no auth bypass, no way to forge the other author, no overwrite race and no privacy leak in the three backend files. The issues below are hardening and correctness fixes. The frontend web component wasn't included, so the UI race and data-loss items are listed as test gaps, not findings.

---

### Medium

**M1. `id` can be an array instead of a string, which saves a broken object for good.** `date/api/_comments.js:23` (and `:34`)
`ID.test(body.id || "")` converts its input to a string first, so `{"id":["<valid-uuid>"]}` passes. Line 26 then keeps the raw array, and line 45 of `comments.js` saves it. `safeComment` (`:34`) accepts it again and returns `id` as an array. Two things break:
- `a.id.localeCompare` at `_comments.js:62` throws whenever two comments have the same `createdAt`. That turns the whole page's GET into a lasting 503.
- The frontend gets a non-string `id`, which can break keys, dedupe or DOM ids.

`page` is safe because `Set.has` doesn't convert types. Only a key holder can trigger this, but a client bug could too, and the object can never be removed.
**Fix:** `typeof body.id === "string" && ID.test(body.id)` in `sanitize`. Do the same with `typeof value.id === "string"` in `safeComment`.

**M2. One bad or failed blob read takes down the whole page.** `date/api/_comments.js:43`, `:60`
`JSON.parse` on a corrupt or partial object throws, and so does a temporary `get` failure. Either one makes `Promise.all` reject, and the GET returns 503 for every comment on the page. A corrupt object never goes away, so neither does the outage.
**Fix:** Use `Promise.allSettled`, or wrap each read in `try { … } catch { return null }`. Then `safeComment` filters out the failures. Log the pathname, not the content.

**M3. Every read depends on how the SDK's `get()` behaves, and that isn't tested.** `date/api/_comments.js:41-42`
The code assumes a missing blob gives `null` or a non-200 `statusCode`, not an exception. It also assumes `statusCode` exists on the result.
- If `get` **throws** on a missing blob, line 40 of `comments.js` throws for every new comment, so all POSTs return 503.
- If `statusCode` is **missing**, every read returns null. GET then shows nothing, and every retry ends in 503 at `comments.js:51`.

**Fix:** Pin the `@vercel/blob` version. Add an integration test against a real private store covering: missing pathname, existing pathname, the `useCache:false` flag and `get` by pathname. Explicitly catch the not-found error and return null.

### Low

**L1. Malformed JSON is reported as a storage error.** `date/api/comments.js:36`
On Vercel, reading `req.body` throws when the JSON is invalid. The outer catch turns that into `503 storage`. If the client retries on 503, it will retry a request that can never succeed.
**Fix:** Wrap the body access in its own try/catch and return `400 json`.

**L2. A stored object that fails validation blocks that id permanently.** `date/api/comments.js:40-51`
If an object exists but `safeComment` rejects it (corrupt, or the M1 case), line 40 sees nothing and line 47 tries to write. The write fails because the object exists, line 50 rejects it again, and line 51 throws a 503. Every retry of that id hits the same 503.
**Fix:** If `read` returns something but `safeComment` rejects it, return `409 immutable` so the client generates a new id.

**L3. Paging order and display order don't match.** `date/api/_comments.js:56` vs `:62`
Paging is sorted by `uploadedAt` (time of upload). Each page is then re-sorted by `createdAt` (server clock before upload). Nothing is lost or duplicated, because the cursor is a stable pathname. But near-simultaneous comments from both authors can appear out of order where pages meet.
**Fix:** Pick one order. Either drop the re-sort at line 62, or put `createdAt` in the pathname and sort by that.

**L4. Every GET lists every comment on the page.** `date/api/_comments.js:51-55`
Each GET lists everything under the page prefix and then makes up to 30 `get` calls. That's fine for two people. But if 8 entrypoints refresh or poll, Blob operation costs and latency grow with the number of comments.
**Fix:** Either put a sortable timestamp in the pathname and use the `list` cursor directly, or cache a small index.

**L5. Invisible formatting characters are kept.** `date/api/_comments.js:24`
Bidi overrides (U+202A–202E, U+2066–2069) and zero-width characters survive sanitizing, so the displayed text can look different from what was typed. This matters little with two trusted authors.
**Fix:** Strip `[\u200b-\u200f\u202a-\u202e\u2066-\u2069\ufeff]`.

### Checked and fine
- **Auth:** the key format requires 43 characters (256 bits). Hash comparisons are equal length and use `timingSafeEqual`. The author is taken from the key, never from the request body, so one person can't post as the other. The access hashes can't be overridden in production, because Vercel calls the handler with only `req` and `res`.
- **Origin/CORS:** the allow-list runs before auth, sends `Vary: Origin` and allows no credentials. CORS isn't the security boundary here; the bearer key is.
- **`before` cursor:** it must be a string of at most 200 characters with the page prefix, so it can't reach other pages.
- **Idempotency:** `sanitize` always produces the same text for the same input. Writing with `allowOverwrite:false`, then reading back after a failure, correctly handles two identical requests racing and writes that succeeded but timed out.
- **Logging:** only `error.name` is logged, so no comment text or keys end up in logs.

### Remaining test gaps (must cover before shipping)
1. **Frontend (not provided):**
   - Comment text must be rendered with `textContent`, never `innerHTML`.
   - The retry `id` must be kept across a timeout or 503. It must be regenerated if the text is **edited** after a failed send, or the user gets a lasting 409 and loses the comment.
   - Unsent drafts must survive switching between the 8 entrypoints and page reloads (save to `localStorage` per page).
   - Slow responses must not land in the wrong place: a late GET for the previous page, or for the previous cursor, must not overwrite newer state.
   - A comment the user just posted must stay visible even if the next GET doesn't include it yet.
2. **Blob listing delay:** check whether `list` shows a new comment immediately after a 201. If not, the UI has to merge in the user's own new comments itself.
3. **Allowed origins:** `localhost` and Vercel preview URLs get a 403. Confirm that's intended and that all 5 deployments map to the 6 listed origins.
4. **Shared Blob store:** confirm `_dates.js` never lists or deletes under `library-comments/`, and that the store is private-only.
5. **Concurrency tests:**
   - Two identical POSTs at once should give exactly one 201 and one 200.
   - Same id with different text should give 409.
   - Same id on two different pages should give two separate objects.
6. **Unit tests for M1 and M2:** an array or number `id`, and a stored object containing invalid JSON.
