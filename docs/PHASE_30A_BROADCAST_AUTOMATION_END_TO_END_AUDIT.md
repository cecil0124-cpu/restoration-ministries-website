# PHASE 30A — Broadcast Automation End-to-End Audit

**Date:** Thursday, November 19, 2026 (America/New_York)  
**Scope:** Diagnostic only — no code changes, no production deployments, no publishing-lock changes  
**Firebase project:** `rm-church-app`  
**Production website:** https://restorationministriescogic.org  

**Verdict:** **NO-GO** for automatic publication and featuring.  
**Primary confirmed blocker:** intentional production safety control — `BROADCAST_PUBLISHING_LOCK=disabled` with scheduler in report-only mode (logs `would-import` / `duplicate` / `proposed-replay`, writes `health.lastCheck`, creates **0** media records).  
**Manual path works:** “Keep Working” (sermon, featured) and “Encourage Yourself in the Lord Part 2” (bible-study, published, not featured) are live in `mediaMessages` and accepted by website/app published queries.

---

## 1. Repository and production baseline

### 1.1 Expected baseline (from Phase 30A brief / Phases 25–29)

| Asset | Expected |
|---|---|
| Website local tree | `/Users/macmini/Documents/ChatGPT/Albums/Broadcast-Automation-Development/website` |
| Firebase local tree | `/Users/macmini/Documents/ChatGPT/Albums/Broadcast-Automation-Development/firebase` |
| Branch | `work/broadcast-automation` |
| Website production commit | `8d7599e` |
| Vercel deployment | `dpl_XT111JRQgKLcrLe2N1URV8DSqtMm` |
| Firebase test commit | `cec18bf` |
| Functions | `checkLatestBroadcast`, `configureLatestBroadcast`, `commitMediaResources` |
| Scheduler | `*/30`, `America/New_York`, report-only |
| Lock | `BROADCAST_PUBLISHING_LOCK=disabled` (do not change) |

### 1.2 What this Cloud Agent environment actually has

| Check | Result | Classification |
|---|---|---|
| Mac Mini Broadcast-Automation-Development trees | **Not present** on this VM | **Baseline discrepancy** |
| Attached GitHub repo `cecil0124-cpu/restoration-ministries-website` | Clean `main` @ `a39a054` — static placeholder site only; **no** `work/broadcast-automation`, **no** `8d7599e` | **Baseline discrepancy** |
| GitHub `restoration-ministries-website-v2` | `main` @ `090690d`; has Media Manager + older `rm-automation/` (launchd YouTube draft checker); **does not** export the three Cloud Functions | Related but not the Phase 26 deploy tree |
| GitHub `restoration-ministries-app` | `main` @ `8f9cf0a`; `mediaMessages` client reads; **no** `checkLatestBroadcast` source | App consumer only |
| Commits `8d7599e` / `cec18bf` on GitHub | **Not found** in accessible remotes | Local-only (Mac Mini) per Phases 26–29N |
| Prior worker `bc-61676cbe` transcript | **Not accessible** from this environment | Evidence gap for raw function source |
| Firebase CLI auth | **None** (`firebase login:list` empty) | Cannot read Cloud Logging / Secret Manager / Scheduler API |
| Firestore emulator `:8080` / isolated `:8180`/`:9198` | **Not running** on this VM | Isolated Scenarios A–E against real functions **untested here** |

**Preservation note:** No existing local Broadcast-Automation working trees were modified (they are absent). Attached repo working tree was clean before this documentation-only branch.

### 1.3 Production live checks performed (read-only)

| Probe | Evidence |
|---|---|
| Site HTTP 200 | `https://restorationministriescogic.org/` and `/sermons` |
| Vercel | `server: Vercel`; CSP allows `https://us-central1-rm-church-app.cloudfunctions.net` |
| YouTube channel | `UCEN2tcDJC_CpaHYKbnveDlg` (“Restoration Ministries Cogic”) — matches Phase 26 |
| YouTube Atom feed | Videos include `i3g-ih3K8jQ` (“Keep Working”) and `OgEJWaaTRNA` (“Encourage Yourself in the Lord Part 2”) |
| Firestore anonymous published queries | Succeed for `status==published` (+ category / featured) |
| Callable presence | `configureLatestBroadcast` → `401 Sign in to manage broadcasts.`; `commitMediaResources` → `401 Sign in to manage attachments.`; `checkLatestBroadcast` → `403 Forbidden` (not a public callable / invoker-restricted) |
| Production Media Manager JS | Includes Broadcast Automation panel + `mediaVideoIds` reservation transactions + `commitMediaResources` / `configureLatestBroadcast` callables (consistent with website HEAD `8d7599e` behavior described in Phases 27–29, **not** with GitHub `website-v2@090690d`) |

