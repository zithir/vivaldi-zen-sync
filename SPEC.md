# vivaldi-zen-sync — Specification

## Background

The browser setup this project serves:

- **Desktop:** Zen (Firefox-based) is the primary browser.
- **Mobile:** Vivaldi on Android is the primary browser, because password-manager autofill (Proton Pass) works reliably in Chromium-based browsers and not in Firefox on Android.
- Vivaldi Sync provides end-to-end-encrypted, full two-way sync of **bookmarks and complete browsing history** between desktop Vivaldi and Android Vivaldi (since Vivaldi 6.5).
- No cross-browser tool exists that syncs **history** between Firefox-based and Chromium-based browsers. Bookmarks have partial answers (floccus and friends), but none that fit this architecture, in which the mobile leg is handled by Vivaldi Sync and only a desktop-to-desktop bridge is missing.

## Problem statement

A user browses on Android (Vivaldi) and on desktop (Zen). Bookmarks and history exist in two disconnected silos:

1. The Vivaldi silo covers phone + desktop Vivaldi via Vivaldi Sync.
2. The Zen silo covers the desktop Firefox ecosystem alone — Zen has no mobile counterpart.

Anything visited or bookmarked on the phone never reaches Zen, and anything bookmarked in Zen never reaches the phone. Firefox Sync cannot cover the mobile leg because making it work would mean using Firefox on Android, where the password-manager experience is broken.

Vivaldi on desktop therefore becomes the natural **hub**: it is already kept in sync with the phone by Vivaldi Sync. The missing piece is a local desktop bridge between Vivaldi's profile data and Zen's profile data — with no cloud service, server, or third party beyond Vivaldi's encrypted sync.

## Goal

Keep **bookmarks** and a **re-visitable record of visited URLs** consistent between Zen and Vivaldi on macOS, fully automatically in the background, with the user never having to launch, close, or otherwise operate either browser manually. Vivaldi Sync remains the sole transport for the phone leg.

## Scope

### In scope

- **Bookmarks**: two-way sync between Vivaldi and Zen on macOS, preserving folder structure as far as both models allow; deletions must propagate; conflicts must resolve deterministically and without data loss.
- **History**: two-way propagation of *visited URLs* at **re-visit fidelity** (defined below). A URL visited in either browser (or on the phone, via Vivaldi Sync) must become discoverable in the other browser's history surface (history page and address bar), so that the user can navigate there again.
- **Automation**: the whole cycle runs unattended on macOS, on a schedule, including hidden (background) launch of Vivaldi when needed to exchange data with Vivaldi Sync.
- **Platforms**: macOS; Zen and Vivaldi desktop only. Mobile is out of the bridge's hands by design (Vivaldi Sync covers it).

### Out of scope (non-goals)

- Passwords, cookies, sessions, tabs, extensions, settings, themes.
- Faithful history semantics: exact visit timestamps, visit counts, transition types, redirect chains, per-visit records, and history *deletion* propagation are explicitly **not** goals.
- Real-time or streaming sync. Periodic eventual consistency is sufficient.
- Firefox, Chrome, or any other browser; Windows and Linux; any mobile client.
- Any server, cloud, or multi-user operation. This is a single-user, single-machine (or single-user multi-Mac) tool.

## History fidelity requirement

The single user-facing job of history sync is: **"I visited this link somewhere; let me find it and go back to it from any of my browsers."**

What that intentionally gives up, and why it helps:

- No exact timestamps are needed. An imported URL may carry a coarse or approximate time, or simply the time of import.
- No per-visit multiplicity is needed. History can be treated as a **set of URLs** (plus title at last sight), not a multiset of visit events.
- No deletion propagation is needed. If a URL disappears from one side's native history (manual delete or browser expiry), the bridge does not chase it.

Consequences the design should exploit:

- Deduplication becomes **set membership**, not event matching, so URL canonicalization makes sync *more* correct rather than fighting exact-visit semantics.
- Re-import of an already-known URL is a no-op, which makes idempotency easy to guarantee.
- Imported rows may inherit current time as their visit time; ordering in the receiving browser does not need to reflect the original timeline.

## Constraints and invariants

