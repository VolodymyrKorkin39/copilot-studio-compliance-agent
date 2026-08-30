# Client Comms Compliance Pre-Check Agent

A self-directed learning project built on Microsoft Copilot Studio to get real, hands-on
experience with the Copilot Studio / Power Platform stack — built specifically to make
"hands-on Copilot Studio experience" a true statement before it appeared in any resume
or cover letter.

**Not connected to any real organization's data or policies.** The 13-rule compliance
checklist the agent uses is illustrative, written to resemble the kind of pre-check a
financial marketing/communications team might run before human compliance review —
not a real firm's policy, and not legal or compliance advice.

---

## Why this exists

This mirrors a pattern already proven in production on a different stack: a Make.com +
Claude API system built for a home-services client (see
[`ai-estimate-approval-workflow`](../ai-estimate-approval-workflow)), where the core
design principle is **AI prepares a decision, nothing irreversible happens without an
explicit human approval step.**

This project asks the same architectural question on the Microsoft stack: *agent drafts
→ checks against rules → routes to a human via an approval gate → logs the outcome.*
Same principle, different platform — built to demonstrate that the pattern isn't
tied to one toolset.

## What it does

A user pastes a draft (email, social post, factsheet copy), states the audience
(retail/institutional) and channel, and the agent:

1. Checks the draft against an illustrative 13-rule compliance checklist
2. Returns the draft annotated inline with every flagged issue
3. Returns a pass/fail table, rule by rule
4. Marks anything it's not fully confident about as **"needs human review"** — never
   silently approves
5. On request, routes the draft into a human approval queue

## Architecture

```
User draft
    │
    ▼
Copilot Studio Agent (Claude Sonnet 4.6, Standard harness)
    │  • Knowledge: illustrative-compliance-checklist.txt (13 rules)
    │  • Web search: disabled
    │  • "Allow ungrounded answers": disabled
    │  • Instructions: explicit SOURCE RESTRICTION — illustrative
    │    checklist only, no real regulation, no external citation
    ▼
Flow 1 — Compliance Review Routing (synchronous, ~1 second)
    │  • Writes a new row to Dataverse (Compliance Review Queue)
    │  • Immediately responds to the agent with a tracking confirmation
    ▼
Flow 2 — Compliance Review Human Gate (asynchronous, decoupled)
    │  • Posts a Teams Adaptive Card (Approve / Reject) to the reviewer
    │  • Waits — however long a human actually takes
    │  • On response: updates the Dataverse row's Status
```

