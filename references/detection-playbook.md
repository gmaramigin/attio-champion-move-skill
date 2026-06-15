# Detection playbook

How to decide, for one person, whether they have moved. The bar is high: a false "they left" wastes BD time and can embarrass the team. When unsure, downgrade to `possible`, never assert.

## The comparison

For each monitored person:

1. **Attio side.** Read the linked `company` record's `name` and the person's `job_title`. Normalize the company name (lowercase, strip legal suffixes like PSC, LLC, PJSC, Insurance, Takaful, Co.).
2. **LinkedIn side.** From `<APIFY_LINKEDIN_ACTOR>`, take the profile's current employer and current title. Normalize the same way.
3. **Decide:**
   - **No move** — normalized current employer matches the Attio company (exact or clear variant).
   - **Confirmed move** — normalized current employer is clearly a different organization, the LinkedIn position looks current, and ideally one corroborator agrees (see below).
   - **Possible move** — there is a difference, but it could be a name variant, an M&A rename, a stale profile, or a partial match. Flag as `possible`, do not assert.

## Corroboration (raise confidence to Confirmed)

Use at least one when you can:

- **LinkedIn recency.** The profile shows a recently started position at the new employer ("started <month year>").
- **Email domain.** A known new email domain on the person matches the new employer (e.g. moved from `@oldco.ae` to `@newco.com`). Domain change alone is weak; pair it with LinkedIn.
- **Web confirmation.** A quick `WebSearch` ("<person> <new company> appointed / joins") returns a credible source.

If none corroborate and the profile is ambiguous, keep it `possible`.

## False-positive guards (treat as NO move)

- **Name variants.** "Abu Dhabi National Takaful Co. PSC" vs "ADNTC" vs "Abu Dhabi National Takaful" is the same employer. Match on the normalized core, not the string.
- **M&A renames.** The company was renamed or merged, but the person did not move (e.g. "AXA Gulf" → "GIG Gulf"). If the person is at the renamed-but-same entity, it is not a champion move; note it as a company-name update for BD, not a departure.
- **Title-only change.** Same employer, new title (promotion). Not a champion move. Optionally note the promotion in `last_job_change`, but do not run the gap/intro play.
- **Stale or private profile.** If the actor returns nothing current or an obviously old position, do not infer a move. Record the check as inconclusive and move on.

## Identify the gap

On a confirmed move:

1. **Affected insurer.** The insurer account the person belonged to (from the insurer index). Use its `name`.
2. **Vacated seat.** Their prior `job_title` at that insurer (e.g. "Head of Motor Claims"). This is the role now likely open.
3. **Remaining coverage.** Are there other monitored contacts still at that insurer? List them (name, title). If none, the account has zero live contact, which raises priority.
4. **Why it matters.** One line: what this seat does for Axxion's pipeline (referral approval, claims handling, decision making) so BD understands the urgency.

This gap analysis goes into the person note and feeds the intro-request draft.

## Budget and order

Roughly one APIFY profile fetch per person, plus an occasional web search to corroborate or learn the new company. Process least-recently-checked first so coverage rotates evenly across weeks. Moves are rare; a run that finds none and just refreshes check dates is a normal, healthy outcome.
