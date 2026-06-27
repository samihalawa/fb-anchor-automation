---
name: fb-anchor-automation
description: >
  Facebook automation via Anchor Browser + Playwright connectOverCDP. Covers: reading and
  replying to Messenger threads, bulk follow-up to prior-inquirer threads, posting listings
  to Marketplace, posting to Facebook Groups, bulk-commenting on group posts, joining groups,
  and searching for rental/room posts to comment on. Use when the user wants to automate any
  Facebook action (messages, marketplace, groups, comments) using Anchor Browser with cookie
  injection. ALWAYS use Anchor Browser, NEVER use local browser/Comet CDP for this skill.
  Triggers: "send to fb", "post to marketplace", "follow up facebook", "reply messenger",
  "post to groups", "bulk comment facebook", "fb automation", "facebook messenger",
  "facebook grupos", "publicar marketplace", "comment on rental posts", "join fb groups".
---

# fb-anchor-automation

## What This Skill Covers

1. **Messenger follow-up** — read + send to threads from prior inquirers by direct thread id
2. **Marketplace listing** — post a new rental/item listing with photos, price, description
3. **Group post** — publish text/image post to one or many groups
4. **Bulk group comments** — comment on recent posts across target groups
5. **Group join + comment** — search for groups, join them, then comment on rental posts
6. **Chat listing** — list recent Messenger conversations

All runs through **Anchor Browser (remote CDP)** + `playwright-core`. NEVER use local browser
(localhost:9222 / Comet). No vision/snapshot loops — direct DOM queries only. One reused `page`
per session. No `new_page` fan-out (hits fd ceiling).

---

## CRITICAL: ALWAYS USE ANCHOR BROWSER

- **NEVER** connect to `localhost:9222` or any local CDP endpoint
- **NEVER** use Comet.app or the user's local Chrome for FB automation
- **ALWAYS** start an Anchor Browser session with cookie injection
- Cookies must be provided by the user (export from Cookie-Editor extension)
- If cookies are stale (xs expired), ask the user to re-export — do not fall back to local

---

## CRITICAL TECHNICAL FACTS (from production runs)

### Cookie injection — sameSite MUST be 'None'
```js
const COOKIES = RAW.map(c => ({
  name: c.name, value: c.value,
  domain: '.facebook.com', path: '/',
  httpOnly: c.httpOnly, secure: true,
  sameSite: 'None'   // raw 'no_restriction' or null = silent auth failure
}));
await ctx.addCookies(COOKIES);  // inject BEFORE first navigation
```

### Required cookies (minimum)
- `xs` — session token (expires after ~6 Anchor sessions/day)
- `c_user` — user ID
- `datr` — device tracking
- `sb` — session binding
- Optional: `dpr`, `wd`, `presence`

### Session lifespan: ~6 Anchor sessions/day max
FB invalidates `xs` token after ~6 new Anchor sessions on same account in one day.
Detection: homepage shows login redirect or wrong user.
Fix: re-export fresh cookies from a live logged-in Chrome tab.

### Mandatory warm-up — initial_url MUST be 'about:blank'
```js
// Start session with about:blank, NOT facebook.com
session: { initial_url: 'about:blank', timeout: { idle_timeout: 900, max_duration: 2400 } }

// Then inject cookies BEFORE any facebook navigation
await ctx.addCookies(COOKIES);

// Then warm up on homepage
await page.goto('https://www.facebook.com/', { waitUntil: 'domcontentloaded' });
await sleep(8000);  // must wait — skipping causes immediate login redirect on feature URLs
```

### Messenger navigation — /messages/t/ auto-redirects to last thread
When navigating to `https://www.facebook.com/messages/t/` (no ID), FB redirects to the
most recent thread. This is normal behavior, not an error. The chat rail renders from there.

### Messenger rail virtualization
The left rail only renders ~2-5 visible threads. Scrolling loads more but extraction is
limited. To see more threads, scroll the rail container multiple times:
```js
for (let scroll = 0; scroll < 6; scroll++) {
  await page.evaluate(() => {
    const containers = document.querySelectorAll('[role="navigation"], [aria-label="Chats"]');
    for (const c of containers) c.scrollTop += 800;
  });
  await sleep(1500);
}
```

