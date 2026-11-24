No confirmed blocking bugs in this file, provided the two things under **High** check out: CORS preflight and how the element gets added. Rendering is safe from XSS (everything goes through `textContent`), the 20s abort-and-retry path is idempotent, and stale responses are discarded by the `generation` check. I couldn't see the backend, so the server-side items are listed as test gaps.

### High (check these against your setup)

**1. Cross-origin preflight vs. "GET and POST only" — comments.js:168**
Every request sends `Authorization`, and POST also sends `content-type: application/json`, so the browser always sends an `OPTIONS` preflight first. If any of the 5 deployments is on a different origin than `chica.keicodes.dev` and the backend rejects `OPTIONS`, every request fails. Those users just see the generic "Hindi muna mabuksan" message.
**Fix:** handle `OPTIONS` with an exact origin allowlist, `Access-Control-Allow-Headers: authorization, content-type`, `Allow-Methods: GET, POST`, and `Vary: Origin`. Don't use `*` with reflected origins.

**2. An element already in the HTML ignores `data-page` and gets no storage listener — comments.js:259**
If an entrypoint has `<library-notes>` in its markup, `boot` returns before `setAttribute("page-key", currentPage)` and before adding the `storage` listener. That element falls back to `"library"`, so notes from different pages get mixed into one thread, and the tab never hears about lock/unlock in other tabs.
**Fix:** look up the existing element, set `page-key` if it's missing, and always register the listener.

### Medium

**3. Every load overwrites the textarea — comments.js:187**
Refresh, "MGA NAUNANG TALA" (older notes) and storage-event reloads all reset `textarea.value` from `sessionStorage`. If storage is blocked (Safari with storage blocked, some embedded webviews), `draft()` returns `""` and the user's unsent text is wiped. Even when storage works, typing during a refresh moves the cursor to the end and can break IME composition.
**Fix:** only restore the draft when `(page, author)` actually changes. Keep a `this.draftScope` and skip the restore when it matches.

**4. The invite key can stay in browser history — comments.js:10, 258**
`replaceState` removes the hash from the current history entry. But the original navigation to `…#comment-key=…` may already be in global or synced history, autocomplete, and screenshots. The key is long-lived, so this is a lasting credential leak.
**Fix:** treat links as one-time invites the server exchanges for a key, or rotate keys after first use. Short of that, document it and make sure no analytics script on the page records the full URL before `boot` runs.

**5. A bad link or typo logs you out in every tab — comments.js:156, 197**
A 401 calls `lock()`, which runs `saveKey("")`. That deletes a valid stored key and, through the storage event at line 264, locks every other tab. One stale invite link is enough to do it.
**Fix:** on 401, only clear storage if `key === readKey()`. Otherwise just show the error and keep the stored key.

**6. Permanent errors are shown as "try again" — comments.js:198, 244**
400, 403, 409 and 413 all show the retry message. If the server measures length differently from the client (the client counts UTF-16 units via `.length`; the server might count code points or bytes), the user keeps retrying with the same pending id and it never succeeds.
**Fix:** check `error.status`. For 4xx other than 401/408/429, show a specific message and clear `pending` where it makes sense. Make sure the server measures length the same way as `.length`.

### Low

**7. A just-sent note can disappear — comments.js:189**
A refresh that started before the POST finished, or a Blob listing that lags behind, replaces `this.comments` without the new note. The user may think it was lost and post it again. Since `pending` was already cleared, that creates a real duplicate.
**Fix:** keep the comments this session has confirmed in a per-page map and merge them into every refresh.

**8. Errors after a successful send are reported as save failures — comments.js:234–244**
If `renderList` throws after the POST succeeds (for example, a bad `createdAt` makes `toISOString()` throw a RangeError), the catch says the save wasn't confirmed. By then the text and pending id are already gone.
**Fix:** wrap only the `request` call in the try, and guard against invalid dates in `renderList` (comments.js:213–214).

**9. A 401 with a non-JSON body doesn't lock — comments.js:169**
If `response.json()` throws, the status is lost, so a proxy or edge 401 with an HTML body never locks.
**Fix:** parse inside a try, and attach `status` before throwing.

**10. Lock can be undone across tabs — comments.js:184, 262–266**
If tab A locks while a load in tab B is in flight, B can finish and call `saveKey(key)` before it processes the removal event. That writes the key back and unlocks A again.
**Fix:** in `load`, only save if the stored value isn't `""` because of a newer lock, for example with a lock epoch in storage.

**11. Can't change identity mid-send — comments.js:116–117**
"IBANG ACCESS" works during a send but `sending` stays true. Separately, a send started on one oras slot silently blocks sends on the next slot (line 223 returns with no message).
**Fix:** disable the "change access" button while sending, or show a "still saving" message.

**12. `replaceState(null, …)` wipes `history.state` — comments.js:258**
That can break routers on any entrypoint that store state there.
**Fix:** pass `history.state` instead of `null`.

**13. Drafts stay behind after switching access — comments.js:154–161**
`lock()` leaves the previous author's drafts and pending text in `sessionStorage`. On a shared device, the next person can read them through devtools.
**Fix:** on an explicit "IBANG ACCESS", clear the `chica.comments.*` keys.

### Remaining test gaps (backend not shown)
- **Author binding:** the author must come from the key hash on the server, never from the body. Reject a body `page` that differs from the query `page` (line 233 sends both). Use an allowlist for `page`.
- **Retry ids:** a repeat of the same `(author, page, id)` with the same text must return the stored object with 200, not 409. The same id with different text must be rejected. The object path must include the author, and the write must not overwrite (`addRandomSuffix:false` plus a no-overwrite check), with a concurrent duplicate POST test.
- **Key check:** hash the bearer value, then compare in constant time. Reject keys that aren't 43 characters of base64url before hashing. Never log the `Authorization` header.
- **Blob privacy:** the blobs need private access and must never be served by URL to the client. The response must not echo blob URLs or the token.
- **Pagination:** the `before` cursor must be stable when timestamps tie (order by `createdAt` and then `id`). Test that refresh and older-page loads produce no gaps or duplicates.
- **Headers:** responses should carry `Cache-Control: no-store`. Rate-limit 401s per IP.
- **`document.currentScript`:** it's `null` under `type=module` or dynamic injection, so `configuredPage` silently falls back to `"library"`. Check this on all 8 entrypoints.
- **XSS on host pages:** any XSS on any of the 5 origins can read the key from `localStorage`. Check the CSP on each host page.
