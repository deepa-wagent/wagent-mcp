# n8n

1. Add an **AI Agent** node, then attach an **MCP Client Tool** node as one of its tools.
2. Endpoint: your private link. Transport: **HTTP Streamable**.
3. **Tools to include → Selected**. For a drafts-only workflow pick `list_chats`, `get_messages`, `search_messages` and `draft_reply`. Leave out `send_message`.

A ready-to-import lead follow-up workflow is in the Wagent docs.

## Ready-to-import template

[n8n-templates/wagent-draft-reply-with-approval.json](n8n-templates/wagent-draft-reply-with-approval.json): on each new customer message, an AI drafts a reply and Wagent holds it for the owner's approval. It only acts in chats set to Draft, so it never sends on its own. Setup steps: [n8n-templates/README.md](n8n-templates/README.md).