### Thread extraction — links with /messages/t/
```js
const links = document.querySelectorAll('a[href*="/messages/t/"]');
// Extract thread ID from href: /t/(\d+)
```

### Message bubble extraction — div[dir="auto"] only
```js
for (const d of pane.querySelectorAll('div[dir="auto"]')) {
  const r = d.getBoundingClientRect();
  if (!r.width) continue;
  const side = (r.x + r.width / 2) > (window.innerWidth * 0.55) ? 'ME' : 'THEM';
  msgs.push({ side, t: d.innerText.trim() });
}
// ❌ WRONG — [role="row"] nests too deep, always returns empty strings
```

### Thread navigation — direct URL only
```js
await page.goto(`https://www.facebook.com/messages/t/${id}`, { waitUntil: 'domcontentloaded' });
```

### Composer selectors — try all in order
```js
const COMPOSER_SELS = [
  'div[aria-label="Mensaje"][contenteditable="true"]',
  'div[aria-label="Message"][contenteditable="true"]',
  'div[aria-label*="nsaje"][contenteditable="true"]',
  '[role="textbox"][contenteditable="true"]',
  'div[contenteditable="true"]',
];
```

### Sending + verification
```js
await box.click(); await sleep(400);
await page.keyboard.insertText(text);
await sleep(600);
await page.keyboard.press('Enter');
await sleep(2500);
const ok = await page.evaluate(frag =>
  [...document.querySelectorAll('div[dir="auto"]')]
    .some(d => d.innerText.includes(frag)),
  text.slice(0, 30)
);
```

---

## Anchor Browser Boilerplate

```js
const { chromium } = require('playwright-core');
const API = process.env.ANCHOR_API_KEY || 'sk-77fea665b58ee8d7534828fa11a83a21';
const sleep = ms => new Promise(r => setTimeout(r, ms));

async function startAnchor() {
  const r = await fetch('https://api.anchorbrowser.io/v1/sessions', {
    method: 'POST',
    headers: { 'anchor-api-key': API, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      session: {
        initial_url: 'about:blank',  // MUST be about:blank, not facebook.com
        timeout: { idle_timeout: 900, max_duration: 2400 }
      },
      browser: { headless: { active: false }, viewport: { width: 1280, height: 950 } }
    })
  });
  return (await r.json()).data; // { id, cdp_url, live_view_url }
}

const endAnchor = id => fetch(`https://api.anchorbrowser.io/v1/sessions/${id}`,
  { method: 'DELETE', headers: { 'anchor-api-key': API } });

// Standard open
const s = await startAnchor();
console.log('SESSION', s.id, 'LIVE', s.live_view_url);
const b = await chromium.connectOverCDP(s.cdp_url);
const ctx = b.contexts()[0] || await b.newContext();
await ctx.addCookies(COOKIES);  // inject BEFORE navigation
const page = ctx.pages()[0] || await ctx.newPage();
await page.goto('https://www.facebook.com/', { waitUntil: 'domcontentloaded' });
await sleep(8000);  // mandatory warm-up
// ... do work ...
await b.close();
await endAnchor(s.id);
```

---

## 1. List Recent Messenger Chats

**Trigger**: user wants to see their recent FB conversations.

After warm-up, navigate to `/messages/t/` and extract thread links from the rail:
```js
await page.goto('https://www.facebook.com/messages/t/', { waitUntil: 'domcontentloaded' });
await sleep(7000);

// Scroll rail to load more
for (let i = 0; i < 6; i++) {
  await page.evaluate(() => {
    const containers = document.querySelectorAll('[role="navigation"], [aria-label="Chats"]');
    for (const c of containers) c.scrollTop += 800;
  });
  await sleep(1500);
}

