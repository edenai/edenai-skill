# Keys, usage, governance and data residency

The account side of Eden AI, for when the task is about spending, keys, teams or compliance rather than calling a model. Sources are the docs' General, Organization and Data governance sections.

## Key types

| Key | Prefix | Calls | Get one |
|---|---|---|---|
| Inference key (`api_token`) | `sk-eden-…` | The model APIs, billed to the organization | Dashboard **API Keys**, or `POST /v3/manage/keys/` |
| Sandbox key (`sandbox_api_token`) | | The same APIs, returning free **mock** data in the real shape | Dashboard (choose **Sandbox**), or `"token_type": "sandbox_api_token"` |
| Management auth key | `mgmt-eden-…` | Only `/v3/manage/*`, with scopes `manage:read` and `manage:write` | Dashboard **Account → Management Keys**, or minted by an issuer key |
| Management issuer key | `mgmt-eden-…` | Only minting auth keys (`POST /v3/manage/auth-keys/`). Store it like a root secret | Dashboard only |

An inference key on `/v3/manage/*` returns 401 with "This endpoint needs a management key". Sandbox keys are the right choice for CI, but don't use them to judge model quality.

## Tags: cost per customer, project or environment

- **How to send them:**
  - The `X-EdenAI-Tags: client=acme,env=prod` header works on every endpoint that runs a model, multipart ones included.
  - A `tags: {"client": "acme"}` body field works on JSON endpoints, except `/v3/videos`, which refuses it.
  - An API key's `default_tags` are added to every call made with that key.
- **Merging:** tags merge by key. The body wins over the header, which wins over the key defaults.
- **Rules:**
  - At most 10 tags after merging.
  - Keys are 1–64 characters and stored in lowercase. Values are 1–128 characters and kept as sent.
  - Allowed characters are letters, digits and `- _ . : /`. Values are strings.
  - `eden:` is reserved.
  - An invalid tag returns 400 before any provider is called, so it isn't billed.
- **What to put in them:** a small, stable set of values. No personal data, and no per-request ids.
- **Reading them back:** `x-edenai-metadata: enabled` shows the recorded tags in `edenai_metadata.tags`. The dashboard's **Monitoring** filters and groups by tag, and exports CSV.
- **Not tags:** OpenAI's `metadata` and `user` fields, and `/v2` endpoints, which ignore tags. Tags are never sent to the provider.

## Management API: `/v3/manage/*`

Call it with `Authorization: Bearer <management key>`. The OpenAPI spec is at https://api.edenai.run/v2/info/splitted-schema/management_api/openapi.json, whose paths are under `/v3/manage`.

| Endpoint | Scope | What it does |
|---|---|---|
| `GET /v3/manage/whoami/` | any | The organization, scopes and expiry of the calling key. A good connectivity check |
| `GET/POST /v3/manage/keys/` | read / write | List keys, including revoked ones, paginated with `limit` ≤ 100 and `offset`. Create a key, which returns `secret` **once** |
| `GET/PATCH/DELETE /v3/manage/keys/{key_id}/` | read / write | Inspect, update (name, budget, `default_tags`) or revoke permanently |
| `POST /v3/manage/keys/{key_id}/rotate/` | write | Get a new secret. The old one stops working immediately |
| `GET /v3/manage/keys/{key_id}/usage/` | read | One key's usage as a time series |
| `GET /v3/manage/usage/` | read | Organization usage per key, or per member with `group_by=user` |
| `GET /v3/manage/members/`, `PATCH /v3/manage/members/{email}/role/` | read / write | Members, and role changes (needs RBAC) |
| `GET /v3/manage/groups/[{id}/]` | read | Directory-synced groups, read-only |
| `GET/POST /v3/manage/auth-keys/`, `DELETE /v3/manage/auth-keys/{id}/` | `manage:mint` | Management keys |