1. **Never corrupt a browser profile.** Both browsers' data stores are touched directly. Every write cycle must be preceded by a backup and validated by integrity checks; on any doubt, abort without writing.
2. **Never write to a store while its browser is running.** Both browsers hold authoritative in-memory state and SQLite locks; writes made against a live browser are lost or dangerous. The bridge schedules writes only for closed targets and queues pending work otherwise.
3. **Never interfere with an active session.** If the user is actively browsing (especially in Zen), the bridge must not quit, restart, or visibly disturb the browser. Work waits.
4. **Idempotence.** Running the same cycle twice must not create duplicate bookmarks or history entries — including across directions, where entries the bridge itself imported must never be re-exported back (loop prevention).
5. **Deterministic bookmark conflict resolution.** Same bookmark edited on both sides, or created on both sides: outcome must be predictable and never silently discard user data.
6. **Privacy.** No data leaves the machine through this tool except what Vivaldi Sync already handles, encrypted, for the mobile leg.

## Technical challenges

The problem is not hard because of volume or algorithms; it is hard because two independent applications consider the same files *their* private state.

### 1. Two totally different data models

- Vivaldi (Chromium): history is a SQLite database (`History`, tables `urls`/`visits`); bookmarks are a JSON file (`Bookmarks`); timestamps are microseconds since 1601-01-01.
- Zen (Firefox): history *and* bookmarks share one relational store (`places.sqlite`: `moz_places`, `moz_historyvisits`, `moz_bookmarks` as a nested tree); timestamps are microseconds since 1970-01-01.

Every field mapping (URL, title, visit time, folder, position) crosses a representation gap, and both schemas evolve with browser updates. The bridge must tolerate and detect schema drift instead of silently writing wrong rows.

### 2. Concurrency with the browsers

The browsers are not aware that anyone else touches their databases. They cache in memory, hold locks, flush on their own schedule, and routinely treat "unrecognized" on-disk changes as corruption or simply overwrite them. The bridge therefore lives inside a window problem: *when both data stores are safe to read and at least one is safe to write*. On top of that, read-only snapshots of a live database (WAL and journal siblings) must be taken in a way that produces a consistent copy.

### 3. A sync dependency that cannot be heard from

Vivaldi only exchanges data with the phone while it is running, and **the sync engine gives no observable "sync finished" signal**. The bridge must launch Vivaldi headlessly, decide when it is safe to snapshot (settle heuristics or timeouts), and shut it down safely — without ever racing the user's own real Vivaldi session.

### 4. Loop prevention and idempotency

Every row the bridge imports exists in two stores plus (implicitly) in Vivaldi Sync, and will be re-read on the next cycle. Without its own authoritative ledger of "what I already imported and exported", the bridge re-imports its own data forever. The canonical-URL/set-semantics history model (see above) is the main mitigation; bookmarks have no equivalent dodge because they cannot be deduplicated by content alone — folder + URL + position all matter, and the same URL may be deliberately bookmarked twice.

### 5. URL canonicalization

The same page is recorded differently everywhere: `https://` vs `http://`, trailing slashes, `utm_` parameters, AMP variants, redirect chains (a visit often lands on a *different* URL than the one typed). The bridge needs a canonicalization rule that is aggressive enough to dedup meaningfully but conservative enough to never conflate genuinely different resources.

### 6. Bookmark tree merging

Bookmarks are a tree with indices, and the two stores order siblings differently. Two-way merge with deletion propagation over trees is the single most error-prone part of the project: risk scenarios include duplicated subtrees after conflicts, lost deletions resurrecting bookmarks, and position thrash (each side re-ordering the other). Off-the-shelf merge logic for exactly this pair of models does not exist to lift — floccus solves Chromium↔Firefox bookmark sync from inside the browser, against its own server model, but its approach is not directly reusable against raw local stores.

### 7. Making injected history actually surface

Importing a URL into a history store is not enough: the receiving browser's address bar ranks results by its own recency/frequency model (frecency in Firefox, visit signals in Chromium). Freshly injected rows can rank low and seem absent. The success criterion ("type a few characters, find the link") therefore depends partly on store internals the bridge does not control and can only mildly influence.

### 8. Failure safety as a first-class feature

The blast radius of a bug is "the user's entire browser profile." The bridge must assume its own bugs will happen: timestamped backups before every write, validation after every write, refusal to run against unexpected sizes/shapes of data, and a documented recovery path that a tired user can follow at 1 a.m.

## Success criteria

1. Visit a page on the phone → within one sync cycle, it is findable from Zen's address bar and history page.
2. Visit a page in Zen → within one cycle, it is findable in Vivaldi desktop (and thereby the phone).
3. Bookmark or move a bookmark in either browser → the change appears on the other side and on the phone, with folder placement intact.
4. Delete a bookmark in either browser → it disappears on the other side.
5. No manual browser launching, closing, or clicking, ever; nothing visibly interrupts normal browsing.
6. Zero profile corruption over months of unattended operation; any failed cycle leaves both stores exactly as they were, and restoration from bridge backups is a documented, tested path.
