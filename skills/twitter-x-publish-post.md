---
name: Publish and manage a Post on X
description: Create, edit, and delete Posts with OAuth 2.0 user context, including the summoned-reply restriction, paid-partnership disclosure, and error handling.
api: openapi/twitter-x-x-api-v2-openapi.json
operations: [getUsersMe, createPosts, deletePosts, hidePostsReply]
generated: '2026-07-21'
method: generated
---

# Publish and manage a Post on X

## Auth
Writes require user context: OAuth 2.0 Authorization Code + PKCE with scopes `tweet.read tweet.write users.read` (add `offline.access` for a refresh token). Confirm identity first with `getUsersMe` (`GET /2/users/me`).

## Steps
1. **Create** — `createPosts` (`POST /2/tweets`) with `{"text": "..."}`. Expect `201` and the new Post id. Costs $0.015 per Post on pay-per-use ($0.20 if the Post contains a URL).
2. **Edit (Premium only)** — call `createPosts` again with `edit_options.previous_post_id`; only your own Posts, created within the last hour.
3. **Disclose paid partnership** — set `"paid_partnership": true` when the Post contains paid promotion.
4. **Delete** — `deletePosts` (`DELETE /2/tweets/{id}`); 50/15min per user.
5. **Moderate replies** — `hidePostsReply` (`PUT /2/tweets/{tweet_id}/hidden`) with scope `tweet.moderate.write`.

## Rules
- Programmatic replies are only permitted when the target author summoned you (@mention or quote) on self-serve tiers — otherwise expect a 403 `client-forbidden` problem.
- There is no Idempotency-Key: a timed-out `createPosts` may or may not have landed. Reconcile by reading your own recent Posts (`getUsersPosts`) before retrying.
- Rate limits: `POST /2/tweets` 100/15min per user, 10,000/24h per app. On 429 check whether the problem `type` is `rate-limit-exceeded` (wait for `x-rate-limit-reset`) or `usage-capped` (plan cap — do not retry).
- Character validation should match the platform: use the first-party twitter-text library.
