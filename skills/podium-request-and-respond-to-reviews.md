---
name: Request a review and respond to it
description: Send a review invite from a Podium location, watch for the review to land, and post or update the business response — the full local-reputation loop.
api: openapi/podium-reviews-openapi.yml
operations: [review_invite.create, review_invite.index, review_invite.get, review.index, review.get, review_response.create, review_response.update, review_summary.index, review_sites_summary.index]
scopes: [read_reviews, write_reviews]
---

# Request a review and respond to it (Podium)

Base URL `https://api.podium.com/v4/`, OAuth bearer token, `podium-version: 2021.04.01`.

## Steps

1. **Send the invite.** `review_invite.create` (`POST /v4/reviews/invites`) creates a review
   invite link for a contact at a location. Requires `write_reviews`.
2. **Track invites.** `review_invite.index` (`GET /v4/reviews/invites`) and `review_invite.get`
   (`GET /v4/reviews/invites/{uid}`), both `read_reviews`.
3. **Detect the review.** Prefer webhooks — `review.created`, `review.updated`,
   `review.invite_link_created`, `review.invite_link_updated` — over polling `review.index`
   (`GET /v4/reviews`, cursor paginated with `limit` and `cursor`).
4. **Read the review.** `review.get` (`GET /v4/reviews/{uid}`).
5. **Respond.** `review_response.create` (`POST /v4/reviews/{uid}/responses`); correct a published
   response with `review_response.update`
   (`PATCH /v4/reviews/{review_uid}/responses/{uid}`). Both require `write_reviews`.
6. **Attribute internally.** `review_attribution.create` (`POST /v4/reviews/{uid}/attributions`)
   credits a Podium user with the review; `review_attribution.delete` removes it.
7. **Report.** `review_summary.index` (`GET /v4/reviews/summary`) and
   `review_sites_summary.index` (`GET /v4/reviews/sites/summary`) give the aggregate rating and
   the per-site connection state.

## Rules

- Responses are published to third-party review sites through Podium; treat
  `review_response.create` as a public, non-reversible write and gate it behind human approval.
- No idempotency key exists on any review write — check `review_response.index`
  (`GET /v4/reviews/{uid}/responses`) before retrying a failed create.
- Watch `review.response_created` / `review.response_updated` webhooks to confirm the write landed.