**Lock:** Not re-read from Secret Manager in this environment. Prior Phase 26/29C evidence: secret value exactly `disabled` (8 bytes). **Not changed** in Phase 30A.

---

## 2. Current automation architecture

```
YouTube channel UCEN2tcDJC_CpaHYKbnveDlg
        │
        ▼
Cloud Scheduler (*/30 America/New_York)
        │
        ▼
checkLatestBroadcast  ──(report-only while lock=disabled)──► logs would-import|duplicate|proposed-replay
        │                                                     writes broadcastAutomation/health.lastCheck
        │                                                     NO mediaMessages create / feature
        │
        ├── configureLatestBroadcast (admin callable)
        │     • getStatus → effectiveMode publish|report + activationRequired
        │     • mode / specialIds / resume (clear pin)
        │     • reads/writes broadcastAutomation/control (+ health)
        │
        └── commitMediaResources (admin callable)
              • Media Manager attachment commits (not the YouTube detect path)

Manual Media Manager (production website admin UI)
        │
        ├── create/update mediaMessages + mediaVideoIds/{videoId} reservation (transaction)
        ├── feature → clears other featured; merges broadcastAutomation/control.pinnedId
        └── commitMediaResources for attachments
```

**Separate legacy path (GitHub `website-v2` `rm-automation/`):** Mac `launchd` `youtube-sunday-check.mjs` creates **draft-only** `mediaMessages` via client SDK. It is **not** the deployed Cloud Function pipeline and must not be confused with Phase 26 production functions.

**Collections involved**

| Collection / doc | Role |
|---|---|
| `mediaMessages/{id}` | Canonical sermon / bible-study / special-service records |
| `mediaVideoIds/{videoId}` | Duplicate reservation (`mediaId` ownership) |
| `broadcastAutomation/control` | Mode/pin/specialIds; Media Manager feature pin (`pinnedId`, `featuredId`) |
| `broadcastAutomation/health` | `lastCheck`, `lastImport`, `recentErrors` (admin UI) |

---

## 3. YouTube detection results

### 3.1 What can be confirmed about `checkLatestBroadcast`

| Question | Finding | Confidence |
|---|---|---|
| Channel monitored | `UCEN2tcDJC_CpaHYKbnveDlg` | **Confirmed** (Phase 26 + site YouTube link + feed) |
| How discovery works | YouTube Data API key secret `BROADCAST_YOUTUBE_DATA_API_KEY` (from Vercel `YOUTUBE_DATA_API_KEY`) | **Confirmed** (Phase 26) |
| Scheduler | `*/30` `America/New_York` ENABLED | **Confirmed** (Phases 26–29C relays) |
| Behavior with lock disabled | Report-only: `would-import` / `duplicate` / `proposed-replay`; `health.lastCheck` updated; **0** media created; featured unchanged | **Confirmed** (Phases 26, 28, 29C) |
| Sermon vs Bible Study rules inside Cloud Function | **Source not available** on this VM | **Unverified** for deployed function |
| Livestream vs upload vs scheduled handling | **Unverified** for deployed function | Analog only: `website-v2` `media-discovery.mjs` requires livestream start in Sunday 10:00–14:00 ET or Wednesday 18:00–22:00 ET |
| Duplicate handling | Logs `duplicate`; Media Manager uses `mediaVideoIds` + youtubeVideoId uniqueness | **Partial** — reservation path confirmed in prod JS; function duplicate algorithm unverified |
| Newest-only vs multi | **Unverified** for deployed function | — |

**Important:** Function source was never present in this Cloud Agent workspace and was not pasted into accessible prior transcripts. Do **not** treat the GitHub `rm-automation` classifier as proof of deployed Cloud Function behavior.

