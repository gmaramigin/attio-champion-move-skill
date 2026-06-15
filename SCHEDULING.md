# Running the Champion Move Alert on a weekly schedule (Claude cloud)

For Axxion. The agent runs on Axxion's own Claude account. Nothing is fetched from GitHub at run time: the skill is installed on the account and the scheduler triggers it.

Three parts: install the skill, connect the connectors, set the schedule.

## Part 1 — Install the skill

Artifacts:
- **champion-move-alert.skill** — single self-contained file (references inlined).
- The skill **folder / .zip** (SKILL.md + references/) from the GitHub repo.

claude.ai (web / desktop): Settings → Capabilities (Skills) → Add / Upload → select the folder, .zip, or .skill file. Confirm it shows as **champion-move-alert**.

Claude Code (CLI): create `~/.claude/skills/champion-move-alert/` and copy SKILL.md + references/ into it.

## Part 2 — Connect the connectors (same account)

1. **Attio** → Axxion workspace, member with **edit** access. Verify with `whoami` → must return `Axxion`.
2. **APIFY** → connected, with a LinkedIn profile actor (returns current company + title). Note its name.
3. **Lusha** → connected (fallback profile resolution).
4. **Web search** → enabled.

Also confirm the **Champion Moves** list exists on the `people` object, and (recommended) the `linkedin_checked` date attribute.

## Part 3 — Set the weekly schedule

### Trigger prompt

```
Run the champion-move-alert skill, Mode 2 (weekly batch run), over the Axxion Attio workspace.

Config:
- People object: people
- Insurer object: insurers
- Champion Moves list: champion_moves
- Recheck days: 30
- Max people per run: 40
- APIFY LinkedIn actor: <name of your LinkedIn profile actor>
- Slack channel (optional): <#channel or empty>
- Workspace expected from whoami: Axxion
- Dry run: false

Monitored set: people linked to an insurer (company maps to an insurer's main_company, or in referral_contact) AND a high-value seat (tpa_role in {Claims handler, Referral approver}, OR buying_role = Decision Maker, OR stakeholder_type in {VIP, Key Stakeholder}) AND with a LinkedIn URL.

Hard rules: never assert a move without LinkedIn (or Lusha) evidence + source URL; never auto-send (drafts only); never rewrite company or job_title; always record the check date; write nothing if whoami is not Axxion.

End by printing the run summary table.
```

Replace `<name of your LinkedIn profile actor>`.

### Option A — claude.ai scheduled task (recommended)
1. Run the trigger prompt once manually; check the run summary, a flagged person's note, and the Champion Moves list.
2. Open the scheduling / automation control for that conversation.
3. Set: Weekly, Monday 07:00 Axxion local, Prompt = the trigger prompt. Save.

(Menu names vary by plan/version. The goal is a recurring task whose prompt is the trigger prompt. If your plan has no scheduling, use Option B.)

### Option B — Claude Code /schedule (cloud-backed)
```
/schedule
Cadence: 0 7 * * 1   (Mondays 07:00 local)
Name: champion-move-weekly
MCP: Attio, APIFY, Lusha
Prompt: <paste the trigger prompt above>
```

## Verify after the first run
1. People → sort by `LinkedIn checked` → recent dates.
2. A moved contact → `Last job change` stamped, a note with the gap analysis + both drafts.
3. Champion Moves list → moved contacts present, Status New, Confidence set.
4. Run summary → checked / moves / possible / no-LinkedIn / recently-checked counts and warnings.

## Tune it
- **Cadence:** change day/time. Weekly is plenty; moves are rare.
- **Recheck window:** `Recheck days` (default 30). Lower = fresher, more APIFY spend.
- **Volume:** `Max people per run` (default 40). Least-recently-checked go first, so coverage rotates.
- **Scope:** widen the monitored set by editing the seat filter in the prompt (e.g. add `Influencer`).
- **Dry run:** set true to detect and print drafts without writing.

## If something looks off
- "Wrote nothing / workspace mismatch" → `whoami` did not return Axxion; point Attio at the Axxion workspace.
- "No moves ever" → expected most weeks. Confirm the APIFY actor returns current employer, and that monitored people have LinkedIn URLs.
- "Flagged someone who did not move" → likely a name variant or M&A rename; the detection guards should catch these, tighten the actor or report the case.
- "Move not in the list" → confirm `champion_moves` exists with the entry attributes; until then moves are flagged on the person record only.
