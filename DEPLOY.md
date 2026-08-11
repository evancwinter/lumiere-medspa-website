# Deploying the Lumière demo

Lumière is a sales demo for Winter Growth Systems — not a real clinic.
It lives at **https://lumiere.wintergrowthsystems.com**.

## Where it's hosted

| | |
|---|---|
| Host | Cloudflare Pages |
| Project name | `lumiere-medspa` |
| Cloudflare account | `evancwinter@gmail.com` (id `e612b8cbcb220229f071ba6662b8e582`) |
| Deploy source | `Landing page prototype review/deploy/` |
| DNS | CNAME at Namecheap: `lumiere` → `lumiere-medspa-2o6.pages.dev` |

Note the `-2o6` suffix on the pages.dev hostname — it is **not** `lumiere-medspa.pages.dev`.
Cloudflare appended it because the plain name was taken. The *project* is
`lumiere-medspa`; only the hostname carries the suffix.

There is **no auto-deploy**. Pushing to GitHub does not publish anything.
The site only changes when someone runs the command below from this machine.

## Deploy

```bash
npx wrangler pages deploy "Landing page prototype review/deploy" \
  --project-name=lumiere-medspa --branch=main
```

Wrangler prints a unique `https://<hash>.lumiere-medspa.pages.dev` URL.
**Check that URL first.** Cloudflare's edge serves stale content on the custom
domain for up to a minute after a deploy, so testing
`lumiere.wintergrowthsystems.com` too early gives a false negative and sends you
debugging a problem that isn't there.

Don't append `?cachebuster=` to static assets — Pages can respond with
`index.html` instead of the asset and the result is confusing.

### Two things that look like bugs and aren't

**`support.js` byte size differs between the repo and the served file.** Git stores
LF line endings and checks out CRLF on Windows, so the same file measures ~1,911
bytes larger on disk (one byte per line). Compare with
`diff <(tr -d '\r' < a) <(tr -d '\r' < b)` before concluding anything changed.

**`index.html` differs between `pages.dev` and the custom domain.** Cloudflare's
Email Address Obfuscation runs on the zone, not on `pages.dev`. It rewrites the
`mailto:hello@lumierechs.com` link into a `/cdn-cgi/l/email-protection` link and
injects a small decoder script. Harmless and expected.

### Caching — why it's set the way it is

Every file whose **name stays the same while its contents change** is set to
`max-age=0, must-revalidate` in `_headers`: `index.html`, `support.js`,
`widget.css`, `elise.svg`, `margot.svg`, `chime.wav`. A deploy therefore takes
effect immediately for everyone, and you never debug a change that already
shipped.

This costs almost nothing. `max-age=0, must-revalidate` does not mean "download
every time" — the browser asks whether the file changed and Cloudflare answers
`304 Not Modified`, a few hundred bytes with no transfer.

`uploads/*` is the exception and keeps `max-age=31536000, immutable`. That's
where the real weight is (~37 MB of photos) and it's safe because Claude Design
gives every image a unique generated filename, so a changed photo is always a new
URL. Don't extend `immutable` to anything with a stable filename.

**Shortening a cache header does not un-cache what's already out there.** A browser
that fetched `widget.css` while it still said `max-age=14400` keeps that copy for
its full four hours; the new header only governs fetches made after the change.
This bit during the 2026-08-11 verification: production was serving the correct
file, but a browser that had visited earlier kept loading the old one, which
looked like a failed deploy. Confirm against a `pages.dev` deployment URL you
haven't visited before, or hard-refresh (Ctrl+Shift+R). Check what the widget
actually parsed, not what the server sends:

```js
const sr = document.getElementById('voiceflow-chat').shadowRoot;
const s = [...sr.styleSheets].find(x => (x.href||'').includes('widget.css'));
s.cssRules.length;   // stale copy had 6 rules, current has 7
```

Fixed 2026-08-11. Before that, `/support.js` sat at 24 hours while `/` was zero,
and `widget.css` plus the two avatars weren't listed at all so they inherited a
silent 4-hour Cloudflare default — the failure mode being a widget tweak that
looks like it didn't deploy, redeployed twice, still "broken", for four hours.
Content-hashed filenames were considered and rejected: they'd be optimal, but
they add a rename step to every export swap, and correctness that depends on
remembering a step is worse than a round trip nobody notices.

## Re-exporting the design — read this first

The page is a Claude Design export (`index.html` + `support.js` + `uploads/`).
`support.js` is the runtime; the page renders blank without it, and the two are a
**matched pair** — always replace `index.html` and `support.js` together.

A fresh export **does not contain the chatbot**, which is the entire point of the
demo. Seven things are hand-made and must survive any re-export:

1. **The Voiceflow block** — ~90 lines at the bottom of `index.html`, just before
   `</body>`. Contains the widget embed (project `6a6ca35d0e67e8f338790c3b`) and
   `lumiereNudge()`.
2. **`widget.css`** — Lumière colors on the widget, plus the red "1" unread badge.
   Loaded by absolute URL from the Voiceflow config, and injected into the
   widget's shadow DOM. Targets Voiceflow's own class names (`.vfrc-*`), so page
   redesigns can't collide with it.
3. **`chime.wav`** — the notification sound, fetched by absolute URL.
4. **`elise.svg`** and **`margot.svg`** — chat avatars. Nothing in this repo
   references them; the **Voiceflow dashboard points at them by URL**. If they
   stop being served at `/elise.svg` and `/margot.svg`, the avatars 404 and Elise
   loses her face with no error anywhere.
5. **`_headers`** — cache and security headers for Pages.
6. **`<title>`** — Claude Design does not emit one. Without it the browser tab
   shows a bare URL, which looks unfinished in a demo.
7. **`<meta name="description">`** — same; needed for link previews.

Items 2–5 live as separate files in the deploy folder, so they survive as long as
you replace only `index.html`, `support.js`, and `uploads/` and leave the rest
alone. Items 1, 6, and 7 live *inside* `index.html` and must be re-pasted by hand
every single time.

## After deploying, verify

- Page loads on the `pages.dev` hash URL, then on the custom domain.
- `/elise.svg`, `/margot.svg`, `/widget.css`, `/chime.wav` all return 200.
- The launcher appears bottom-right and nothing in the layout covers it —
  **check a phone viewport**, not just desktop. The design has a **mobile sticky
  CTA bar** (`showMobileBar`, full-width, `position:fixed;bottom:0;z-index:40`,
  roughly 80px tall) that appears below 860px once you scroll. It shares the
  bottom-right corner with the launcher. Any future design change that alters
  that bar's height is a launcher-collision risk.
- **The 4-second nudge fires**: chime, red "1" badge on the launcher, proactive
  bubble ("Question about treatments or pricing?"), launcher reading "Reply to
  Elise". It runs **once per browser session** — use a fresh incognito window to
  test it again, or clear the `lumiere-nudged` key from sessionStorage.

## Reverting

Every design change is committed on its own. To roll back, find the previous
commit with `git log --oneline`, restore the deploy folder from it, and redeploy:

```bash
git checkout <commit> -- "Landing page prototype review/deploy"
npx wrangler pages deploy "Landing page prototype review/deploy" \
  --project-name=lumiere-medspa --branch=main
```