### 3.2 November 15 sermon / November 18 Bible study — detection vs publication

| Video | YouTube ID | Feed published (UTC) | Firestore published record | Automation-created? |
|---|---|---|---|---|
| Keep Working | `i3g-ih3K8jQ` | 2026-11-09T06:01:50Z | **Yes** — `mediaMessages/x20uwKFl4KNl0EcvLrDQ` | **No** — manual admin `AtPXXBlNbYgijoAxk0iNFYOe5Vx2` |
| Encourage Yourself in the Lord Part 2 | `OgEJWaaTRNA` | 2026-11-19T13:05:47Z | **Yes** — `mediaMessages/1dvMfLb5c0li7kplOuPi` | **No** — same admin UID; doc id reuses Phase 29 test id |

**Keep Working (`x20uwKFl4KNl0EcvLrDQ`) — verified Firestore fields**

| Field | Value |
|---|---|
| title | Keep Working |
| category | `sermon` |
| messageDate | `2026-11-15T17:00:00Z` (noon America/New_York on Nov 15, 2026) |
| speaker | Juan M Goodson |
| status | `published` |
| featured | `true` |
| createdAt | `2026-11-19T20:13:06.587Z` → **3:13:06 PM ET** |
| publishedAt | `2026-11-19T20:13:27.931Z` → **3:13:27 PM ET** |
| createdBy | `AtPXXBlNbYgijoAxk0iNFYOe5Vx2` |

**Encourage Yourself Part 2 (`1dvMfLb5c0li7kplOuPi`)**

| Field | Value |
|---|---|
| title | Encourage Yourself in the Lord Part 2 |
| category | `bible-study` |
| messageDate | `2026-11-18T17:00:00Z` (noon ET Nov 18, 2026) |
| speaker | Juan M Goodson |
| status | `published` |
| featured | `false` |
| createdAt | `2026-11-19T15:05:03.360Z` → **10:05:03 AM ET** |
| publishedAt | `2026-11-19T19:59:12.875Z` → **2:59:12 PM ET** |
| createdBy | `AtPXXBlNbYgijoAxk0iNFYOe5Vx2` |

**Critical continuity note:** Doc id `1dvMfLb5c0li7kplOuPi` is the same id Phase 29 used for the ARCHIVED Media Manager verification test (`TEST — Media Manager Verification`) with reservation `mediaVideoIds/OgEJWaaTRNA`. The Nov 18 Bible study was therefore published by **reusing/updating that document**, not by automation minting a new id.

**Detection logs for these two videos:** **Unverified** in Phase 30A — Cloud Logging requires authenticated Firebase/GCP access unavailable here. Prior production scheduler evidence only shows generic `would-import`/`duplicate`/`proposed-replay` outcomes while lock stayed disabled. **Do not infer detection solely because the videos exist on YouTube.**

**Related YouTube item not published:** `C792v8yRgEY` (“Jesus Flipped The Script”, feed 2026-11-16) — no published `mediaMessages` row with that `youtubeVideoId` (anonymous published+videoId query returned empty). Consistent with automation not publishing.

---

## 4. Production scheduler execution evidence

| Evidence | Source | Result |
|---|---|---|
| Scheduler enabled `*/30` America/New_York | Phase 26 Stage 1A / 28 / 29C relays | **Confirmed** (historical) |
| Report-only outcomes | Phase 26: “would-import/duplicate only… 0 media created… featured not updated”; Phase 28 9:30 AM ET; Phase 29C 11:00 AM ET `would-import/duplicate/proposed-replay`, “no imported” | **Confirmed** (historical) |
| `health.lastCheck` written | Phases 25–28 | **Confirmed** (historical) |
| Fresh Nov 19 2026 Cloud Logging pull | This environment | **Unavailable** (no Firebase login) |
| `checkLatestBroadcast` public HTTP | Probe | `403 Forbidden` (expected for scheduler/IAM-restricted function) |

---

## 5. Effective configuration