const chats = await page.evaluate(() => {
  const results = [], seen = new Set();
  const links = document.querySelectorAll('a[href*="/messages/t/"]');
  for (const a of links) {
    const m = a.href.match(/\/t\/(\d+)/);
    if (!m || seen.has(m[1])) continue;
    seen.add(m[1]);
    let container = a;
    for (let i = 0; i < 6; i++) { if (container.parentElement) container = container.parentElement; }
    const spans = [...container.querySelectorAll('span')];
    const texts = spans.map(s => s.innerText?.trim()).filter(t => t && t.length > 1 && t.length < 120);
    results.push({ id: m[1], name: texts[0] || '?', preview: texts.slice(1,3).join(' · ') });
  }
  return results;
});
```

---

## 2. Messenger Follow-up (Prior Inquirers)

**Trigger**: user wants to follow up with people who already chatted.

Navigate directly to `/messages/t/<id>` for each thread. Use composer selectors.
See boilerplate above for sending + verification.

---

## 3. Marketplace Listing

```js
await page.goto('https://www.facebook.com/marketplace/create/rental', { waitUntil: 'domcontentloaded' });
await sleep(6000);
// Fill fields, upload images via setInputFiles, submit
```

---

## 4. Group Post

Navigate to group, click composer, insertText, submit.
Min 15s between posts across groups.

---

## 5. Group Join + Comment on Rental Posts (PROVEN WORKFLOW)

**Trigger**: user wants to comment on rental/room posts in FB groups.

### Step 1: Search for groups
```js
await page.goto('https://www.facebook.com/search/groups/?q=alquiler%20habitaciones%20madrid',
  { waitUntil: 'domcontentloaded' });
await sleep(6000);
// Extract group slugs from a[href*="/groups/"]
```

### Step 2: Join groups (click Join button)
```js
await page.goto(`https://www.facebook.com/groups/${slug}`, { waitUntil: 'domcontentloaded' });
await sleep(5000);
// Check for join button: [role="button"]:has-text("Join"), [role="button"]:has-text("Unirse")
// Click it, wait 4s, check if joined or pending
```

### Step 3: Find comment entry points (CRITICAL — proven pattern)
The comment textbox in group feeds is a `[contenteditable="true"][role="textbox"]` with
aria-label containing "comment" or "Write a public comment". Do NOT look for spans with
text "Comment"/"Comentar" — those are often inside existing comment threads, not the
post-level comment action.

```js
// Scroll to load posts first
for (let i = 0; i < 5; i++) {
  await page.evaluate(() => window.scrollBy(0, 800));
  await sleep(1500);
}
await page.evaluate(() => window.scrollTo(0, 300));
await sleep(1000);

// Find comment textboxes (PROVEN — this is what works)
const commentBoxes = await page.evaluate(() => {
  const results = [];
  const editables = document.querySelectorAll('[contenteditable="true"][role="textbox"]');
  for (const ed of editables) {
    const label = ed.getAttribute('aria-label') || '';
    if (/comment|comentario/i.test(label)) {
      const r = ed.getBoundingClientRect();
      if (r.width > 50 && r.y > 0) {
        results.push({ x: r.x + r.width/2, y: r.y + r.height/2, label });
      }
    }
  }
  return results;
});
```

### Step 4: Type and send comment
```js
await page.mouse.click(cb.x, cb.y);
await sleep(2000);
let cbox = page.locator('[contenteditable="true"][role="textbox"]').last();
await cbox.waitFor({ state: 'visible', timeout: 3000 });
await cbox.click(); await sleep(300);
await page.keyboard.insertText(commentText);
await sleep(800);
await page.keyboard.press('Enter');
await sleep(4000);
// Verify with div[dir="auto"] check
```

### Comment pacing rules
- Min 12-14s between comments
- Rotate ≥3 comment variants
- Max ~10 comments per session total
- Never identical text to same group twice in <24h

### Known working groups (Madrid rentals, joined June 2026)
- alquilahabitacionesenmadrid
- alquilerhabitacionesmadrid
- madridrent0
- alquilerparticularesmadrid
- alquiler.habitaciones.en.madrid (pending approval)

---

## 6. Bulk Group Comments (existing member groups)

Same as section 5 step 3-4, but iterate over known groups without needing to join.

---

## Failure Modes & Exact Fixes

| Symptom | Root cause | Fix |
|---|---|---|
| URL = `/login/?next=...` on any FB page | `xs` cookie expired (>6 Anchor sessions/day) | Re-export fresh cookies from Chrome |
| Homepage shows wrong user / "Continue" | Same — `xs` dead | Same |
| Composer `waitForSelector` timeout | xs expired OR FB rate-limited account | Fresh cookies |
| `MP_ROWS 0` on marketplace folder | MP uses virtual grid, no hrefs | Navigate directly to `/messages/t/<id>` |
| All rail name-clicks → same thread id | Rail virtualization | Use direct URL nav |
| `too many open files` | `new_page` in loop | One reused `page`, no parallel `new_page` |
| 429 from Anchor endpoints | `take_snapshot` in hot loop | Direct DOM only, zero snapshot calls |
| Group shows "not_member" | Need to join first | Use join flow (section 5 step 2) |
| Group shows "pending" | Admin approval required | Wait or try different group |
| 0 comment buttons found in group | Looking for wrong selectors | Use `[contenteditable="true"][role="textbox"]` with aria-label "comment" |
| Comment button click times out | Stale DOM after scroll | Re-query positions after scrolling |
| Messenger shows only 2 threads | Rail virtualization | Scroll rail container multiple times |
| `/messages/t/` redirects to login | Messenger needs fresh xs | Re-export cookies, use about:blank initial_url |
| initial_url facebook.com breaks auth | Cookie injection happens after session start | Use `about:blank` as initial_url |

---

## Environment Constants

- Anchor API key: `sk-77fea665b58ee8d7534828fa11a83a21`
- FB account: `c_user 61579001435313`
- Confirmed thread IDs: `1353482306539828`, `1030651092759166`
- Marketplace inquirers (need id from URL): Yulia Roshchina, Jp Hrez, Daniel García, Ralph Sidali, Enzo Quinteros
- CORS proxy: `https://cors.trigox.workers.dev?url=<encoded>`
- Skill repo: `https://github.com/samihalawa/fb-anchor-automation`

