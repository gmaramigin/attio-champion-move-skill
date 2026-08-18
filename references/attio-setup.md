# Attio setup

Verify against the live workspace before the first scheduled run. Slugs reflect Axxion after the August 2026 migration, when the `insurers` custom object was retired into **Companies**. `list-attribute-definitions object=people` and `list-list-attribute-definitions list=insurers_3` are the source of truth.

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

1. **Via company.** `people.company` is a record-reference to a `companies` record. That company is an insurer when it appears on the **Insurers list** (`insurers_3`). Match on the companies `record_id`.
2. **Via referral_contact.** `referral_contact` is a record-reference **entry attribute on the Insurers list**, listing people directly.

Build an index once per run: `list-records-in-list list=insurers_3` → the set of insurer Company record_ids with their names, plus the `referral_contact` person ids from the entries. A person is linked if their `company.record_id` is in that set or their id is in the referral set. Use the Company `name` for the gap analysis.

The old model put an `insurers` record in the middle, and you had to hop `insurer.main_company → companies`. That hop is gone: the Company *is* the insurer.

**Similar names are separate accounts.** The workspace holds distinct Companies for general vs life licences, takaful vs conventional arms, and branches vs parents (*Chubb Insurance Egypt* / *Chubb Life Egypt*, *Liva Insurance B.S.C.* / *Liva Insurance B.S.C. (c) (UAE Branch)*). Never treat them as one account, and never treat a move between them as a name variant.

## Monitored-set query

In scope = linked to an insurer **and** a high-value seat **and** has a checkable LinkedIn:

- `tpa_role` in {`Claims handler`, `Referral approver`}, OR
- `buying_role` = `Decision Maker`, OR
- `stakeholder_type` in {`VIP`, `Key Stakeholder`}

Practical approach: `list-records object=people` with an OR filter across those select values, then intersect with the insurer index, then keep those with a `linkedin` URL (or resolvable via Lusha). Exclude anyone checked within `<RECHECK_DAYS>`. Cap at `<MAX_PEOPLE_PER_RUN>`, least-recently-checked first.

## Champion Moves list

**Built 18 August 2026.** Parent object **people**, slug `champion_moves`. Nothing to create.

| Entry attribute | Slug | Type | Agent writes |
|---|---|---|---|
| Status | `status` | select | `New`. Options: New, Drafts ready, Outreach sent, Closed |
| Affected insurer | `affected_insurer` | record-reference → **companies** | The insurer account with the gap |
| Vacated role | `vacated_role` | text | The seat now open, the person's prior job title |
| New company | `new_company` | text | Where the person moved to |
| New role | `new_role` | text | Their new title |
| Detected on | `detected_on` | date | Run date, not the date the person moved |
| Source | `source` | text | The LinkedIn URL the move was read from |
| Confidence | `confidence` | select | `Confirmed` or `Possible` |

`affected_insurer` points at **companies**, not the retired insurers object. Insurers are ordinary Companies now and the Insurers list (`insurers_3`) defines the set.

Still discover the live slugs at runtime with `list-list-attribute-definitions list=champion_moves` before writing.

## Connectors checklist

- Attio MCP connected, member has **edit** access.
- APIFY MCP connected; LinkedIn profile actor identified (current company + title).
- Lusha MCP connected (fallback profile resolution).
- Web search/fetch available.
- Slack MCP (optional) connected; alert channel invited the Claude app.