| Setting | Effective value | Stored / controlled by |
|---|---|---|
| Detection / scheduler | Enabled every 30 minutes ET | Cloud Scheduler → `checkLatestBroadcast` |
| Publishing mode (UI) | Report-only when lock disabled / activation required | `configureLatestBroadcast` + `broadcastAutomation/control`; UI states: `report`, `publish`, `activation-required` |
| Publishing lock | `disabled` (8 bytes) — **do not change** | Secret Manager `BROADCAST_PUBLISHING_LOCK` (Phase 26) |
| YouTube channel | `UCEN2tcDJC_CpaHYKbnveDlg` | Function config / secret-backed API key |
| YouTube API key | `BROADCAST_YOUTUBE_DATA_API_KEY` | Secret Manager |
| Duplicate prevention | `mediaVideoIds/{videoId}` + youtubeVideoId checks | Media Manager transaction (prod JS); automation path intended to respect same reservations (Phase 27+) |
| Approval requirements | Admin Google auth for Media Manager / callables | Firebase Auth + media admin rules; unauthenticated callables → 401 |
| Default category | **Unverified** in Cloud Function; UI allows sermon / bible-study / specialIds override | `broadcastAutomation/control.specialIds` via `configureLatestBroadcast` |
| Speaker mapping | Manual Media Manager field; automation historically leaves speaker empty in draft analogs | Media Manager vs (unverified) function |
| Message date | Manual noon-ET timestamps observed; discovery analogs use livestream start date | Media Manager / (unverified) function |
| Featured behavior | Manual Feature / Publish+featured; automation with lock off does **not** update featured | Media Manager transaction updates `featured` + `broadcastAutomation/control.pinnedId` |

Admin UI strings (production bundle): “Report-only. Automatic publishing is off.” / “Enable automatic publishing” confirms `mode: report|publish` is operator-controllable **subject to** lock / `activationRequired`.

---

## 6. Publishing-lock behavior

| Question | Answer | Evidence |
|---|---|---|
| Which functions check the lock? | Operationally gates `checkLatestBroadcast` write path; exact code branches **unverified** without source | Phase 25–26: Stage 1 must stay `disabled`, never `authorized-v1` |
| Prevents detection? | **No** — scheduler still runs; YouTube quota / checks still occur | Phase 26 report-only description |
| Prevents record creation? | **Yes** while disabled | “0 media created by broadcast-automation” |
| Prevents publication / featuring? | **Yes** for automation worker | Featured id remained `191QQ8wz2JLbhm9av9TR` through Phases 26–29 until **manual** Keep Working feature on Nov 19 |
| Report-only vs lock | UI `effectiveMode` can be `report`/`publish`; lock/`activationRequired` can force report-only even if prior publish settings exist | Prod UI: `activation-required` → “Report-only. Previous publishing settings require activation.” |
| Admin approval draft mode | Media Manager drafts are manual; automation draft-preparation under lock **not demonstrated** in production (creates 0 media) | Phases 26–29 |
| Can scheduler prepare drafts while publishing disabled? | **Not observed** — production runs log proposals only | Confirmed blocker for “auto draft” as currently deployed |
| Separate change for auto-publish? | **Yes** — unlock / authorize beyond `disabled` **and** set effective mode to `publish` without activationRequired; **not authorized in Phase 30A** | Phase 25 NO-GO for auto-publish |

**What production automation currently does**

- Runs on schedule  
- Consults YouTube  
- Writes health check timestamps / proposal logs  
- Does **not** create, publish, or feature `mediaMessages`

**What it does not do**

- Does not produce the same published+featured records as Media Manager  
- Does not replace manual Keep Working / Encourage Yourself workflows

---

## 7. Media record creation path

### 7.1 Manual Media Manager (production website — verified via live JS + Firestore)

1. Target collection: **`mediaMessages`** (auto ids; Keep Working → `x20uwKFl4KNl0EcvLrDQ`)  
2. Document ID: Firestore auto id on create  
3. Video ID: parsed from YouTube URL → `youtubeVideoId` + canonical `youtubeUrl`  
4. Title / description / speaker / scripture / series fields: form-mapped  
5. Message date: admin-entered → Timestamp (observed noon ET)  
6. Category: `sermon` | `bible-study` | (special-service)  
7. Status: `draft` on Save Draft; `published` on Publish  
8. Featured: only when `status==published` and featured checked / Feature action  
9. Duplicate reservation: transaction on `mediaVideoIds/{videoId}` with `{mediaId}`; throws “This YouTube video already has a Media Manager record.”  
10. Feature transaction: clears other featured docs; `broadcastAutomation/control` merge `{pinnedId}`; refuses unfeature/archive while pinned without clearing pin  
11. Attachments: `commitMediaResources` callable (not required for Keep Working, which has empty `resources`)  
12. Errors: status concurrency (`expectedStatus`), reservation conflicts, validation failures  

