# Attio setup

Verify against the live workspace before the first scheduled run. Slugs reflect the Axxion `people` and `insurers` objects at build time. `list-attribute-definitions` is the source of truth.

## People field map

| Field | Slug | Type | Use |
|---|---|---|---|
| Name | `name` | personal-name | Identity |
| Email addresses | `email_addresses` | email (multi) | Lusha lookup, email-domain corroboration |
| Company | `company` | record-reference → companies | Current employer of record. Read its `name` to compare. Never auto-overwrite |
| Job title | `job_title` | text | Current title of record. Never auto-overwrite |
| LinkedIn | `linkedin` | text | Profile URL to check |
| TPA Role | `tpa_role` | select | Seat importance (Referral approver, Claims handler, Finance/Billing, Workshop manager, Broker/Agent) |
| Buying Role | `buying_role` | select | Seat importance (Decision Maker, Influencer, Blocker) |
| Stakeholder Type | `stakeholder_type` | select | Seat importance (VIP, Key Stakeholder, F contact) |
| Contact Owner | `contact_owner` | actor-reference (multi) | BD owner to notify |
| Last job change | `last_job_change` | text | Stamp the detected move here (append, newest first) |
| LinkedIn checked | `linkedin_checked` | date | Idempotency anchor. Stamp the check date every run, even on a no-move pass. Confirmed present in the workspace |

## The people ↔ insurer link

A person belongs to an insurer account by either path:

1. **Via company.** `people.company` is a record-reference to a `companies` record. An insurer's `insurers.main_company` points to the same `companies` record. Match on the companies `record_id`.
2. **Via referral_contact.** An insurer's `insurers.referral_contact` is a multi record-reference listing people directly.

Build an index once per run: for every insurer, map `main_company.record_id → {insurer record_id, name}` and collect `referral_contact` person ids. Then a person is linked if their `company.record_id` is in that map or their id is in the referral set. Use the insurer `name` for the gap analysis.

## Monitored-set query

In scope = linked to an insurer **and** a high-value seat **and** has a checkable LinkedIn:

- `tpa_role` in {`Claims handler`, `Referral approver`}, OR
- `buying_role` = `Decision Maker`, OR
- `stakeholder_type` in {`VIP`, `Key Stakeholder`}

Practical approach: `list-records object=people` with an OR filter across those select values, then intersect with the insurer index, then keep those with a `linkedin` URL (or resolvable via Lusha). Exclude anyone checked within `<RECHECK_DAYS>`. Cap at `<MAX_PEOPLE_PER_RUN>`, least-recently-checked first.

## Champion Moves list

Does not exist yet. The MCP cannot create lists, so the customer creates it in Attio:

1. New list, parent object **people**, name **Champion Moves** (suggested slug `champion_moves`).
2. Entry attributes the agent writes via `add-record-to-list`:

| Entry attribute | Suggested slug | Type | Agent writes |
|---|---|---|---|
| Status | `status` | status or select | `New` (options: New, Drafts ready, Outreach sent, Closed) |
| Affected insurer | `affected_insurer` | record-reference → insurers | The insurer account with the gap |
| Vacated role | `vacated_role` | text | The seat now open (the person's prior job title) |
| New company | `new_company` | text | Where the person moved to |
| New role | `new_role` | text | Their new title |
| Detected on | `detected_on` | date | Run date |
| Source | `source` | text | LinkedIn URL evidence |
| Confidence | `confidence` | select | `Confirmed` or `Possible` (options: Confirmed, Possible) |

3. Put the slug into `<MOVES_LIST>`.

Discover the real entry-attribute slugs at runtime with `list-list-attribute-definitions list=champion_moves`; the names above are suggestions. Until the list exists, the agent flags moves on the person record (`last_job_change` + a note) and warns once in the summary.

## Connectors checklist

- Attio MCP connected, member has **edit** access.
- APIFY MCP connected; LinkedIn profile actor identified (current company + title).
- Lusha MCP connected (fallback profile resolution).
- Web search/fetch available.
- Slack MCP (optional) connected; alert channel invited the Claude app.
