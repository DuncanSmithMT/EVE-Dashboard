# EVE Dashboard

A personal, phone-friendly dashboard for EVE Online characters. One place to see what all of your characters are doing.

**Live site:** https://duncansmithmt.github.io/EVE-Dashboard/

## What it shows (Phase 1)

### Overview
- **Needs attention** strip: characters that must sign in again, stopped (paused/empty) skill queues, queues ending in under 24 hours, accounts whose Omega ends within 7 days, characters with free SP.
- **Totals:** total ISK across all characters, how many are training, total free SP.
- **One row per character:** portrait, name, what's training, live countdown to queue end, wallet, system · ship, corporation. On a phone the row shrinks to name, status dot, queue time and wallet.
- **View menu** (the sorted views): grouped by account (default; with each account's Omega countdown), queue ends soonest, wallet highest first, name A–Z. The choice is remembered.

### Character detail
Click/tap a character. On desktop it opens as a side panel; on a phone it's a full screen with a back link.
- Account label, Omega countdown, free SP
- Current skill with progress bar, time left on the skill and the whole queue, and every queued skill with its finish countdown
- Wallet, system, docked station/structure (or "In space"), ship type and name, corporation
- Remove from this device

### Accounts page
The **Accounts** button opens a page to:
- type each character's account name (suggestions from names already used), and
- set an Omega expiry date per account.
Characters without an account go under **Unassigned** at the bottom of the overview.

### Refreshing
Data refreshes from EVE 5 minutes after the last refresh while the page is visible, and when you reopen it (if it's been over 2 minutes). Nothing is fetched while the page is hidden. When a skill finishes, the page updates itself without asking EVE. **⟳ Refresh** forces a refresh. A temporary EVE/network error keeps showing the last good data and retries; only a real login problem asks you to sign in again. See **How much the app asks EVE for** below.

## How it works

- The whole app is a single file, `index.html`, hosted free on GitHub Pages. There is no server.
- Characters sign in through EVE's official login (EVE SSO) using the browser-only PKCE method, so no client secret is needed or used.
- Login tokens are stored only in the browser of the device you signed in on (browser local storage). They are never sent to GitHub.
- Character data is fetched live from EVE's API (ESI) every time and is never saved. Also stored on the device: a cache of public names (skills, ships, systems, stations, corporations, structures), your account labels and Omega dates, and your chosen view.

## How much the app asks EVE for (ESI usage)

Audited 2026-11-13 against EVE's published per-route limits (the `x-rate-limit` entries in ESI's OpenAPI spec) and live responses. EVE charges "tokens" per call: 2 for a normal answer, 1 for "not changed", 5 for a client error. Each character has its own allowance per app and per group, refilling over 15 minutes.

| Data | EVE group | EVE allowance (per character) | EVE cache time | App use with the page open |
|---|---|---|---|---|
| Skill queue + skills | `char-detail` | 600 tokens / 15 min | 60 s | 2 calls every 5 min ≈ 12 tokens / 15 min (2%) |
| Wallet | `char-wallet` | 150 tokens / 15 min | 120 s | 1 call every 5 min ≈ 6 tokens / 15 min (4%) |
| Location + ship | `char-location` | 1200 tokens / 15 min | 5 s | 2 calls every 5 min ≈ 12 tokens / 15 min (1%) |
| Public character info (corporation) | none (error limit only) | — | 24 h | Repeats are served from the browser's cache; ~1 real call per character per day |
| Names (skills, ships, systems…) | none | — | — | Only IDs not seen before; stored on the device |

**Totals for 28 characters:** about 140 calls every 5 minutes (≈0.5 per second) while the page is open and visible; **zero while it's hidden** or closed. Each open window/device polls on its own.

Rules the app follows:
- **Refresh timing:** 5 minutes after the previous refresh *finished* (EVE asks apps not to fire on a fixed clock), never while the page is hidden. On reopening, it refreshes if the last sync was over 2 minutes ago. A tab opened in the background waits until it's looked at.
- **Finished skills:** the page just redraws; nothing is sent to EVE.
- **Spread out:** characters load 4 at a time.
- **Caching:** the browser honours EVE's cache times (verified: repeating a request before it expires costs nothing and never reaches EVE).
- **Backing off:** on 420 (error limit), 429 (rate limit) or 502–504 (EVE down), or when 10 or fewer errors remain in EVE's error window, **all** calls stop until EVE's stated reset time (60 s if it doesn't say). The header shows "Paused: … Resuming h:mm".
- **No repeated failures:** a route refused with 403 for a character isn't asked again that session; a character whose login failed isn't loaded again until it signs in again; names EVE doesn't know are remembered as "Unknown".
- **Identifies itself:** sends `X-User-Agent: EVE-Dashboard/1.1 (+repo URL)`, as EVE's best practices ask (browsers can't set `User-Agent`).
- Login token refreshes go to EVE's login service, not ESI: about one per character every 20 minutes while in use.

## Setup reference

| Item | Value |
|---|---|
| Developer portal | https://developers.eveonline.com/applications |
| Application name | Tempest's EVE Dashboard |
| Client ID | `28bb45097b894b8fa8885427fd253f28` (public by design) |
| Callback URL | `https://duncansmithmt.github.io/EVE-Dashboard/` (must match exactly, including capitals and the trailing `/`) |
| Client secret | **Not used. Never put it in this repo or share it.** |

## Adding characters

### Known issue: "Login failed. Your login session has expired"

**Symptom:** Clicking **+ Add character** while *not already signed in* to EVE's website leads to EVE's login page, and after entering your password EVE shows: *"Login failed. Possible reasons can be: Your login session has expired. Please try again."*

**Suspected cause:** The dashboard requests all 75 permissions (scopes) at once, which makes the login request about 2,700 characters long. When you are not already signed in, EVE's login page has to carry that whole request through the password step, and it appears to lose it, so EVE treats the login as expired. When you are already signed in, EVE skips the password step and the request goes through. The problem started after the permission list was expanded, which supports this explanation. It hasn't been formally confirmed.

**Workaround (works reliably):**
1. Sign in at https://login.eveonline.com first, in the same browser.
2. Open the dashboard and click **+ Add character**. EVE goes straight to character selection.
3. Pick the character and click **Authorize**. Repeat for each character on that account.
4. For characters on another EVE account: use **Switch user** (or sign out at login.eveonline.com), sign in with the other account, and repeat steps 2–3.

This is only needed when adding characters. Once a character is added, the dashboard keeps it signed in automatically.

**Possible permanent fix (not done):** request fewer permissions per login, e.g. only what each module needs. Trade-off: characters would need to re-authorize whenever a new module needs new permissions. Not worth it at current usage.

### Each device keeps its own character list
Logins are stored per browser, per device. Characters added in one place do not appear in another:
- Chrome on the PC, the Claude app's browser pane, and the phone each need their own one-time sign-ins.
- **iPhone:** a home-screen app has its own storage, separate from the browser. Add the site to the home screen **first** (Chrome or Safari → Share → Add to Home Screen), then add characters from the home-screen icon.
- **Android:** the home-screen app (⋮ → Add to Home screen / Install app) shares Chrome's storage, so order doesn't matter.

### Removing a character
- **Remove from this device** (in the character's detail view) only forgets it on that device.
- To fully revoke the app's access, use **third-party applications** in your EVE account settings. Deleting the application in the developer portal cuts off every character at once.

## Permissions (scopes)

The app requests **all 75 scopes** enabled on the application, so future modules work without re-signing characters.
- **67 information-only scopes** (read access, plus `publicData`, structure search/markets, and `manage_planets`, which despite its name only reads planetary colonies): skills, clones, wallet, assets, industry, mining, markets, contracts, mail, location, notifications, fittings, killmails, planets, corporation data (corporation scopes only return data if the character holds the relevant corporation roles), and more.
- **8 action scopes**, approved by the owner on 2026-11-13. None are used by the app yet:

| Scope | Allows |
|---|---|
| `esi-ui.open_window.v1` | Open market/info/contract/mail windows in the running client |
| `esi-ui.write_waypoint.v1` | Set destination / add waypoints |
| `esi-calendar.respond_calendar_events.v1` | Accept or decline calendar events |
| `esi-fleets.write_fleet.v1` | Manage fleet members and the fleet message (when fleet boss) |
| `esi-fittings.write_fittings.v1` | Save or delete saved fittings |
| `esi-mail.organize_mail.v1` | Mark read, label, delete mail |
| `esi-characters.write_contacts.v1` | Add, edit, delete contacts and standings |
| `esi-mail.send_mail.v1` | Send EVE mail as the character |

No scope lets anyone move ISK or items, trade, log into the game, or change account passwords.

**Adding a new scope later:** tick it on the developer portal, add it to `SCOPES` in `index.html`, then re-add each character on each device.

## Security model

- **The public repo contains only code and the Client ID.** No passwords, tokens, or character data.
- **Logins are tied to the registered Callback URL.** EVE only sends logins back to this site, so a copy of the code elsewhere can't capture your tokens.
- **Main risks:** someone gaining control of the GitHub account (could change the code), or someone using an unlocked device that has characters added.
- **Protections in place:** two-factor authentication on GitHub and on the EVE accounts, private email on GitHub commits, and all on-screen text escaped against code injection.

## Troubleshooting

| Problem | Fix |
|---|---|
| "Login failed… session has expired" | See the workaround under **Adding characters**. |
| Character shows "Sign in again" | Open the character and click **Sign in again** (or + Add character). |
| Character shows "Couldn't load, will retry" | Temporary EVE or network problem; it retries automatically. |
| Changes not showing after an update | Hard-refresh with Ctrl+F5. GitHub Pages can take 1–2 minutes to publish. |
| Claude app's browser pane is white | The pane is hidden. Press Ctrl+Shift+B (Cmd+Shift+B on Mac) or click the globe icon in the side panel. |
| Login works but error about state mismatch | Start again from **+ Add character**. Don't reuse an old EVE login tab. |
| Header says "Paused: …" | EVE asked apps to slow down, or is having problems. The app stops all calls and resumes on its own at the time shown. |

## Updating the site

1. On GitHub: **Add file → Upload files**, drag in the new `index.html` (same name replaces the old one), **Commit changes**.
2. Wait 1–2 minutes, then hard-refresh (Ctrl+F5).

## Roadmap

**Scale to design for:** 10 EVE accounts, 28 characters (one account has 2 open character slots that will likely be filled).

### Planned capabilities and data availability

| # | Capability | Source | Notes |
|---|---|---|---|
| 1 | Skill queues | ESI | ✅ Built |
| — | Layout B + sorted views | — | ✅ Built (Phase 1) |
| 2 | Industry jobs | ESI | Manufacturing, research, invention, reactions; "ready to deliver" alerts |
| 3 | Red / blue jump timers | ESI + calculation | Red = jump fatigue expiry (direct). Blue = jump activation cooldown, **estimated** from last jump and fatigue; shown as "approx." |
| 4 | Wallet balance | ESI | ✅ Built (Phase 1): per character plus total |
| 5 | Market orders | ESI | Buy/sell, location, expiry |
| 6 | Jump clone cooldown | ESI + calculation | Last clone jump time + cooldown length from the Infomorph Synchronizing skill |
| 7 | Character location | ESI | ✅ Built (Phase 1): system, station/structure, ship. Online status not yet |
| 8 | Installed implants | ESI | Active clone (jump clone implants also available) |
| 9 | Free skill points | ESI | ✅ Built |
| 10 | Account name / grouping | **Manual entry** | ✅ Built (Phase 1). Not available from ESI (CCP: account-level information is not exposed). See below |
| 11 | Omega status and expiry | **Manual entry** (+ inference) | ✅ Expiry date entry built (Phase 1). Omega vs Alpha inference from training speed not built |
| 12 | Corporation + history | ESI (public) | ✅ Current corporation built (Phase 1). History not yet |
| 13 | EVE mail read / write | ESI | Read first; writing (compose, recipient search, send) is the largest item and is planned as its own later step |

### Manual data: account labels and Omega expiry

ESI has no account-level data, so you enter it yourself on the **Accounts** page:
- **Account label:** assign each character to an account name of your choosing (e.g. "Main", "Industry"). The dashboard groups characters by this label.
- **Omega expiry date:** entered once per account; the dashboard counts down and warns as it approaches. Must be updated by hand when you renew.
- **Where it's stored:** in that device's browser, like logins, so it must be entered on each device (or copied with an export/import feature if that becomes worth building).
- Agreed by the owner on 2026-11-13.

### Layout

**Decision (2026-11-13): Option B (overview + detail), plus Option C's sorted views** as sort/filter controls on the overview (e.g. sort all characters by queue end, industry ready, wallet, fatigue).

Options that were considered:

Three layout options were mocked up for desktop and phone on 2026-11-13 (Claude design canvas "EVE Dashboard layout options"):
- **A. Grouped cards:** accounts as sections, a full card per character.
- **B. Overview + detail:** "needs attention" strip, totals, one compact row per character grouped by account; tap/click a character for full details. (Recommended.)
- **C. Topic tabs:** Skills / Industry / Market / Timers / Wallet / Clones / Mail, each listing all characters.

### Build phases
1. ✅ **Phase 1:** layout B + sorted views, account labels and Omega dates, wallet, location/ship, corporation.
2. **Phase 2:** industry jobs, jump clone cooldown, red/blue jump timers, implants; "industry ready" and "fatigue" sorts.
3. **Phase 3:** market orders, EVE mail (read).
4. **Phase 4:** EVE mail compose (write).

## Changelog

- **2026-11-13:** First release: skill queue module. Reliability fixes from code review (iOS-safe login storage, single token refresh at a time, auto re-sync when a skill finishes). Expanded to all 75 scopes. Documented the login workaround. Captured the full capability list, manual account/Omega entry decision, and three layout mockups. Chose layout B + C's sorted views. **Phase 1 built:** overview/detail layout, sorted views, Accounts page (labels + Omega dates), wallet, location/ship, corporation; staggered loading; transient errors no longer ask to sign in again. **ESI usage audit:** verified EVE's per-route limits and cache behaviour; no polling while hidden; refresh 5 min after the last one finishes; skill completion redraws locally; pause on 420/429/502–504 or near the error limit; don't repeat refused calls or failed logins; X-User-Agent header.
