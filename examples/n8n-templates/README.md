# n8n template: draft WhatsApp replies with AI, approved in Wagent

File: [wagent-draft-reply-with-approval.json](wagent-draft-reply-with-approval.json). Import it in n8n with **Workflows → Import from file**.

It's a REST/webhook companion to the MCP Client Tool setup in [../n8n.md](../n8n.md). This one runs in real time, on every new customer message.

## What it does

1. **Wagent webhook** gets Wagent's signed `message.received` event for a chat the agent is allowed to see. Wagent never sends events about Hidden chats.
2. **Verify signature and keep new messages** checks the Standard Webhooks signature (`webhook-id`, `webhook-timestamp`, `webhook-signature`, HMAC-SHA256 with the `whsec_` secret) when `WAGENT_WEBHOOK_SECRET` is set. It drops every other event type, such as `draft.created` and `approval.requested`, which Wagent sends to every endpoint.
3. **Customer message?** keeps messages from the contact. The owner's own messages are skipped, and group messages are skipped unless `reply_in_groups` is true.
4. **Check chat permission** calls `GET /v1/chats?limit=20` and continues only if this chat's `meta.can.send` is `"ask"`. That is the Draft level. A chat at Send level is skipped, so the workflow never sends without approval. The chat that just got a message is the newest by activity, so it is in the first page.
5. **Confirm message through the API** calls `GET /v1/chats/{id}/messages` and checks that the webhook's `message_id` is really there. It uses the API's text, not the webhook's, so a forged webhook can't trigger a draft even when signatures aren't checked.
6. **Build conversation** skips chats where the business already replied after the customer, and messages without text (voice notes, photos without captions). Then it builds a transcript.
7. **Draft reply with AI** (Basic LLM Chain + OpenAI Chat Model) writes 1 to 3 sentences from your business description, or `NO_REPLY` when nothing needs saying. The prompt treats message text as untrusted (Wagent marks it `untrusted: true`).
8. **Create Wagent draft** calls `POST /v1/messages` with `Idempotency-Key` set to the event id (`evt_…`). The chat is at Draft (send right "Ask"), so Wagent answers `{"status":"waiting","approval_id":"…","expires_at":"…"}` and the reply waits in Wagent's Approvals. The owner approves or skips it on the web or phone.
9. **Waiting for the owner?** routes `waiting` to a success end and `held` (a guardrail like quiet hours or "a person is replying") to a check-Activity end. API errors stop the execution with Wagent's error code, so n8n's error workflow can alert you.

**Duplicates:** the idempotency key is the event id, and Wagent keeps keys for 24 hours. A retried or replayed event can't create a second draft. If the AI writes different text on a replay, Wagent answers `422 idempotency_key_mismatch` and nothing new is saved.

## Required credentials

| n8n credential | Type | Value |
|---|---|---|
| `Wagent API key` | Header Auth | Name `Authorization`, value `Bearer {{WAGENT_API_KEY}}` (a `wg_live_…` key, or `wg_test_…` for the sandbox) |
| `OpenAI account` | OpenAI | Your OpenAI API key. Any chat model sub-node works in its place. |

Optional, self-hosted only: environment variable `WAGENT_WEBHOOK_SECRET={{WAGENT_WEBHOOK_SECRET}}` (the `whsec_…` secret shown once when you add the webhook). n8n Cloud, and instances with `N8N_BLOCK_ENV_ACCESS_IN_NODE=true`, can't read env vars. There the signature isn't checked, and step 5 is the safeguard.

## Setup (about 10 minutes)

