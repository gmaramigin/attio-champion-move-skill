---
name: champion-move-alert
description: >
  Weekly job-change watch over the people linked to Attio insurer accounts. For
  each high-value contact (claims handlers, referral approvers, decision makers,
  VIPs) with a LinkedIn URL, compares their current LinkedIn employer and title
  against what Attio holds; when someone has moved, flags the move on the person
  record, identifies the gap left in the insurer account, drafts a
  congratulations message to their new role and an introduction request to fill
  the vacated seat, and routes it to the BD team through a Champion Moves list.
  Protects pipeline continuity when claims directors rotate. Runs on the
  customer's own Claude subscription. Use when the user wants to watch for
  contact job changes, catch when a champion leaves an insurer, detect claims
  director moves, draft congratulations or warm-intro outreach after a move, or
  run the weekly champion-move check. Trigger on phrases like "champion move",
  "did any contacts change jobs", "who left their insurer", "claims director
  moved", "watch my contacts for job changes", or when the user names a contact
  and asks whether they have moved.
metadata:
  version: 1.0.0
  workspace: Axxion
---

# Champion Move Alert

You run a weekly job-change watch over the people linked to insurer accounts in the customer's Attio workspace. When a high-value contact moves on, a champion leaving is both a risk (the account loses its inside relationship) and an opportunity (a warm contact now sits somewhere new). You catch the move early, flag it, work out what gap it leaves, and draft the two messages the BD team should send.

You never assert a move you cannot evidence. You never auto-send anything. You never rewrite a person's company or title automatically, because that quietly breaks the account's history.

## The two plays on every move

1. **Congratulate the mover** at their new role. Keeps the relationship warm; today's departed champion is tomorrow's buyer somewhere else.
2. **Request an introduction** to whoever now fills the seat they vacated, so the insurer account keeps a live relationship. The ask goes to the departed champion (warm handoff) or to a remaining contact at the insurer.

## When this skill fires

Four situations:

1. **Setup** — first run for the customer. Verify the workspace, confirm connectors (Attio, APIFY, Lusha, web), confirm the monitored-set query returns the right people, confirm the Champion Moves list exists, dry-run on 2-3 known contacts.
2. **Weekly batch run** — the production loop. For every monitored person not checked in the last `<RECHECK_DAYS>`, fetch their current LinkedIn employer, compare to Attio, and on a confirmed move flag it, build the gap analysis, draft the two messages, and route to the Champion Moves list.
3. **Demo mode** — the user names one contact and wants to see the check and (if moved) the drafts live, with no writes.
4. **Scheduled run** — a weekly scheduled task fired this skill. You ARE the runtime. No external fetch. The skill is installed on the account; read its bundled references and run Mode 2.

## Configuration

| Placeholder | Default | Replace with |
|---|---|---|
| `<YOUR_WORKSPACE_NAME>` | `Axxion` | Exact value `whoami` returns from the Attio MCP |
| `<PEOPLE_OBJECT>` | `people` | The people object slug |
| `<INSURER_OBJECT>` | `insurers` | The insurers object slug |
| `<MOVES_LIST>` | `champion_moves` | The Champion Moves list slug (created by the customer; see references/attio-setup.md) |
| `<RECHECK_DAYS>` | `30` | Skip any person checked within this many days (LinkedIn roles do not change weekly) |
| `<MAX_PEOPLE_PER_RUN>` | `40` | Cap people checked per run. The next run continues with the least-recently-checked |
| `<APIFY_LINKEDIN_ACTOR>` | — | The APIFY actor that returns a LinkedIn profile's current company + title |
| `<SLACK_CHANNEL>` | empty | Optional. BD channel for a one-line alert per confirmed move |
| `<DRY_RUN>` | `false` | If true, detect and print drafts but write nothing |

## Monitored set (high-value seats first)

A person is **in scope** when both hold:

1. They are **linked to an insurer**:
   - their `company` record-reference points to the same companies record an insurer's `main_company` points to, OR
   - they appear in an insurer's `referral_contact`.
2. They hold a **high-value seat**:
   - `tpa_role` in {`Claims handler`, `Referral approver`}, OR
   - `buying_role` = `Decision Maker`, OR
   - `stakeholder_type` in {`VIP`, `Key Stakeholder`}.

And to be checkable: they have a `linkedin` URL (or one resolvable via Lusha by email + name). Skip anyone without a resolvable profile; note `no_linkedin` in the summary.

## Verified people field map (Axxion `people`)

Discover live with `list-attribute-definitions object=people`. Relevant slugs:

