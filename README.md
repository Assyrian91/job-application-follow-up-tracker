# Job Application Follow-Up Tracker

An automated job search assistant that logs applications, waits and re-checks their status over time, nudges you to follow up when nothing's changed, drafts a follow-up email when there's a contact to reach, and quietly retires applications that have genuinely gone cold — all without manual tracking in a spreadsheet.

## Problem it solves

Job searching at volume means losing track of what's pending, what needs a follow-up, and what's effectively dead. This system handles that bookkeeping automatically: log an application in one Telegram message, and the system manages the entire follow-up lifecycle from there.

## Architecture

```
Telegram message: "Applied: Company - Role" (optionally "- contact@email.com")
  → parsed, given a unique ID, logged to a tracking sheet (status: Applied)
  → Wait 7 days
  → re-check status:
      still "Applied" →
        - update status to "Followed Up" + nudge yourself via Telegram
        - if a contact email exists: AI drafts a personalized follow-up email
          → saved as a Gmail draft (never sent automatically — human review required)
      anything else (you updated it yourself) → do nothing, you're already on top of it
  → Wait another 7-14 days
  → re-check again:
      still "Followed Up" → mark "Stale", final notice to mentally close it out
      anything else → do nothing
```

## Tech stack

- **n8n** — workflow orchestration, including the Wait node for true mid-execution pausing
- **Telegram Bot API** — application intake and all status notifications (separate bot from other projects, to avoid webhook conflicts)
- **Google Sheets API** — application tracking, with unique-ID-based row matching
- **Google Gemini API** (gemini-2.5-flash) — personalized follow-up email drafting
- **Gmail API** — draft creation (intentionally draft-only, not auto-send)
- **JavaScript (n8n Code nodes)** — regex-based parsing, unique ID generation, elapsed-time calculation

## Features

### Application intake
One Telegram message logs an application with company, role, applied date, and an optional contact email — no spreadsheet typing required.

### Time-delayed, automatic re-checking
Using n8n's Wait node, each logged application pauses its own independent execution for real days, then wakes up and checks whether its status has changed — the same "wait and follow up" pattern used in production reminder systems.

### Two-stage escalation
A quiet "Followed Up" nudge after the first wait period, escalating to a "Stale" closing note after a second period if there's still been no movement — so nothing lingers indefinitely without you noticing.

### AI-drafted follow-up emails, with a human in the loop
When a contact email is on file, the AI writes a complete, personalized follow-up email. It's saved as a Gmail draft, not sent — a deliberate decision, since outbound communication to a real recruiter is too high-stakes to send unreviewed.

## Setup

You'll need:
1. A dedicated Telegram bot token (via [@BotFather](https://t.me/BotFather)) — kept separate from any other Telegram-triggered workflow to avoid the two workflows fighting over the same webhook
2. A Google Sheet for tracking (`ID`, `Company`, `Role`, `AppliedDate`, `Status`, `ContactEmail` columns)
3. A Google Gemini API key
4. A Gmail account connected via OAuth2 for draft creation

## Challenges & debugging

This project surfaced the most varied set of real bugs so far:

- **Non-unique matching key:** early versions matched rows by Company + Role. Testing the same application twice created duplicate rows that all matched the same search, so a single re-check incorrectly updated multiple rows at once. Fixed by generating a genuinely unique ID (`Date.now()`) at creation time and matching on that everywhere downstream instead.
- **A wire silently bypassing the core logic:** a connection accidentally ran straight from the confirmation message to the decision node, skipping the Wait and re-check nodes entirely — meaning the branch was deciding based on stale data from the moment of logging, not the actual current status. Found by reading the canvas wiring carefully rather than assuming the diagram matched intent.
- **Date math on the wrong data type:** an AI prompt tried to calculate elapsed days using a formatted date string, which date-diff functions can't read directly. Solved by calculating elapsed days in JavaScript from the unique ID's embedded timestamp instead, and passing a plain number into the prompt.

## Demo

🎥 [https://www.loom.com/share/1adab62d29a845a589e127f2ae724622]

## Possible next steps

- Auto-detect replies to logged applications (via Gmail trigger) and update status automatically instead of manual edits
- A weekly digest of all currently-pending applications, similar to the finance project's report pattern
- Light CRM-style notes per application (interview dates, interviewer names)

## Running cost

Telegram Bot API, Google Sheets API, Gmail API: free. Gemini 2.5 Flash: free at this usage level. n8n: free if self-hosted, or from $24/month on n8n Cloud after the trial period.