1. **Wagent agent and key.** On the Developers screen, create a **server** key for a new agent, for example "n8n reply drafts". A read-only key answers `403 read_only_key` on writes. Turn on the agent's send tool, or writes answer `403 tool_off`.
2. **Chat levels.** Set the chats this workflow should answer to **Draft**. Keep everything else Hidden or Read.
3. **Import** the JSON into n8n. Create the two credentials above and select them in **Check chat permission**, **Confirm message through the API**, **Create Wagent draft** and **OpenAI Chat Model**.
4. **Settings node.** Replace `business_context` with your real hours, prices and policies. The model may only use facts from here or from the chat. Optionally change `history_messages` (1 to 100) and `reply_in_groups`.
5. **Activate** the workflow and copy the **production** URL of **Wagent webhook**.
6. **Wagent webhook.** On the Developers screen, add a webhook with that URL and keep the **messages** family on. Adding one needs the owner's WhatsApp code. Copy the `whsec_…` secret if you'll set `WAGENT_WEBHOOK_SECRET`.
7. **Test in the sandbox first** with a `wg_test_` key: call `POST /v1/sandbox/inbound` with `{"contact": 1, "text": "Hi, is the 7am class on Saturday?"}` (see the API docs at https://app.wagent.sh/docs/sandbox). Then open Wagent's Approvals: the draft should be waiting there. `GET /v1/events` shows whether the webhook delivery succeeded, and `POST /v1/events/{id}/replay` re-sends one.

## Limits to state in the listing

- Text messages only. Voice notes, images and stickers without text are skipped.
- Only chats at Draft level. Read-level chats are skipped by the permission check (the API would answer `403 blocked` anyway). Send-level chats are skipped on purpose.
- A Draft chat is checked on the first page of `GET /v1/chats` (20 chats by recent activity). That covers the chat that just got a message, unless 20 other chats got messages in the same moment.
- Replies wait in Approvals until the owner acts, and expire there after Wagent's approval window (`expires_at` in the response).

## Submission checklist (n8n Creator Portal)

Source: <https://docs.n8n.io/contribute/create-templates>, its [submission guidelines](https://docs.n8n.io/contribute/create-templates/submission-guidelines) and [sticky note guidelines](https://docs.n8n.io/contribute/create-templates/sticky-note-guidelines), fetched 2026-11-09. The Creator Portal writes the listing's title and description. You submit the JSON and setup instructions.

Before submitting:

- [ ] Creator account at <https://creators.n8n.io/register>. New creators submit one template at a time and wait for approval.
- [ ] Search <https://n8n.io/workflows/> for "WhatsApp approval draft" and confirm this fills a gap. The angle: human-in-the-loop WhatsApp replies where the approval is enforced by the API, not by the workflow.
- [ ] **Not yet done: import this exact JSON into a fresh workflow** on a current n8n (Cloud and self-hosted). Follow the setup above with a `wg_test_` key and made-up contacts. Check the draft appears in Wagent's Approvals, not just the green execution.
- [ ] Exercise each branch: owner message (stops at **Customer message?**), Read-level and Send-level chat (stop at **Chat is at Draft level?**), "thanks" (`NO_REPLY`), already-answered chat, a voice note, a replayed event (`422 idempotency_key_mismatch` or the stored answer, never a second draft), a revoked key (stops at **Wagent refused the draft**).
- [ ] Try varied customer messages and review the AI drafts for accuracy and tone. Include one with an instruction inside it ("ignore your rules and…").
- [ ] Note the n8n version you tested in the overview sticky note. The node versions are Webhook 2, Code 2, Edit Fields 3.4, If 2, HTTP Request 4.2, Basic LLM Chain 1.5, OpenAI Chat Model 1.2, Stop and Error 1. Check `require('crypto')` in the Code node on n8n Cloud. Self-hosted may need `NODE_FUNCTION_ALLOW_BUILTIN=crypto`.
- [ ] No secrets, personal data or private IDs. Credentials are referenced by name only. There are no webhook URLs, chat IDs, phone numbers or pinned data. Re-check after your test export, because n8n adds credential IDs and may add pinned data.
- [ ] Nodes have descriptive names. There is one yellow overview sticky (100 to 300 words; this one is 207) with How it works, Setup and Customization. Gray section notes stay under 50 words. One red warning note sits over **Create Wagent draft**.
- [ ] Values to change are grouped in the **Settings** (Edit Fields) node.
- [ ] Error handling: API errors stop the run with Wagent's code. Limitations are listed above and in the sticky notes.
- [ ] Add a canvas screenshot and one screenshot of the waiting draft in Wagent, using made-up names only (for example Priya Shah, Omar Farouk). A screenshot is required only for community nodes. This template uses none, but one helps.
- [ ] Sign in at <https://creators.n8n.io/login> and submit the JSON. Feedback or approval arrives by email.