### 7.2 Automation path (intended / partially verified)

| Step | Status |
|---|---|
| Detect via `checkLatestBroadcast` | Scheduler exists; detection details unverified |
| Write `mediaMessages` | **Blocked** by lock/report-only in production |
| Intermediate workflow | Health/control docs — **not** a separate public collection for messages |
| Direct write vs Media Manager parity | **Unverified** without function source; Phase reports assert no creates while locked |

---

## 8. Manual versus automated record comparison

| Field | Manual “Keep Working” | Automation (current production) |
|---|---|---|
| Collection | `mediaMessages` | Would target `mediaMessages` (intended); **writes blocked** |
| Video ID | `i3g-ih3K8jQ` | Would reference same YouTube ids when unlocked (intended) |
| Title | Keep Working | Unknown mapping until source audited on Mac tree |
| Category | `sermon` | Unverified in function; must be `sermon` for sermon archive queries |
| Message date | Nov 15, 2026 noon ET | Unverified |
| Status | `published` | Not written by automation today |
| Featured | `true` | Not updated by automation today |
| Publication timestamp | Nov 19, 2026 3:13:27 PM ET | N/A |
| Duplicate reservation | `mediaVideoIds` (prod JS) | Unverified write while locked |
| createdBy | Admin UID | Would be function service account if unlocked (unverified) |

### Would an automated record be accepted by website + RM app queries?

**If and only if** the automation writes a document matching the public contract:

- `status == "published"`
- `category` ∈ `{sermon, bible-study, special-service}`
- non-empty `title`
- `youtubeVideoId` and/or `youtubeUrl`
- for homepage/Watch pin: `featured == true` (and Media Manager pin semantics via `broadcastAutomation/control` if used)

**Evidence**

- Website: `subscribeToPublishedMessages` / `subscribeToFeaturedPublishedMessage` in production Media stack; GitHub analog `website-v2` `src/lib/media/repository.ts` + `usePublishedMediaMessages.ts`  
- App: `fetchPublishedMediaPage` / `fetchLatestPublishedSermon` in `restoration-ministries-app/src/services/mediaMessages.js` filter `status==published` + category; `normalizeMediaMessage` rejects missing title/video/category  
- Live proof: Keep Working already satisfies these queries (featured sermon); Encourage Yourself Part 2 satisfies bible-study published queries without featuring  

**Gap:** Automation today does **not** write that document. Acceptance of a hypothetical unlocked payload is **conditionally yes** for query shape, but speaker/date/category/feature/pin parity remains **unverified** without function source + isolated tests against the real functions.

---

## 9. Sermon and Bible study handling

| Concern | Finding |
|---|---|
| Manual sermon path | Works end-to-end (Keep Working featured on both platforms per owner + Firestore) |
| Manual bible-study path | Works for publish-without-feature (Encourage Yourself Part 2) |
| Automation classification | Deployed rules **unverified**; GitHub `media-discovery.mjs` analog separates Sunday sermon vs Wednesday bible-study windows and never auto-publishes/features |
| Special services | Admin UI supports `specialIds` via `configureLatestBroadcast` |
| Category mixing | Series UI forbids mixing Sermon and Bible Study |

---

## 10. Isolated test results (Scenarios A–E)

**Environment reality:** No Firestore emulator on `:8080`/`:8180`, no Broadcast-Automation firebase tree, no function source, no Firebase auth. Therefore **the actual Cloud Functions were not executed in isolation**.

