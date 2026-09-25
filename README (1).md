# Social Media Automation (n8n + Zernio)

An n8n project that automates three parts of running a business's social presence: scheduled posting from a content calendar, auto-replying to Instagram/Facebook DMs, and auto-replying to comments — all through [Zernio](https://zernio.com), a unified social media API.

## What it does

**1. Scheduled posting.** Content lives in a Google Sheet (caption, a Google Drive link to the media, target platforms, status). The workflow pulls the next pending row, downloads the media from Drive, uploads it to Zernio, and publishes it to every connected account in one call. The sheet is updated back to `posted` or `failed` with the resulting post ID, so the sheet doubles as a status log.

**2. DM auto-reply.** A webhook listens for Zernio's `message.received` event. Incoming (not outgoing) messages are passed to an LLM that drafts a short, on-brand reply, which is sent back through Zernio's inbox API — giving near-instant first-line responses on Instagram/Facebook DMs without a human watching the inbox.

**3. Comment auto-reply.** A second webhook listens for `comment.received`. New comments (excluding the account's own) get a short LLM-drafted reply, posted back via Zernio's comments API.

## Stack

| Component | Role |
|---|---|
| [n8n](https://n8n.io) | Workflow orchestration |
| [Zernio](https://zernio.com) | Unified API for posting, DMs, and comments across social platforms |
| OpenAI (via LangChain node) | Drafts DM and comment replies |
| Google Sheets | Content calendar / posting queue and status log |
| Google Drive | Media storage referenced by the content calendar |

## Setup

1. Import `Social_Media_Automation.json` into n8n.
2. Configure credentials:
   - **Zernio API** — API key from your Zernio account.
   - **Google Sheets OAuth2** and **Google Drive OAuth2** — access to your content calendar sheet/media folder.
   - **OpenAI** — API key for reply drafting.
3. Set up your content calendar sheet with columns: `Post_ID`, `Caption`, `Drive_URL`, `Media_Type`, `Status`, `Zernio_Post_ID`.
4. Activate the workflow, then in Zernio's webhook settings, register the **production** webhook URLs (not the test URLs) for `message.received` and `comment.received`:
   - `https://<your-n8n-host>/webhook/zernio-message-received`
   - `https://<your-n8n-host>/webhook/zernio-comment-received`
5. Trigger the posting flow manually or on a schedule to publish pending rows from the sheet.

## Notes / known limitations

- The posting flow currently targets specific hardcoded account IDs for Instagram and Facebook; swap these for a dynamic "fetch all connected accounts" step if you want new accounts picked up automatically without editing the workflow.
- DM and comment replies are unfiltered LLM drafts with no human-in-the-loop step or conversation ownership tracking — a message sent by a human agent (e.g. through Zernio's inbox) won't currently stop the bot from also replying. Add a "conversation control" check (via Zernio's `conversation.control_changed` / `message.sent` events) before using this in a live support inbox.
- No retry or dead-letter handling on failed Zernio API calls beyond the sheet's `Status = failed` marker for the posting flow.
- Comment/DM auto-reply are free-text LLM output with no guardrails beyond the system prompt — review Zernio's platform policies (Meta, in particular) around automated messaging before enabling in production.
