# attio-champion-move-skill

A Claude skill that watches your insurer contacts for job changes every week. For each high-value person linked to an insurer (claims handlers, referral approvers, decision makers, VIPs), it checks their current LinkedIn employer against what Attio holds. When someone has moved, it flags the move, works out the gap left in the insurer account, drafts a congratulations message to their new role and an introduction request to fill the vacated seat, and routes it to the BD team through a Champion Moves list. Protects pipeline continuity when claims directors rotate. Runs on your own Claude account.

## What it does

Every week, for your high-value insurer contacts:

- **Checks** their current LinkedIn employer + title against the Attio record.
- **On a confirmed move:**
  - Stamps `Last job change` on the person (append, never overwrites).
  - Identifies the gap: which insurer account, which seat is now open, who (if anyone) still covers it.
  - Drafts two messages on the person record: congratulations to the mover at their new role, and an introduction request to fill the vacated seat.
  - Adds the person to the **Champion Moves** list (status New, affected insurer, vacated role, new company/role, confidence) for BD to work.
- **Never** auto-sends anything, and **never** rewrites the person's company or title (that would break account history).

## What it does NOT do

- Does not assert a move it cannot evidence. Ambiguous cases are flagged `Possible` for a human to verify, with no drafts.
- Does not send email. Every message is a draft.
- Does not rewrite `Company` or `Job title`. It records the move and leaves the linkage to BD.
- Does not re-check everyone every week (LinkedIn roles change rarely). It rechecks on a `<RECHECK_DAYS>` window, least-recently-checked first.
- Does not template the drafts. Each one references the real person, move, and account.
- Does not create schema or lists (the MCP cannot). You create the Champion Moves list and one optional attribute once.

## Prerequisites

- Claude with the **Attio**, **APIFY**, and **Lusha** MCP connectors connected, and web search available, on the account that runs the schedule.
- An Attio workspace with `people` and `insurers` objects and **edit** access.
- A **Champion Moves** list on the `people` object (see setup step 2).
- Recommended: a `linkedin_checked` date attribute on `people` (setup step 3).

## Setup

### 0. Connect the connectors in Axxion's Claude
All three (Attio → Axxion workspace, APIFY with a LinkedIn profile actor, Lusha) plus web search must be on the account that runs the schedule. Axxion owns the subscription and the APIFY / Lusha credits.

### 1. Confirm the field map
> List the attribute definitions on my Attio `people` object.
Confirm `company`, `job_title`, `linkedin`, `tpa_role`, `buying_role`, `stakeholder_type`, `contact_owner`, `last_job_change` exist (they do as of build).

### 2. Create the Champion Moves list
New list, parent object **people**, name **Champion Moves**. Entry attributes: `Status` (status: New / Drafts ready / Outreach sent / Closed), `Affected insurer` (record-reference → insurers), `Vacated role` (text), `New company` (text), `New role` (text), `Detected on` (date), `Source` (text), `Confidence` (select: Confirmed / Possible). See [references/attio-setup.md](references/attio-setup.md#champion-moves-list). Note the slug for `<MOVES_LIST>`.

### 3. (Recommended) Add a `linkedin_checked` date attribute
On the `people` object, add a date attribute `LinkedIn checked`. The agent stamps it every run so it does not re-check people too often. Without it, the agent falls back to `last_job_change` dates and Champion Moves membership.

### 4. Identify the APIFY LinkedIn actor
> Which APIFY actors do I have for LinkedIn profiles?
Pick one that returns a profile's current company + title. Lock it into `<APIFY_LINKEDIN_ACTOR>`.

### 5. Verify the workspace name
> Run `whoami` against my Attio.
Note `workspace_name` (Axxion) for `<YOUR_WORKSPACE_NAME>`.

### 6. Dry run
> Run the champion-move agent in demo mode on "<a contact you know recently moved>". Do not write anything.
Check that detection is right and the two drafts read like a person wrote them.

## Usage

### On demand
> Check my insurer contacts for job changes this week.
> Did anyone at our insurer accounts change jobs? Draft the congrats and intro for any who did.
> Has <contact name> moved? If so, draft the outreach.

### Weekly schedule
See [SCHEDULING.md](SCHEDULING.md). In short: install this skill on the account, connect the three connectors, then schedule the trigger prompt weekly (claude.ai scheduled task or Claude Code `/schedule`). The trigger prompt names the installed skill and passes the config; it does not fetch from GitHub.

## Verify it works
1. People object → sort by `LinkedIn checked` (or `Last job change`) → recent dates on processed contacts.
2. A person flagged as moved → `Last job change` stamped, a note with the gap analysis + both drafts.
3. Champion Moves list → moved contacts present, Status `New`, Confidence set.
4. Run summary → checked / moves / possible / skipped counts and any warnings.

## How it's built
Full design in `SKILL.md`; field map, detection rules, and draft voice in `references/`. No Python, no model API key. The scheduled runtime IS Claude: the skill is installed on the account and the schedule triggers it.

This is use case #4 ("Champion-move alert") from the Axxion AI-agent use-case list. Sibling: #2 Insurer enrichment (`attio-insurer-enrichment-skill`). Next: #6 AoI follow-up sequencing.

## License
MIT. Fork, adapt, ship.