| Scenario | Result in Phase 30A | Notes |
|---|---|---|
| A — New Sunday sermon | **Untested** against Cloud Functions | Analog: `media-discovery` would `create-draft` for Sunday-window livestream; production lock would still block Cloud Function writes |
| B — New Bible study | **Untested** against Cloud Functions | Analog: Wednesday-window → `bible-study` draft |
| C — Existing video | **Partial (live data)** | Keep Working / Encourage already exist; duplicates would hit reservations / already-exists semantics |
| D — Publishing disabled | **Confirmed by prior production evidence** | Lock disabled → 0 media creates; featured not automation-updated |
| E — Invalid/incomplete | **Untested** against Cloud Functions | Analog: upload-time-only → `manual-review` / skip |

**Analog script executed** (GitHub `website-v2` `rm-automation/lib/media-discovery.mjs` only — **not** production functions): synthetic Keep Working / Encourage Yourself / Jesus Flipped classifications and duplicate short-circuit when existing map populated.

**Portion remaining untested:** end-to-end `checkLatestBroadcast` → (draft|publish) → `mediaVideoIds` → feature/pin against isolated emulators using the **real** deployed source at `cec18bf` / `work/broadcast-automation`.

---

## 11. Confirmed blockers

| Blocker | Class | Confirmed / Suspected | Notes |
|---|---|---|---|
| `BROADCAST_PUBLISHING_LOCK=disabled` | **Intentional safety control** | **Confirmed** | Blocks automatic media create/publish/feature |
| Scheduler report-only outcomes | **Intentional safety control** | **Confirmed** | `would-import`/`duplicate`/`proposed-replay` only |
| Auto-publish repeatedly NO-GO in Phases 25–29N | **Intentional safety control** | **Confirmed** | Policy, not a silent crash |
| Cloud Function / website source only on Mac Mini trees | **Missing functionality (for this audit env)** / process gap | **Confirmed** | Cannot re-verify source or run isolated function tests here |
| No Firebase/GCP auth in Cloud Agent | **External service limitation** (credential) | **Confirmed** | No fresh logs / secret / scheduler API reads |
| `activationRequired` UI state | **Configuration issue** (possible) | **Suspected** | Can keep UI in report-only even with prior publish settings |
| Exact sermon/bible-study eligibility in deployed function | — | **Unverified** | Need Mac firebase tree + logs |
| Whether unlocking alone enables drafts vs full publish | — | **Unverified** | Need source of lock branches |
| Reuse of test doc `1dvMfLb5c0li7kplOuPi` for real Bible study | **Data-integrity risk** | **Confirmed** as fact; impact **suspected** | Reservation/history coupling with Phase 29 test |

### Safe operating modes

| Mode | Currently safe? |
|---|---|
| 1. Detection and reporting only | **Yes — this is production today** |
| 2. Automatic draft preparation + admin approval | **Not demonstrated** under current lock (0 creates). Requires explicit Stage design + tests before enablement |
| 3. Fully automatic publication + featuring | **NO-GO** — safeguards for category/date/speaker/feature/pin/duplicates not re-proven against current YouTube items in an isolated harness here |

---

## 12. Security and data-integrity risks

1. **Unlocking publish without isolated replay** could create wrong category/date or feature the wrong message.  
2. **Feature/pin coupling:** Media Manager writes `broadcastAutomation/control.pinnedId`; clearing automation pin vs manual Feature must stay consistent.  
3. **Duplicate reservations:** `mediaVideoIds` must remain authoritative; automation must not bypass.  
4. **Test record reuse:** Publishing real content into former QA doc `1dvMfLb5c0li7kplOuPi` / video `OgEJWaaTRNA` blurs audit trails.  
5. **Callable auth:** Unauthenticated access correctly denied (`401`/`403`) — keep that posture.  
6. **Secret lock:** Keep `BROADCAST_PUBLISHING_LOCK=disabled` until a dedicated authorization phase.  
7. **Repo split risk:** GitHub `website-v2@090690d` ≠ production website `8d7599e` automation UI — future edits on the wrong tree will miss production behavior.

---

## 13. Minimum changes for automatic draft preparation

**Do not implement in Phase 30A.** Minimum *future* steps (order matters):

1. Mount Mac Mini `Broadcast-Automation-Development/{firebase,website}` at `work/broadcast-automation` (`cec18bf` / `8d7599e`) into an agent environment **or** push those trees to a private Git remote.  
2. Read-only confirm lock remains `disabled`.  
3. Trace `checkLatestBroadcast` source for a **draft-only** write path that cannot set `status=published` or `featured=true`.  
4. Isolated emulator tests on **8180/9198** (leave any host `:8080` alone) for Scenarios A–C + E.  
5. If no draft path exists, add the smallest guarded draft writer + admin Media Manager queue — still behind lock/report controls.  
6. Production: authorize a **draft-only** stage explicitly; keep featuring manual.