---

## NEVER DO

- Never use localhost:9222 / Comet / local Chrome
- Never use `take_snapshot` or vision-based loops
- Never spawn subagents for sequential FB work
- Never use `new_page()` in loops
- Never hardcode person names in message templates
- Never skip the 8s warm-up after homepage navigation
- Never set initial_url to facebook.com (use about:blank)
- Never look for "Comment"/"Comentar" span text to find comment buttons (use textbox aria-label)

---

## LEARNINGS FROM ALL PRIOR SESSIONS (forensic audit June 2026)

### What WORKED (proven in production)

1. **Anchor Browser + cookie injection with `sameSite:'None'`** — the only reliable remote CDP path. Worked across 6+ sessions in June 2026.

2. **`about:blank` as initial_url** — injecting cookies BEFORE any facebook.com navigation. Setting initial_url to facebook.com caused cookie injection to race and fail silently.

3. **8-second warm-up** on homepage after navigation — skipping caused immediate login redirects on /messages/ and other feature URLs.

4. **Comment textbox discovery via `[contenteditable="true"][role="textbox"]` with aria-label containing "comment"** — this is the ONLY reliable selector. Prior sessions wasted time trying span text matching ("Comment"/"Comentar") which finds elements inside existing comment threads, not the post-level comment action.

5. **`page.keyboard.insertText()`** for typing — handles unicode, emoji, and special chars. `page.type()` and `fill()` both fail on FB's contenteditable divs.

6. **Verification via `div[dir="auto"]` text scan** — checking if the first 30-35 chars of the sent message appear in any visible bubble. Simple and reliable.

7. **Direct URL navigation to threads** (`/messages/t/<id>`) — the ONLY reliable way to open a specific conversation. Rail clicks collapse to the currently-open thread due to virtualization.

8. **Group search → join → scroll → find textbox → comment** — the full pipeline worked end-to-end when run sequentially in a single Anchor session.

9. **`page.mouse.click(x, y)` by coordinates** — more reliable than Playwright locator clicks for FB's obfuscated DOM, especially for comment buttons.

### What FAILED (anti-patterns to avoid)

1. **Local CDP (localhost:9222 / Comet.app)** — cannot be reached from Claude's container. Even from macOS shell, the stale node process holding the CDP socket caused connection timeouts. Desktop Commander + start_process with recursive searches caused 4-minute hangs and connector unresponsiveness.

2. **`[role="row"]` for message extraction** — nests too deep in FB's DOM, always returns empty strings. Only `div[dir="auto"]` works.

