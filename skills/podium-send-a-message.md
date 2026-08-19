---
name: Send a message to a customer
description: Find or create a contact, locate or open the conversation, and send an SMS/email message from a Podium location — including the opt-in and 10DLC constraints that make most first attempts fail.
api: openapi/podium-messenger-openapi.yml
operations: [contact.index, contact.create, conversation.index, message.send, message.send_with_attachment, message.index]
scopes: [read_contacts, write_contacts, read_messages, write_messages]
---

# Send a message to a customer (Podium)

Base URL `https://api.podium.com/v4/`. Every call carries `Authorization: Bearer {access_token}`
and `Content-Type: application/json`, and should carry `podium-version: 2021.04.01` — without the
version header Podium serves the latest version, which may contain breaking changes.

## Steps

1. **Find the contact.** `contact.index` (`GET /v4/contacts`) with a `search` /`searchFields`
   filter, or `contact.get` (`GET /v4/contacts/{identifier}`) when you already hold the phone or
   email identifier. Requires `read_contacts`.
2. **Create it if missing.** `contact.create` (`POST /v4/contacts`). Requires `write_contacts`.
3. **Confirm messaging consent.** A contact that has not opted in cannot be messaged. Use
   `contact.opt_in` (`POST /v4/contacts/campaigns/opt_in`) for campaign consent, and remember that
   in a *test account* all contacts start unsubscribed — text `START` to the location's Podium
   number to opt a test handset in.
4. **Send.** `message.send` (`POST /v4/messages`) with the location, the contact and the body.
   For media use `message.send_with_attachment` (`POST /v4/messages/attachment`,
   `multipart/form-data`). Both require `write_messages`.
5. **Read the thread back.** `conversation.index` (`GET /v4/conversations`) to find the
   conversation, then `message.index` (`GET /v4/conversations/{conversation_uid}/messages`) —
   cursor paginated, `limit` 1–100 (default 10), follow `metadata.nextCursor`.
6. **Track delivery with webhooks, not polling.** Subscribe to `message.sent`, `message.received`
   and `message.failed` (`webhook.create`). Verify every delivery with the
   `podium-timestamp` / `podium-signature` HMAC-SHA256 pair.

## Rules

- **No idempotency.** `message.send` has no idempotency key. A retry after a timeout sends a
  second text. Confirm via `message.index` before resending.
- **10DLC / A2P.** Outbound messages to mobile devices from a test account will not be received.
  Test copy and error handling in a test account; test real delivery only on a registered
  production account.
- **Rate limit** 300 requests/minute; on exhaustion the API returns the `rate_limit` error code.
- **Errors** come back as `{"code","message","moreInfo"}` — `unauthorized` almost always means the
  token is missing `write_messages`, not that the contact is wrong.
