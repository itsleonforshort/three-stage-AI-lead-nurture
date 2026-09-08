# Three-Stage Lead Nurture Sequence

An n8n automation that follows up new leads over five days, writes each email with AI, and — the part that matters — knows when to stop.

71 nodes. Three AI stages. Six different ways for a lead to exit early.

## The problem

Most leads never say no. They go quiet, and by the time anyone remembers to follow up they've already bought elsewhere. Manual follow-up doesn't scale; naive automation keeps emailing people who already replied, already unsubscribed, or whose address bounced on the first send.

## How it runs

**Stage 1 — Welcome (immediate)**

A lead arrives by webhook. Before anything is sent, three gates run:

- Has this lead already been contacted?
- Is it a duplicate of one already in the sheet?
- Is the address on the do-not-email list?

Only then does an AI agent write the welcome email. A guard checks the output before Gmail sends it.

**Stage 2 — Follow up (2 days later)**

Three checks before doing anything:

| Check | If yes |
|---|---|
| Unsubscribed? | Stop, log it |
| Bounced? | Stop, log it |
| Replied? | Stop, mark as replied, hand to a human |
| Silence | A second AI agent writes the follow up |

**Stage 3 — Close the loop (2 more days)**

Same three checks. Still silence, so a third agent sends one final message and tags the lead as not interested. Then it leaves them alone.

## The parts that took the longest

**Detecting a real reply is harder than it looks.** The workflow fetches the Gmail thread and removes everything that isn't the lead: messages tagged `SENT` or `DRAFT` (our own), and robot senders (`mailer-daemon`, `postmaster`, `no-reply`, `do-not-reply`, `bounce`, `auto-reply`). Whatever remains is a human writing back.

Two details that cost real debugging time:

- n8n strips `labelIds` off thread messages and replaces it with a `labels` array of objects. Reading `labelIds` returned `undefined`, so the SENT filter never fired, **our own welcome email counted as an inbound reply**, and every lead was marked "Replied" two days after being written to.
- The robot check is anchored to the *start of the local part* of the address. Testing the whole `From` header threw away real people at addresses like `sarah@notificationlabs.com`.

**A bounce is not a reply, and not silence either.** A hard bounce arrives in the same thread from `mailer-daemon` or `postmaster`. The address is dead, so two further emails would only damage sending reputation. It gets its own terminal state.

**It fails closed.** If the Gmail thread cannot be read at all, the workflow stops and alerts rather than assuming silence and sending another email. Every ambiguity here ends in a real person's inbox.

**A guard sits on every AI output.** The structured output parser validates the *shape* of what the model returns, not the content — an empty subject and body passes it cleanly. Each stage therefore has a code node that rejects an empty subject, an empty body, or a body under 40 characters before it can reach Gmail.

**Per-item state is passed hand to hand, never looked up.** `$('Node').first()` returns that node's *last run in the whole execution*, not the current item's run — so any counter read back through a node reference inherits the previous lead's value. `$runIndex` has the same flaw.

## Error handling

Every fallible node has an error output. All of them converge on one path: explain the failure, record it against the lead with a reason, and email the team immediately.

The reason a lead's row is never left mid-sequence saying "New" is that this path exists. Without it, a lead could be emailed once and then abandoned with nobody finding out for days.

## Unsubscribe

A real one, not a decorative link:

- A GET endpoint serves a confirmation page
- A POST endpoint verifies a per-lead token before acting
- The address is added to a do-not-email list and the lead is marked unsubscribed
- Every later stage re-checks that list before sending

## Stack

| Piece | Used for |
|---|---|
| n8n | Orchestration, webhooks, scheduling |
| Google Gemini | Writing each of the three emails |
| Gmail | Sending, and reading threads for replies and bounces |
| Google Sheets | Lead record, status, failure log, do-not-email list |

## Setup

Import `workflows/three-stage-lead-nurture.json` into n8n, then:

1. Connect Gmail, Google Sheets and Gemini credentials inside the n8n interface
2. Replace `YOUR_GOOGLE_SHEET_ID` with your own sheet
3. Replace `your-email@example.com` with the address that should receive team alerts
4. Create a sheet with columns: `lead_id`, `name`, `email`, `company`, `pipeline_stage`, `status`, `final_status`, `thread_id`, `message_id`, `error_reason`, `created_at`, `unsub_token`

No credentials are stored in this repository. The workflow JSON references credentials by name only.
