---
name: Search posts on X
description: Search recent or full-archive Posts on X, with counts, using app-only Bearer auth, cursor pagination, and the fields/expansions sparse-response model.
api: openapi/twitter-x-x-api-v2-openapi.json
operations: [searchPostsRecent, searchPostsAll, getPostsCountsRecent, getPostsCountsAll]
generated: '2026-07-21'
method: generated
---

# Search posts on X

## Auth
Use an app-only Bearer token (`Authorization: Bearer $TOKEN`) from your app in the Developer Console (console.x.com). No user context is needed for search over public Posts.

## Steps
1. **Recent window (last 7 days)** — call `searchPostsRecent` (`GET /2/tweets/search/recent`) with `query` (max 512 chars). Full archive needs `searchPostsAll` (`GET /2/tweets/search/all`, pay-per-use; max 1,024-char query, 1 req/sec).
2. **Ask for the fields you need** — responses are minimal by default (`id`, `text`, `edit_history_tweet_ids`). Add `tweet.fields=created_at,public_metrics,lang` and `expansions=author_id&user.fields=username,name`; expanded objects arrive under `includes`.
3. **Paginate** — pass `max_results` (up to 100; 500 on full archive), then follow `meta.next_token` via `pagination_token` until absent. Results are reverse-chronological.
4. **Volume before content** — use `getPostsCountsRecent` / `getPostsCountsAll` (`/2/tweets/counts/*`) to size a query before pulling Posts.

## Rules
- Respect `x-rate-limit-remaining`; on 429 wait until `x-rate-limit-reset` (Unix seconds). Recent search: 450/15min per app.
- Errors are RFC 9457 problem+json (`type` = `https://api.twitter.com/2/problems/...`); a 200 can still carry a partial `errors[]` alongside `data` — handle both (see errors/twitter-x-problem-types.yml).
- Retweets are not returned in keyword search results (since the 2026-05-04 index migration); use the filtered stream if you need them.
- No idempotency contract exists — searches are safe to retry; do not blind-retry writes.