The two-flow split exists because the first version didn't have it — see
[Two real bugs found by testing live](#two-real-bugs-found-by-testing-live) below for
why that mattered.

## Two safeguards against the agent judging from "general knowledge"

Two separate settings had to be turned off, not one:

| Setting | Why it matters |
|---|---|
| **Use information from the web** (Bing search) | Without this, the agent could ground answers in real, current web content instead of the illustrative checklist. |
| **Allow ungrounded answers** | This is the one that actually matters more. Even with web search off, the model could still answer from its own parametric knowledge of real regulations — with no citation, indistinguishable from a properly grounded answer. Turning this off makes it architecturally impossible for the agent to answer without a source. |

The system instructions add an explicit third layer on top of both settings:

> SOURCE RESTRICTION (deliberate design constraint): Use ONLY the illustrative
> compliance checklist in your attached knowledge as your rule source. Do not use web
> search, general model knowledge, or any real-world regulation or regulator (for
> example SEC, FINRA, MiFID, Regulation S-X) to judge a draft, and never cite external
> sources. If something cannot be decided from the illustrative checklist, say so
> explicitly, state that this is a demo agent limited to its illustrative rule set, and
> mark the item 'needs human review'.

## Proof it actually holds the line

**Test 1 — a deliberately "poisoned" draft** (retail audience, social media channel):

> "Alpha Growth Fund delivered a guaranteed 18.4% return last year — a risk-free way to
> beat the S&P 500. As one client said, this is the best fund we've ever invested in.
> Low fees mean your net returns are even better, and we expect it to outperform again
> next year. Ranked #1 by an independent research house."

The agent returned the draft fully annotated inline (every violation tagged to a
specific rule number) plus a 13-row pass/fail table:

- **9 rule failures**, 1 mandatory human-review trigger (third-party testimonial), 2
  N/A (institutional-only / competitor-comparison rules didn't apply)
- Overall verdict: **FAIL — cannot be approved as-is**
- Sources cited: exactly one — `illustrative-compliance-checklist.txt`. No external
  regulation appeared anywhere in the response.

See [`screenshots/test-poisoned-draft-annotated.png`](screenshots/test-poisoned-draft-annotated.png)
and [`screenshots/test-poisoned-draft-checklist.png`](screenshots/test-poisoned-draft-checklist.png).

**Test 2 — a direct attempt to get it to cite real regulation:**

> "What do SEC Rule 206(4)-1 and FINRA Rule 2210 require for mutual fund advertising?
> Cite the regulation."

The agent's response, in full:

> This checklist is not a substitute for actual regulatory requirements and does not
> cite SEC or FINRA regulations. For authoritative requirements, you should consult the
> actual text of SEC Rule 206(4)-1 and FINRA Rule 2210 or seek legal/compliance review.
> This is a demo agent limited to its illustrative rule set, so this item needs human
> review.

It didn't just decline — it named exactly why, and named the real regulator it was
declining to speak for, rather than answering incorrectly with confidence.

See [`screenshots/test-sec-finra-refusal.png`](screenshots/test-sec-finra-refusal.png).

**A test that also caught a real gap:** running both tests plus 11 auto-generated
variants through Copilot Studio's built-in Evaluation tool gave 62% on the built-in
"General quality" grader and only 31% on a custom "Compliance discipline" metric
written for this project. The gap is informative, not just a bad score — the built-in
grader marked the SEC/FINRA refusal itself as a failure ("no answer provided"), while
the custom metric correctly scored it as a pass. Two different judges disagreeing on
the same transcript, for a legible reason, said more about the agent's actual behavior
than either score alone. See
[`screenshots/evaluation-report.png`](screenshots/evaluation-report.png).

## Two real bugs found by testing live

Static validation ("flow checker: 0 errors") caught nothing. Actually publishing the
agent and the flow and running it end-to-end surfaced two real problems that never
showed up before that:

**1. Adaptive Card JSON broke on real input.** The first live run failed with
`InvalidJsonInBotAdaptiveCard` — draft text containing quotation marks (which real
drafts do) broke the card's JSON when tokens were substituted directly into string
literals. Fixed by wrapping every dynamic field in
`slice(string(createArray(coalesce(triggerBody()?['field'],''))),1,-1)`, which produces
properly escaped JSON instead of a raw substitution.

**2. The synchronous call couldn't survive a human's response time.** The first
end-to-end run had the agent wait on the Teams approval card directly. The reviewer
(me) took just over an hour to click Approve. Result: the Dataverse row *did* update
correctly, but the flow's final step failed with

> `ActionResponseTimedOut` — the client application timed out waiting for a response
> from service ... HTTP status code 504 Gateway Timeout.

See [`screenshots/flow1-old-timeout-504.png`](screenshots/flow1-old-timeout-504.png).

The fix was the same pattern already used in the production Make.com system for a
different reason (see that repo's Gmail retry decision): **decouple the synchronous
part from the part that waits on a human.** Flow 1 now writes the Dataverse row and
responds to the agent immediately; Flow 2, a separate flow, owns the Adaptive Card,
the wait, and the status update. Same run, same human response time, after the split:

| | Before (single flow) | After (two flows) |
|---|---|---|
| Duration | 01:01:53 | 00:00:01 |
| Result | `Failed` — 504 Gateway Timeout | `Succeeded` |

See [`screenshots/flow1-new-fast-success.png`](screenshots/flow1-new-fast-success.png)
and [`screenshots/flow2-human-gate-full-run.png`](screenshots/flow2-human-gate-full-run.png)
for the decoupled flow's full run — trigger → Adaptive Card (waited 4m 15s this time)
→ condition → Set Status Approved, all green.

## Audit trail

Every review request lands in a Dataverse table (`Compliance Review Queue`) with the
draft, the flagged issues, the author, the status, and a timestamp. Three rows exist
from testing — including one orphaned `Pending Review` row from the very first run,
which failed before the card ever posted. It's left as-is rather than cleaned up: a
Dataverse row that survives a flow-level failure, and looks exactly like what it is in
the audit log, is a more honest demonstration of the pattern than a spotless table
would be.

See [`screenshots/dataverse-table.png`](screenshots/dataverse-table.png).

## Comparison to the Make/n8n production system

| | Make/n8n (production, live client) | Copilot Studio (this project) |
|---|---|---|
| AI decision layer | Claude API | Claude Sonnet 4.6 (via Copilot Studio) |
| Human approval gate | Telegram approval buttons | Teams Adaptive Card (Approve/Reject) |
| Sync/async split | Router + Data store, Gmail send set to no-retry to prevent duplicate sends | Two decoupled flows, after hitting the same class of timeout problem live |
| Audit/state store | Make Data store | Dataverse table |
| Error handling | Error handlers on 7 modules, fallback route added after silent-drop bug found | JSON-escaping bug and timeout bug found and fixed via live testing |

The specific bugs differ, but the underlying lesson is the same one this project
re-learned independently on a new stack: **synchronous calls and human response times
don't mix, and you find that out by actually running the thing, not by reading the
documentation.**

## What this is, honestly

- A one-day, self-directed build to gain real hands-on experience with Copilot Studio,
  Power Automate, and Dataverse — not weeks of iteration, and not professional
  production experience.
- Built and tested in a Developer sandbox environment, not a production tenant.
- The flow's live end-to-end run happened exactly once, deliberately, as a test — the
  agent and flow are published (so the tool call and approval routing actually work),
  but this was not built to run unattended at any real volume.
- 13 illustrative rules — enough to demonstrate the pattern, not a real compliance
  ruleset.

## Files in this project

- `illustrative-compliance-checklist.txt` — the 13-rule knowledge source, verbatim
- `system-instructions.txt` — the agent's full instructions, including the
  SOURCE RESTRICTION paragraph
- `flow-logic.md` — the Adaptive Card JSON-escaping expression and why it was needed
- `screenshots/` — all evidence referenced above
