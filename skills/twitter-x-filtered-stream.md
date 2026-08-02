---
name: Stream real-time Posts from X
description: Set up filtered-stream rules and consume the real-time Post stream, with reconnection discipline and connection management.
api: openapi/twitter-x-x-api-v2-openapi.json
operations: [getRules, updateRules, getRuleCounts, streamPosts, getConnectionHistory, deleteAllConnections]
generated: '2026-07-21'
method: generated
---

# Stream real-time Posts from X

## Auth
App-only Bearer token.

## Steps
1. **Add rules first** — `updateRules` (`POST /2/tweets/search/stream/rules`) with `{"add":[{"value":"from:xdevelopers","tag":"my-rule"}]}`. Connecting with zero rules returns 409 `conflict`. Check caps with `getRuleCounts`.
2. **Verify rules** — `getRules` (`GET /2/tweets/search/stream/rules`).
3. **Connect** — `streamPosts` (`GET /2/tweets/search/stream`) and hold the connection open; add `tweet.fields`/`expansions` params for payload richness. One connection per app; 50 connection attempts/15min.
4. **Reconnect with backoff** — on disconnect (problem types `operational-disconnect`, `client-disconnected`, `streaming-connection`) reconnect with exponential backoff; consume fast enough to avoid full-buffer disconnects.
5. **Clean up stuck connections** — `getConnectionHistory` (`GET /2/connections`) and `deleteAllConnections` (`DELETE /2/connections/all`) resolve TooManyConnections errors without waiting for timeouts.

## Rules
- Rule syntax errors return problem types `invalid-rules` / `duplicate-rules` / `noncompliant-rules`; rule caps return `rule-cap` (see errors/twitter-x-problem-types.yml).
- Rule length 512-1,024+ chars and rule count depend on tier (Enterprise Filtered Stream Webhooks supports 25,000+ rules and async webhook delivery instead of a held connection — see asyncapi/twitter-x-webhooks.yml).
- Prefer streaming over polling search for continuous monitoring (rate-limit best practice).