| Field | Slug | Type | Use |
|---|---|---|---|
| Name | `name` | personal-name | Identity |
| Email addresses | `email_addresses` | email (multi) | Domain corroboration, Lusha lookup |
| Company | `company` | record-reference → companies | Current employer of record. Do NOT auto-overwrite |
| Job title | `job_title` | text | Current title of record. Do NOT auto-overwrite |
| LinkedIn | `linkedin` | text | Profile to check |
| TPA Role | `tpa_role` | select | Seat importance |
| Buying Role | `buying_role` | select | Seat importance |
| Stakeholder Type | `stakeholder_type` | select | Seat importance |
| Contact Owner | `contact_owner` | actor-reference (multi) | BD owner to notify |
| Last job change | `last_job_change` | text | Stamp the detected move here |
| LinkedIn checked | `linkedin_checked` | date | Stamp every run (the idempotency anchor), even on a no-move pass |

The `insurers` object links to people via `referral_contact` (people) and `main_company` (companies). Read an insurer's `name` for the gap analysis.

## Modes

### Mode 1: Setup
1. `whoami` → confirm workspace equals `<YOUR_WORKSPACE_NAME>`. Hard-stop on mismatch.
2. `list-attribute-definitions object=people` and `object=insurers` → confirm slugs.
3. Confirm APIFY connected; identify the LinkedIn profile actor → `<APIFY_LINKEDIN_ACTOR>`. Confirm Lusha + web available.
4. `list-lists query="champion"` → confirm the Champion Moves list exists. If not, the customer creates it (the MCP cannot create lists). See [references/attio-setup.md](references/attio-setup.md#champion-moves-list). Until it exists, moves are flagged on the person record only.
5. Build the monitored set once and eyeball it: are these the seats BD cares about?
6. Run Mode 3 (demo) on 2-3 contacts, including one you know recently moved. Check that detection is right and the drafts read like a person wrote them.
7. Install the weekly scheduled task (see SCHEDULING.md).

### Mode 2: Weekly batch run

**Preflight** (once, cache):
1. **Workspace gate.** `whoami` → must equal `<YOUR_WORKSPACE_NAME>`. Hard-stop on mismatch.
2. **Field map + option maps.** `list-attribute-definitions object=people`.
3. **Moves list.** `list-lists query="champion"` → cache the list ID + entry attributes (`list-list-attribute-definitions`). If absent, set `moves_list_available = false`.
4. **Insurer index.** `list-records object=insurers` → cache a map of `main_company` record_id → insurer (record_id, name), and collect `referral_contact` person ids. This is how you resolve which insurer a person belongs to.
5. **Build the monitored set.** `list-records object=people` filtered to the high-value seats above; intersect with people linked to an insurer (via the insurer index); keep those with a `linkedin` URL; exclude anyone checked within `<RECHECK_DAYS>` (see idempotency); cap at `<MAX_PEOPLE_PER_RUN>`, least-recently-checked first.

**Per-person loop:**
1. **Resolve the LinkedIn profile.** Use `linkedin`; if missing, try Lusha by email + name. If unresolved, skip with `no_linkedin`.
2. **Fetch current employer + title** via `<APIFY_LINKEDIN_ACTOR>`.
3. **Compare to Attio.** Read the linked `company` record's `name` and the person's `job_title`. Decide move vs no-move per [references/detection-playbook.md](references/detection-playbook.md), applying the false-positive guards (name variants, M&A renames, title-only changes).
   - **No move** → record the check (idempotency), continue.
   - **Confirmed move** → go to step 4.
   - **Ambiguous** → treat as a `possible move` flagged for human verification, not asserted as fact.
4. **Identify the gap.** The insurer account this person belonged to now has a vacant seat = their prior `job_title`. Note the insurer name, the vacated role, and any remaining contacts at that insurer (from the monitored set / insurer links).
5. **Draft the two messages** per [references/outreach-drafts.md](references/outreach-drafts.md): a congratulations to the mover at their new role, and an introduction request to fill the vacated seat. Personalized to the actual person, move, and account. Drafts only.
6. **Write to Attio** (skip if `<DRY_RUN>`):
   - Stamp `last_job_change` on the person: `YYYY-MM-DD: moved from <old company> / <old role> to <new company> / <new role> (source: <linkedin url>)`. Append, do not overwrite prior history.
   - Do **not** change `company` or `job_title`. Leave the CRM linkage for BD to decide.
   - Write a note on the person record containing the gap analysis + both drafts (see output rules).
   - If `moves_list_available`: `add-record-to-list` to `<MOVES_LIST>` with the entry values in references/attio-setup.md (status `New`, affected insurer, vacated role, new company, new role, detected on, source). Else note `moves_list_missing` in the summary.
   - If `<SLACK_CHANNEL>` set: one-line alert (person, old → new, affected insurer, vacated role).
7. **Stamp `linkedin_checked = today`** on the person (the idempotency anchor), even on a no-move pass. Skip if `<DRY_RUN>`. If the stamp write fails, do not count the person as checked, so the next run retries.
8. **Progress log**: checked, moves confirmed, possible moves, no-LinkedIn skips, recently-checked skips, failures.

**Run summary:** print a table (person / old → new / affected insurer / vacated role / status) + counts + warnings (moves list missing, recheck field missing, APIFY unavailable).

### Mode 3: Demo (single contact, no writes)

```
champion-move-alert demo "<contact name>" --no-write
```

`whoami` → resolve the person via `search-records` → fetch their LinkedIn current employer → compare → if moved, print the gap analysis and both drafts; if not, say so and show what was compared. No writes.

### Mode 4: Scheduled run (you ARE the runtime)

A weekly task fires this skill. The skill is installed on the account; read its references directly, do not fetch GitHub. Run Mode 2 preflight + loop. Print the run summary.

## Idempotency

LinkedIn roles change rarely, so checking everyone every week wastes APIFY credit and risks re-flagging the same move. Two guards:

- **Recheck window.** Skip anyone whose `linkedin_checked` date is within `<RECHECK_DAYS>` of today. Stamp `linkedin_checked = today` on every pass (move or no move) so coverage rotates and nobody is re-fetched too often. (`linkedin_checked` is a date attribute on people, confirmed present. If it is ever missing, fall back to the date prefix of the latest `last_job_change` entry plus Champion Moves list membership.)
- **Move dedup.** Before flagging, check whether this exact move (same new company) is already recorded in `last_job_change` or already an entry in the Champion Moves list. If so, skip; do not create a duplicate.

## Error handling

| Failure | Behavior |
|---|---|
| `whoami` mismatch | Hard-stop. No writes. Report actual workspace. |
| APIFY actor unavailable / rate-limited | Soft-fail that person; record no check so the next run retries; note `linkedin_unavailable`. |
| LinkedIn profile unresolved | Skip with `no_linkedin`. Try Lusha first if email present. |
| Current employer is a name variant of the Attio company | Treat as no move (false-positive guard). Do not flag. |
| Difference is ambiguous (possible M&A rename, partial match) | Flag as `possible move` for human verification, not asserted. |
| Champion Moves list missing | `moves_list_available = false`; flag the move on the person record + note only; warn once. |
| `add-record-to-list` fails for one | Soft-fail; the person note + stamp already exist; continue. |
| `update-record` (stamp) fails for one | Soft-fail; do not record the check, so the next run retries. |

## Hard rules

- **Never assert a move without evidence.** A confirmed move needs a LinkedIn (or Lusha) current-employer that clearly differs from the Attio company, plus the source URL. Otherwise it is a `possible move`, not a fact.
- **Never auto-send.** Every message is a draft for a human to review and send.
- **Never rewrite `company` or `job_title`.** Record the move in `last_job_change` and the Champion Moves list; leave the account linkage for BD.
- **Never blank a populated field.** `last_job_change` is append-only, newest first.
- **Workspace gate is absolute.** `whoami` mismatch means hard-stop, no writes.
- **One flag per move.** Dedup before writing.
- **Respect the seat filter and the cap.** Stay on high-value seats; stop at `<MAX_PEOPLE_PER_RUN>`.
- **Drafts are personal, not templated.** Each draft references the actual person, move, and account. No mail-merge voice. No em dashes.

## MCP tools used

| MCP | Tools |
|---|---|
| Attio | `whoami`, `list-attribute-definitions`, `list-lists`, `list-list-attribute-definitions`, `list-records`, `search-records`, `get-records-by-ids`, `update-record`, `create-note`, `add-record-to-list` |
| APIFY | LinkedIn profile actor (`<APIFY_LINKEDIN_ACTOR>`) — current company + title |
| Lusha | Resolve a profile / current employer by email + name when `linkedin` is missing |
| Web | `WebSearch`, `WebFetch` — corroborate a move, learn the new company |
| Slack (optional) | `slack_send_message` for per-move alerts |

## Reference files

| File | Read when |
|------|-----------|
| [references/attio-setup.md](references/attio-setup.md) | People field map, the people↔insurer link, monitored-set query, Champion Moves list spec to create |
| [references/detection-playbook.md](references/detection-playbook.md) | Comparing LinkedIn to Attio, false-positive guards, confirming vs possible moves, gap identification |
| [references/outreach-drafts.md](references/outreach-drafts.md) | Voice and structure of the congrats + intro-request drafts, personalization rules, what never to do |

## Cost expectation

Runs on the customer's existing Claude subscription. One weekly pass checks up to `<MAX_PEOPLE_PER_RUN>` high-value contacts, most of them a single APIFY profile fetch. Moves are rare, so most runs produce a short summary and a few rechecks. The customer's own Attio, APIFY, and Lusha credits.
