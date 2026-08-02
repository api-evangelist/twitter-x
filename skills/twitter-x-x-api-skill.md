---
name: X
description: Use when building applications that interact with X (formerly Twitter) data. Reach for this skill when you need to search posts, manage user interactions, stream real-time data, handle authentication, work with rate limits, or integrate X data into applications.
metadata:
    mintlify-proj: x
    version: "1.0"
---

# X API Skill

## Product summary

The X API provides programmatic access to X's public conversation through REST endpoints. Build applications that search posts, retrieve user data, manage interactions (likes, reposts, follows), stream real-time posts, send direct messages, and analyze trends. The API uses pay-per-usage pricing with Bearer Token (app-only) or OAuth authentication. Key endpoints live at `https://api.x.com/2/`. Primary documentation: https://docs.x.com/x-api/introduction

## When to use

Use this skill when:
- Building search or monitoring tools for posts and users
- Creating applications that publish or manage posts
- Implementing real-time streaming of posts matching filter rules
- Handling user authentication and authorization flows
- Managing user interactions (follows, likes, bookmarks, mutes, blocks)
- Retrieving timelines, mentions, or user profiles
- Implementing direct messaging features
- Analyzing engagement metrics or trends
- Handling rate limits and pagination in API responses
- Debugging authentication or API errors

## Quick reference

### Authentication methods

| Method | Use case | Scope |
|:-------|:---------|:------|
| Bearer Token (app-only) | Public data, no user context | App-wide limits |
| OAuth 1.0a | User-context actions, private data | Per-user limits |
| OAuth 2.0 + PKCE | User authorization flows, web apps | Per-user limits |

