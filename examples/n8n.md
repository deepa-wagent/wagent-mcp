# n8n

1. Add an **AI Agent** node, then attach an **MCP Client Tool** node as one of its tools.
2. Endpoint: your private link. Transport: **HTTP Streamable**.
3. **Tools to include → Selected**. For a drafts-only workflow pick `list_chats`, `get_messages`, `search_messages` and `draft_reply`. Leave out `send_message`.

A ready-to-import lead follow-up workflow is in the Wagent docs.
