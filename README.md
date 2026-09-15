# Quorum

**An AI pipeline that turns messy meeting notes into tracked, owned, accountable action items.**

Built end to end with n8n and the Gemini API · [▶ 2-minute demo](https://www.loom.com/share/13d8435cea2744ab82d6523fede40cb4)

---

## The problem

Every meeting ends the same way. Decisions get made, tasks get mentioned, everyone nods — and a week later nobody remembers who owns what.

Note-taking tools solve the *recording* problem. They don't solve the *accountability* problem.

## What Quorum does

Drop in unstructured meeting notes and it:

1. **Extracts** every action item as structured data — owner, task, due date, work type
2. **Flags** tasks with no clear owner as `UNASSIGNED` rather than guessing
3. **Routes** each task by work type toward where that kind of work actually lives
4. **Chases** overdue items automatically on a daily schedule
5. **Alerts** on failure, so nothing disappears silently

### Example

**Input** — raw meeting notes:
```
Team sync 18 Aug. Priya will finalise the vendor contract by 22 Aug.
We decided to delay the mobile launch to Q4. Rahul to prepare the revised
timeline slides by 20 Aug. Anjali will confirm the budget in Power BI by 25 Aug.
Dev team to fix the login bug this week, owner unclear.
```

**Output** — structured, routed records:

| owner | task | due | type |
|---|---|---|---|
| Priya | finalise the vendor contract | 22 Aug | general |
| Rahul | prepare the revised timeline slides | 20 Aug | slides |
| Anjali | confirm the budget in Power BI | 25 Aug | bi |
| **UNASSIGNED** | fix the login bug | this week | dev |

Note two things: the ownerless task is **flagged, not guessed**, and *"delay the mobile launch to Q4"* is correctly **ignored** — it's a decision, not an action item.

---

## How it's different from Copilot / Otter / Fireflies

Note-taking is commoditised. Those tools do transcription and summarisation better than a solo build could, and Quorum doesn't compete there.

It solves what they structurally leave open:

| | Note-takers | Quorum |
|---|---|---|
| Capture what was said | ✅ | ✅ |
| Route tasks to where work lives | ❌ walled garden | ✅ by work type |
| Own the follow-through loop | ❌ | ✅ scheduled chase |
| Surface accountability gaps | ❌ assigns or omits | ✅ flags `UNASSIGNED` |

**Notes are easy. Accountability is the hard part.**

---

## Architecture

Three separate workflows, split by trigger cadence rather than crammed into one:

```
1 · EXTRACT & ROUTE        (on new meeting notes)
   Trigger → AI extraction → JSON parse → split → Switch (by type) → tracked record

2 · OVERDUE CHECKER        (daily schedule)
   Schedule → read tasks → filter overdue (date comparison) → AI drafts reminder

3 · ERROR HANDLER          (on any failure)
   Error trigger → alert with workflow name + execution link
```

Why three and not one: each has a different trigger and lifecycle. Coupling them would mean things that should fail and run independently could take each other down.

---

## Design decisions worth calling out

**Never guess an owner.** A wrong owner is worse than a visible gap — a guess quietly creates a false record, while a flagged gap forces a human to resolve it. Surfacing uncertainty beats hiding it behind confident output.

**Classify, then route.** The LLM judges the work type; a deterministic Switch node does the actual routing. Models are good at interpreting messy human text and unreliable at deterministic actions, so the AI decides and never owns the reliable step.

**Structured output, not prose.** The extraction returns strict JSON so downstream systems can act on it. Free text can't be routed anywhere.

**Retry ≠ error handling.** Retries are configured for transient failures (a brief API blip). Permanent failures — a bad key, a malformed prompt — surface through a dedicated error workflow instead of being silently retried.

---

## A failure worth documenting

The first extraction prompt produced *perfectly formatted* output: clean JSON, every field populated. It had also invented every single task — it ignored the actual meeting notes and generated plausible fiction matching the schema it was asked for.

It was caught by testing against notes where the correct answers were known in advance.

The fix was restructuring the prompt to lead with the real input and explicitly forbid invention. The lesson generalised: **format compliance and factual accuracy are separate failure modes.** An AI feature that "produces output" and one that "produces true output" need completely different checks.

---

## Honest limitations

- **Context-aware routing is demonstrated, not production-grade.** Real routing to each owner's actual toolset would need a mapping of people to systems plus API access to each. This build proves the routing concept with tagged destinations.
- **Extraction quality depends on the model.** Messy or ambiguous transcripts produce weaker results; production would need confidence thresholds and human review on low-confidence extractions.
- **Meeting content is sensitive.** A real deployment needs proper data-handling policy on what the model sees and whether it's retained.

## What production would add

Per-owner tool mapping · human review queue for low-confidence extractions · real integrations (Jira, Notion, Slack) in place of tagged records · analytics on extraction accuracy over time.

---

## Repo contents

- `workflows/` — exported n8n workflow definitions (JSON), inspectable and importable
- Demo: [Loom walkthrough](https://www.loom.com/share/13d8435cea2744ab82d6523fede40cb4)

---

**Built by [Devansh Sharma](https://devansh-sharma.netlify.app)** — Product Manager · AI & Enterprise Platforms