---

## 14. Minimum changes for fully automatic publication

**NO-GO until drafts are proven.** Additional minimums:

1. Proven category/date/speaker rules for Sunday vs Wednesday (and specialIds).  
2. Duplicate reservation identical to Media Manager.  
3. Feature/pin policy: either never auto-feature, or single-featured + `broadcastAutomation/control` pin semantics with admin override.  
4. Change lock away from `disabled` **only** under an explicit Stage authorization (historical Stage 1 forbade `authorized-v1`).  
5. Set `configureLatestBroadcast` mode to `publish` with `activationRequired=false` only after lock policy is defined.  
6. Production acceptance: one sermon + one bible study match Media Manager field contract; website + app queries pass; rollback plan to report-only.

---

## 15. Recommended implementation order

1. **Phase 30B — Source access & log replay:** Obtain firebase/website `work/broadcast-automation` trees + authenticated read of Cloud Logging for Nov 9–19 around `i3g-ih3K8jQ` / `OgEJWaaTRNA` / `C792v8yRgEY`.  
2. **Phase 30C — Isolated function harness:** Ports 8180/9198; Scenarios A–E against real functions; no production writes.  
3. **Phase 30D — Draft-only stage design:** Smallest safe draft path; still no auto-feature.  
4. **Phase 30E — Controlled publish stage (optional):** Only after draft acceptance; explicit lock/mode change; one-video pilot; immediate report-only rollback.  
5. Keep Media Manager manual publish/feature as the supported path until 30E acceptance.

---

## 16. GO / NO-GO recommendation

| Decision | Recommendation |
|---|---|
| Continue report-only detection | **GO** (already live; leave lock `disabled`) |
| Rely on Media Manager for publish/feature | **GO** (proven Nov 19, 2026) |
| Enable automatic draft preparation | **NO-GO** until Phase 30B–30D |
| Enable fully automatic publication/featuring | **NO-GO** |
| Phase 30A code/deploy changes | **NO-GO** (none performed) |

### Bottom line

New YouTube sermons and Bible studies are **not** automatically published/featured because production broadcast automation is **intentionally locked to report-only**. The website↔app path is healthy for manually published records. The next phase must recover the Mac Mini function source and production logs; it must not unlock publishing.

---

## Appendix A — Sources cited

| Source | Use |
|---|---|
| Firestore anonymous queries on `rm-church-app` / `mediaMessages` | Live Keep Working + Encourage Yourself + sermon/bible-study lists |
| YouTube Atom feed `channel_id=UCEN2tcDJC_CpaHYKbnveDlg` | Video existence / ids / feed timestamps |
| Production admin JS chunks (`configureLatestBroadcast`, `commitMediaResources`, `mediaVideoIds`, Broadcast Automation panel) | Architecture / modes / pin behavior |
| Callable HTTP probes | Function deployment presence + auth |
| Prior agent relays from Phases 25–29N (`bc-10cf9da7` summarizing worker `bc-61676cbe`) | Lock, scheduler, report-only logs, revisions `checklatestbroadcast-00001-wim`, etc. |
| GitHub `restoration-ministries-website-v2` `rm-automation/lib/media-discovery.mjs` | Draft-only analog classifier (labeled as analog only) |
| GitHub `restoration-ministries-app` `src/services/mediaMessages.js` + `src/utils/mediaMessages.js` | App acceptance contract |
| GitHub `website-v2` `src/lib/media/repository.ts` | Public query/feature contract analog |

## Appendix B — Explicit non-actions (Phase 30A)

- No website/Firebase code modifications (except this audit document in the attached placeholder repo)  
- No deployments, rules, IAM, scheduler, or secret changes  
- No production media edits; featured sermon not altered by this audit  
- Publishing lock left disabled  
- No deletion of archived/test lineage  
- Emulator port 8080 not used/interrupted  
- Attachment-permission investigation not reopened  