3. **Span text matching for comment buttons** ("Comment", "Comentar") — these spans exist inside existing comment reply threads, not as the top-level post comment action. The actual comment entry is a textbox with aria-label.

4. **Rail click navigation** — virtualization makes every click resolve to the same currently-open thread. Must use direct URL.

5. **`take_snapshot` / vision loops in Codex** — triggers 429 rate limits from Anchor. Direct DOM queries only.

6. **`new_page()` in loops** — hits file descriptor ceiling. Reuse one page object.

7. **Subagent swarm for sequential FB work** — overhead, state confusion, "no regression keep going" false positives. Single linear driver only.

8. **Setting initial_url to facebook.com** — cookie injection happens after session creation but before navigation. If the session starts on facebook.com, the first load has no cookies.

9. **`playwright_fill()` on FB textboxes** — fails because FB's React controlled components intercept input events. Only `keyboard.insertText()` works.

10. **`playwright_click()` with CSS selectors on FB** — FB uses obfuscated class names that change between deploys. Overlay divs intercept pointer events (logged as `subtree intercepts pointer events`). Coordinate-based `page.mouse.click()` is more reliable.

11. **Bare curl to facebook.com for session validation** — returns HTTP 400 (FB edge rejects it). Need `--http1.1` plus full browser headers. `USER_ID: "0"` = logged out.

12. **Assuming groups allow immediate commenting after join** — some groups require admin post approval, or are "pending" (need admin to accept membership). The script must check group status after join click.

13. **FB search results for "habitacion madrid"** — returns sponsored/promoted content with broken DOM structure (Episode.living, Habyt ads). Not usable for finding real group posts to comment on.

### What was INEFFICIENT (optimize in future)

1. **Multiple Anchor sessions per task** — each session burns one of the ~6 daily sessions. Combine chat listing + commenting in a single session.

2. **Scrolling without re-querying positions** — after scroll, element positions change. Must re-evaluate DOM positions before clicking.

3. **Hardcoded group IDs that don't exist** — the initial group IDs (1200515470132498, 578525122338402, 1584498498498498) were fabricated, wasting 3 navigation attempts. Always discover groups via search first.

4. **Not joining groups proactively** — the account wasn't a member of any rental groups. The skill should maintain a list of known-good joined groups and try those first.

5. **Cookie re-export friction** — user has to manually export cookies via Cookie-Editor. A future improvement would auto-decrypt from Chrome's cookie store using the Keychain key (proven working in the June 22 session).

### Prior approaches that exist but are NOT part of this skill

- **messagix (Go library)** — headless Messenger via MQTT protocol. Lower ban risk than browser for messages. Built as `fb-cli` in `/Users/samihalawa/git/PROJECTS_ON_PROCESS/fb-cli/` (commit d748cbd). Returns 8+ threads. Good for message-heavy workflows but NOT for group commenting or marketplace listings.

- **mautrix-meta** — full Matrix bridge over messagix. Rejected: requires Synapse homeserver. Too heavy.

- **GeorgiKeranov/facebook-marketplace-bot** — Python cookie-login CSV repost. Only covers marketplace listings, not comments/groups.

- **browser-use / midscene** — generic browser automation. Midscene uses Chrome extension (vision-based). browser-use is headless agentic. Both work but are not FB-specific. This skill uses Anchor Browser + Playwright directly instead.

### Auto-cookie-extraction (proven, optional enhancement)

If the user can't export cookies manually, decrypt from Chrome's cookie store:
```python
# macOS only — requires Keychain access (user clicks "Allow")
import subprocess, sqlite3, hashlib
key_pw = subprocess.check_output(['security','find-generic-password','-ws','Chrome Safe Storage']).strip()
key = hashlib.pbkdf2_hmac('sha1', key_pw, b'saltysalt', 1003, 16)
# ... decrypt encrypted_value from ~/Library/Application Support/Google/Chrome/Default/Cookies
# Filter: host_key LIKE '%facebook%' AND name IN ('c_user','xs','datr','sb')
```
This is NOT part of the Anchor Browser flow (cookies go to Anchor, not local Chrome), but useful for bootstrapping when the user says "just use my Chrome cookies."