Get credentials from [Developer Console](https://console.x.com). Bearer Token is simplest for read-only public data.

### Common endpoints

| Endpoint | Method | Purpose |
|:---------|:-------|:---------|
| `/2/tweets/search/recent` | GET | Search posts from last 7 days |
| `/2/tweets/search/all` | GET | Search complete post archive (pay-per-use only) |
| `/2/tweets/search/stream` | GET | Connect to real-time filtered stream |
| `/2/tweets/search/stream/rules` | POST/GET | Manage stream filter rules |
| `/2/users/by/username/:username` | GET | Look up user by username |
| `/2/users/:id/tweets` | GET | Get user's posts |
| `/2/tweets/:id` | GET | Get post by ID |
| `/2/tweets` | POST | Create a post |
| `/2/users/:id/following` | GET/POST | Manage follows |
| `/2/users/:id/likes` | GET/POST | Manage likes |

### Field and expansion parameters

Always use fields to request additional data—responses are minimal by default:

```bash
# Request specific fields
?tweet.fields=created_at,public_metrics,lang
?user.fields=created_at,description,public_metrics

# Include related objects
?expansions=author_id,attachments.media_keys
```

Common field combinations:
- Post analytics: `tweet.fields=created_at,public_metrics,possibly_sensitive`
- User profiles: `user.fields=created_at,description,location,public_metrics,verified`
- Full context: `expansions=author_id,referenced_tweets.id&user.fields=username,name`

### Rate limit headers

Every response includes:
```
x-rate-limit-limit: 900
x-rate-limit-remaining: 847
x-rate-limit-reset: 1705420800
```

Check `x-rate-limit-remaining` before making requests. When you hit 429, wait until `x-rate-limit-reset` (Unix timestamp).

### Pagination

Use cursor-based pagination for large result sets:

```bash
# First request
?max_results=100

# Check response meta for next_token
# {"meta": {"next_token": "abc123", ...}}

# Next page
?max_results=100&pagination_token=abc123
```

Continue until `next_token` is absent.

## Decision guidance

### When to use search vs. filtered stream

| Scenario | Use search | Use filtered stream |
|:---------|:-----------|:-------------------|
| Historical data | ✓ | — |
| Real-time monitoring | — | ✓ |
| One-time lookup | ✓ | — |
| Continuous listening | — | ✓ |
| Recent 7 days | ✓ (all devs) | ✓ |
| Full archive | ✓ (pay-per-use) | — |
| Complex queries | ✓ | ✓ |

### When to use fields vs. expansions

| Need | Use fields | Use expansions |
|:-----|:-----------|:---------------|
| More data on primary object | ✓ | — |
| Related objects (author, media) | — | ✓ |
| Reduce API calls | — | ✓ |
| Specific metrics | ✓ | — |

### When to use Bearer Token vs. OAuth

| Scenario | Bearer Token | OAuth 1.0a | OAuth 2.0 |
|:---------|:-------------|:-----------|:----------|
| Public data only | ✓ | — | — |
| User-context actions | — | ✓ | ✓ |
| Web app sign-in | — | — | ✓ |
| Mobile app | — | — | ✓ |
| Server-to-server | ✓ | ✓ | — |

## Workflow

### 1. Set up authentication

1. Create a developer account at [console.x.com](https://console.x.com)
2. Create a Project and App
3. Generate credentials:
   - For app-only: Copy Bearer Token from App Settings
   - For user context: Generate API Key, API Secret, Access Token, Access Token Secret
4. Store credentials securely (environment variables, not in code)

### 2. Make your first request

```bash
# Test with user lookup (requires Bearer Token)
curl "https://api.x.com/2/users/by/username/xdevelopers" \
  -H "Authorization: Bearer $BEARER_TOKEN"
```

Check for 200 response with user data in `data` field.

### 3. Request additional fields

```bash
# Add fields parameter to get more data
curl "https://api.x.com/2/users/by/username/xdevelopers?user.fields=created_at,description,public_metrics" \
  -H "Authorization: Bearer $BEARER_TOKEN"
```

### 4. Handle pagination for large result sets

```bash
# Get first page
curl "https://api.x.com/2/users/123/tweets?max_results=100" \
  -H "Authorization: Bearer $BEARER_TOKEN"

# Extract next_token from response meta
# Use it for next request
curl "https://api.x.com/2/users/123/tweets?max_results=100&pagination_token=TOKEN" \
  -H "Authorization: Bearer $BEARER_TOKEN"
```

### 5. Implement error handling

Check HTTP status code first:
- 200/201: Success
- 400: Fix your request (invalid params, malformed JSON)
- 401: Fix authentication (invalid token, wrong method)
- 403: Check app permissions (endpoint access, user context required)
- 404: Resource doesn't exist
- 429: Rate limited—wait until `x-rate-limit-reset`
- 5xx: Server error—retry with backoff

### 6. For streaming: set up filtered stream

```bash
# Add a rule
curl -X POST "https://api.x.com/2/tweets/search/stream/rules" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"add": [{"value": "from:xdevelopers"}]}'

# Connect to stream
curl "https://api.x.com/2/tweets/search/stream" \
  -H "Authorization: Bearer $TOKEN"
```

Keep connection open. Implement reconnection logic for disconnects.

## Common gotchas

- **Minimal responses by default**: Always add `tweet.fields` or `user.fields` or you'll only get `id`, `text`, and `edit_history_tweet_ids`. This is not a bug.
- **Bearer Token vs. OAuth**: Bearer Token is app-only and can't access private data or perform user actions. Use OAuth for those.
- **Rate limits reset in 15-minute windows**: Don't assume they reset at the top of the hour. Check `x-rate-limit-reset` header.
- **Pagination tokens expire**: Don't store them for later use. Get fresh tokens in each session.
- **Stream requires at least one rule**: If you connect without rules, you'll get a 409 Conflict. Always add rules first.
- **Search query length limits**: Recent search is 512 chars; full-archive is 1,024 chars. Longer queries fail silently.
- **Protected accounts**: Posts from protected accounts are only visible if you're authorized. You won't get 403—the post just won't appear.
- **Deleted posts return 404**: Don't retry; the post is gone.
- **Expansions don't reduce API calls**: They're included in the same response but still count as one request. Use them to avoid multiple calls.
- **Field order in responses may differ**: Don't assume fields appear in request order.
- **You can't request subfields**: Requesting `public_metrics` gives you all metrics (likes, reposts, replies, quotes, bookmarks, impressions). You can't request just `public_metrics.like_count`.

## Verification checklist

Before submitting work with X API integration:

- [ ] Authentication credentials are stored in environment variables, not hardcoded
- [ ] Bearer Token or OAuth method matches the endpoint's requirements
- [ ] All requests include required `Authorization` header
- [ ] Field parameters are included for any data beyond `id` and `text`
- [ ] Pagination is implemented for endpoints that return multiple results
- [ ] Rate limit headers are checked before making requests
- [ ] Error responses are handled (check HTTP status code first, then error type)
- [ ] 429 responses trigger a wait until `x-rate-limit-reset`, not immediate retry
- [ ] Filtered stream rules are added before connecting to the stream
- [ ] Stream reconnection logic handles disconnects gracefully
- [ ] Expansions and fields are combined correctly (expansions include objects, fields request data on those objects)
- [ ] Query syntax is valid (operators, quotes, parentheses)
- [ ] Response parsing handles both `data` and `includes` sections

## Resources

**Comprehensive navigation**: https://docs.x.com/llms.txt

**Critical documentation pages**:
1. [X API Introduction](https://docs.x.com/x-api/introduction) — Overview, pricing, key features
2. [Make Your First Request](https://docs.x.com/make-your-first-request) — Quickstart with cURL examples
3. [Authentication Overview](https://docs.x.com/fundamentals/authentication/overview) — Auth methods and setup
4. [Rate Limits](https://docs.x.com/x-api/fundamentals/rate-limits) — Per-endpoint limits and handling 429s
5. [Fields & Expansions](https://docs.x.com/x-api/fundamentals/fields) — Customizing response data
6. [Search Posts](https://docs.x.com/x-api/posts/search/introduction) — Recent and full-archive search
7. [Filtered Stream](https://docs.x.com/x-api/posts/filtered-stream/introduction) — Real-time streaming
8. [Response Codes & Errors](https://docs.x.com/x-api/fundamentals/response-codes-and-errors) — Error handling

---

> For additional documentation and navigation, see: https://docs.x.com/llms.txt