---
name: Sync contacts, attributes and tags
description: Keep an external CRM and Podium contacts in step — create and update contacts, manage the organization-level attribute and tag vocabularies, and apply them per contact.
api: openapi/podium-contacts-openapi.yml
operations: [contact.index, contact.get, contact.create, contact.update, contact.delete, contact_entity_attribute.index, contact_entity_attribute.create, contact_entity_attribute.update, contact_entity_attribute.delete, contact_entity_tag.list, contact_entity_tag.create, contact_entity_tag.update, contact_attribute.create, contact_attribute.update, contact_attribute.delete]
scopes: [read_contacts, write_contacts]
---

# Sync contacts, attributes and tags (Podium)

Base URL `https://api.podium.com/v4/`, OAuth bearer token, `podium-version: 2021.04.01`.
Contacts are the only Podium resource addressable by a business identifier (`{identifier}`,
phone/email based) rather than only a `uid`.

## Steps

1. **Define the vocabulary once, at the organization level.**
   - Attributes: `contact_entity_attribute.index` (`GET /v4/contact_attributes`),
     `contact_entity_attribute.create` (`POST /v4/contact_attributes`),
     `contact_entity_attribute.get`, `contact_entity_attribute.update`,
     `contact_entity_attribute.delete`.
   - Tags: `contact_entity_tag.list` (`GET /v4/contact_tags`), `contact_entity_tag.create`
     (`POST /v4/contact_tags`), `contact_entity_tag.get`, `contact_entity_tag.update`.
2. **Page the existing contacts.** `contact.index` (`GET /v4/contacts`) — cursor pagination,
   `limit` 1–100 (default 10), follow `metadata.nextCursor` until it is absent. An expired cursor
   returns `invalid_cursor`; restart the page run without the cursor.
3. **Upsert.** `contact.get` (`GET /v4/contacts/{identifier}`) then `contact.update`
   (`PATCH /v4/contacts/{identifier}`), or `contact.create` (`POST /v4/contacts`).
4. **Apply per-contact values.** `contact_attribute.create`
   (`POST /v4/contacts/{identifier}/attributes/{uid}`), `contact_attribute.update`,
   `contact_attribute.delete`; tags are attached with
   `POST /v4/contacts/{identifier}/tags/{uid}` and removed with the DELETE on the same path.
5. **Respect consent.** `contact.opt_in` (`POST /v4/contacts/campaigns/opt_in`) and
   `contact.opt_out` (`POST /v4/contacts/campaigns/opt_out`) govern campaign messaging. Never
   infer consent from the presence of a phone number.
6. **Stay in step with webhooks.** `contact.created`, `contact.updated`, `contact.deleted`,
   `contact.merged`, `contact.unchanged`. `contact.merged` matters: two of your CRM records may
   now be one Podium contact.

## Rules

- Two operations share the operationId `contact_tag.create` in Podium's published spec — the POST
  (add tag) and the DELETE (remove tag) on `/v4/contacts/{identifier}/tags/{uid}`. Bind by
  **method + path**, not by operationId, or you will call the wrong one.
- No idempotency key on any contact write. Deduplicate on your side by identifier.
- 300 requests/minute; a bulk sync must throttle or it will collect `rate_limit` errors.
