# Chat

A single non-streaming response per message, with an optimistic send and idempotent retry.

```mermaid
flowchart TD
    Page["Chat screen: SessionList + Transcript + Composer"]
    Page --> Sessions["GET /ai/chat/sessions?page=1&limit=20"]
    Sessions -->|empty| NoChats["'No chats yet.<br/>Start one to ask about a claim.'"]
    Sessions -->|"click a session"| Navigate["navigate('/chat/:id')"]

    Page -->|"New chat"| NewCall["POST /ai/chat/sessions (empty body)"]
    NewCall -->|success| NewNav["Session list invalidated,<br/>navigate to the new session"]
    NewCall -->|error| NewErr["'The AI service did not<br/>start the chat.'"]

    Navigate --> Transcript["GET /ai/chat/sessions/:id"]
    Transcript -->|no session selected| Prompt["EmptyState prompt:<br/>'Ask about a claim, a diagnosis code,<br/>or what to check in a document'"]
    Transcript --> Composer["Composer (textarea only —<br/>model is fixed per session)"]

    Composer -->|Send| Optimistic["Pending user message appended<br/>to the cache immediately<br/>(client_message_id minted once)"]
    Optimistic --> SendCall["POST /ai/chat/sessions/:id/messages<br/>{content, client_message_id}"]
    SendCall -->|success| Replace["Pending message replaced with<br/>the real user_message; assistant_message<br/>appended — cache patched, no refetch"]
    SendCall -->|failure| Rollback["Rolled back to the pre-mutation<br/>snapshot (or pending message stripped).<br/>'The message didn't send.' + Send again"]
    Rollback -->|"Send again"| SendCall

    Replace --> Warn{assistant_message.warning<br/>or .truncated?}
    Warn -->|yes| Partial["'The model returned a partial answer.<br/>Try a shorter question or a<br/>different model.'"]
    Replace --> Trunc{result.context_truncated?}
    Trunc -->|yes| Banner["'Older messages are no longer<br/>being sent to the model.'"]
```

## Details

- **Not streamed.** `sendSessionMessage` resolves once with the complete result; there is
  no token-by-token rendering.
- **`client_message_id` is minted once per submission** and reused verbatim on "Send
  again," so a retry after a network failure can't create a duplicate message
  server-side.
- **The session list doesn't refresh after sending a message** — only "New chat" (session
  creation) invalidates it. A session's `last activity` in the sidebar can lag behind
  what's actually in its transcript until the user navigates away and back.
- **The model is fixed per session** (shown in the session list), not chosen per message —
  there's no model/provider picker in the composer itself.