**Key fields on create:**
- `name`: required, and unique among active keys.
- `token_type`: `api_token` or `sandbox_api_token`. It can't be changed later.
- `member`: an email, to attribute the key's usage to a person.
- `balance` and `active_balance`: a hard spending cap. The key stops at $0.
- `balance_reset_period` (`none`, `daily`, `weekly` or `monthly`) with `balance_reset_amount`: a periodic budget. Resets happen at midnight Europe/Paris.
- `expire_time`: at least a day ahead.
- `default_tags`.

**Usage query parameters:**
- `step` is required: 1 daily, 2 weekly, 3 monthly, 4 yearly.
- `begin` and `end` are `YYYY-MM-DD`, with `end` exclusive. They default to the last 7 days, with at most 366 days.
- Filters: `feature`, `subfeature`, `provider`, `phase`, `billing` (`eden`, `own_keys` or `all`), `token`, `key_id` and `user`.

**Usage response:**

```json
{"usage": [{"token": "production-v1", "data": {"2026-09-01": {"text__chat": {"total_cost": 11.3, "details": 381, "cost_per_provider": {"openai": 11.28}}}}}]}
```

- **Feature keys** look like `text__chat`, `ocr__ocr` or `audio__text_to_speech`.
- **Credit balance:** the account's balance isn't exposed by the API. Check it in the dashboard.
- **The old `/v2/info/splitted-schema/cost_management/` endpoints** are no longer documented, and the one we tried returned 405. Use `/v3/manage/usage/`.

## Pricing and limits

- **Pricing:** the provider's own price with no markup, plus a **5.5% platform fee** on the self-serve plan. The Advanced plan has custom pricing, bulk discounts, private deployments and an SLA. Credits are prepaid through Stripe, with optional auto-refill.
- **Rate limits:** **10 requests per second** per account by default, shared by all its keys. The pricing page says 7, but the rate-limits page says 10. It can be raised through live chat. In an organization, the owner's limit is the ceiling, and members can be capped lower.
- **Response caching:** identical requests (same model and input) are served free from cache. It's on by default, with a per-project toggle in the dashboard. It's scoped by region.

## BYOK (bring your own provider keys)

- **Setup:** add provider keys in the dashboard under **Settings → Bring Your Own Keys**. They apply to the whole organization.
- **Billing:** requests to that provider then use your key and are billed by the provider. Eden AI's `cost` becomes an estimate from public prices.
- **What you keep:** routing, fallbacks, monitoring and the unified format still work, and you can mix BYOK and Eden-billed providers.
- **Checking it:** `edenai_metadata.is_byok` confirms your key was used, and `billing=own_keys` filters usage to BYOK traffic.

## Guardrails (organization plans)

- **What a guardrail is:** a reusable policy attached to a role, a member or an API key. It's created in the dashboard under **Settings → Organization → Guardrails**.
- **What it controls:**
  - **Model rules:** a default allow or deny, plus ordered rules on patterns like `openai/gpt-latest`, `anthropic/*` or `*/*`, optionally per feature.
  - **Rate limits:** written `N/second`, `N/minute`, `N/hour` or `N/day`.
  - **A member budget:** a soft, periodic cap.
- **Blocked requests:** a request that hits a deny rule is rejected.

## Data residency

- **The EU endpoint:** `https://api.eu.edenai.run` takes the same key, bodies and responses.
  - **Discovery:** `/v3/models` and `/v3/info` on that host list only EU-eligible models and providers, and each model's `regions` field shows where it runs.
  - **Rejections:** a non-EU model, or a non-EU `fallbacks` entry, returns **451** before any provider is contacted. It's never silently re-routed out of the EU.
- **Per request:** `model@eu` pins a region on the global endpoint.
- **Hosting and retention:**
  - Eden AI runs in European data centers.
  - A static egress IP is available for fetching your file URLs, so you can whitelist it.
  - Each provider's training and retention policy, including zero-data-retention options, is listed at https://www.edenai.co/docs/v3/data-governance/provider-data-policies.
