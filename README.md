# Externa Bruno collection

Bruno collection for Externa's:

- Public CMS REST API at `/api/v1`
- GraphQL endpoint at `/api/graphql`
- A couple of admin-only pack routes
- One outbound-webhook docs helper (`Verify Webhook Signature` — not an Externa route; see [Webhooks helper](#webhooks-helper))

Docs: [Outbound webhooks](https://github.com/qiick-io/externa-docs/blob/main/src/app/docs/webhooks/page.md).

`package.json` currently declares version `1.0.0-beta.1`. `bruno.json` `"version": "1"` is Bruno's collection format version, not the package semver.

## Open in Bruno

1. Install [Bruno](https://www.usebruno.com/).
2. Open this folder as a collection.
3. Select the **Local** environment (`environments/Local.bru`).
4. Provide secrets (see below), then send requests or run the collection / folder to execute tests.

## Secrets

`api_key` and `webhook_secret` must not sit in git as plaintext. Bruno documents two approaches; this collection uses the one that keeps **Local** reliably visible.

### Recommended: collection `.env` (what Local uses)

1. Copy `.env.sample` → `.env` in the collection root (already gitignored via `.env*`).
2. Set `EXTERNA_API_KEY` / `EXTERNA_WEBHOOK_SECRET`.
3. `environments/Local.bru` reads them as `{{process.env.EXTERNA_API_KEY}}` and `{{process.env.EXTERNA_WEBHOOK_SECRET}}`.

See [DotEnv File](https://docs.usebruno.com/secrets-management/dotenv-file).

### Official `vars:secret` (UI Secrets tab) — why it is not shipped empty

Per [Secret Variables](https://docs.usebruno.com/secrets-management/secret-variables), marking a var as secret writes only the **name** into the env file:

```bru
vars:secret [
  api_key
  webhook_secret
]
```

Values live in Bruno’s local encrypted store (Secrets tab from v4+), not in the `.bru` file. That is correct — but **empty** secret entries have historically made environments disappear on load (Bruno decrypt / empty-secret bugs). An earlier attempt with an empty `vars:secret` block hid **Local** in the UI.

So this repo does **not** commit an empty `vars:secret` template. If you prefer the Secrets tab: paste non-empty values in the env UI first, then check **Secret**; Bruno may rewrite `Local.bru` locally — keep those secret values out of commits.

## Tests

Every Public API request that hits a real core route (plus the webhook docs helper) has a `tests { }` block (Chai via Bruno). They check status ranges and JSON shape (`data` / `meta` / GraphQL `data|errors`), and skip strict field checks when the response is a client error (missing local ids, empty DB, etc.).

**List Collections Anonymous** accepts `200` or origin-gate `403`. **Verify Webhook Signature** asserts `404` on the intentional placeholder path.

Assert vs tests: Bruno Assert is declarative (`res.status` equals `200`) and fine for fixed happy-paths. These endpoints often need conditionals (200 vs 404, empty lists, GraphQL errors), so the collection uses tests only.

### Origin gate smoke (when allowlist is on)

With Settings → Project → Allowed origins non-empty:

1. Ensure `EXTERNA_API_KEY` is set in collection `.env`.
2. Run Bearer-authenticated Public API / GraphQL requests → expect success (subject to grants).
3. **List Collections Anonymous** (`auth: none`, no Origin) → expect `403` with `"Origin required or API key."`.
4. Optional: add request header `Origin: https://your-allowed-origin` on an anonymous request → expect past the origin gate (still subject to `public` role grants).

With allowlist empty (default Local), anonymous and Bearer behave as before the gate.

### Run tests in Bruno UI

1. Select **Local**, ensure `.env` (or pasted) secrets are set when you need auth.
2. Open a request → **Tests** tab to inspect/edit; send the request and check results under the response.
3. Or use **Run** on a folder / the collection to execute many requests and their tests together.

Optional CLI: `npx @usebruno/cli run . --env Local` (requires Bruno CLI + a reachable `base_url`).

## Environment and auth

### Default environment variables

The shipped **Local** environment defines only shared values:

| Variable | Example | Used by |
| --- | --- | --- |
| `base_url` | `http://externa-core.test` | All requests |
| `api_key` | via `.env` `EXTERNA_API_KEY` | Public API and GraphQL requests using Bearer auth |
| `webhook_secret` | via `.env` `EXTERNA_WEBHOOK_SECRET` | Webhook signature helper |

Request-specific values like `collection_slug`, `item_id`, `file_id`, and `include` live in each affected request’s `vars:pre-request` block (Bruno’s request vars), not in the environment.

### Item `?include=` (Public REST) — typical site pattern

Item list/show responses are **lean unless you opt in**. A frontend should almost never N+1 files or relations: pass expansions in **one** `GET` via `?include=`.

| Token | Effect |
| --- | --- |
| `files` | Expand `image` / `file` / `files` (and block file slots) → each expanded file has `url` |
| `users` | Populate `user_created` / `user_updated` mini-objects |
| *field name* | Depth-1 hydrate for relation / m2a fields (e.g. `author`, `categories`) |

| Pattern | `include` | When |
| --- | --- | --- |
| Slim | omit / empty | IDs only, you’ll fetch files separately (usually worse) |
| Media + editors | `files,users` | **Bruno default** on List/Get Item (+ filter list examples) |
| Typical page | `files,users,<relations>` | List or detail with authors + linked items in one round-trip |

**Breaking:** clients that expected automatic file expansion must pass `include=files`. Change the request var (e.g. `files,author,users`) — do not invent extra REST routes for expansions. Full docs: externa-docs [Expand with `include`](https://github.com/qiick-io/externa-docs/blob/main/src/app/docs/public-cms-api/page.md).

### Origin allowlist (project setting)

Externa can restrict the Public API with project setting **`public_api_allowed_origins`** (Settings → Project → Public API).

| Allowlist | What Bruno needs |
| --- | --- |
| Empty (default local) | Anonymous `auth: none` requests work when the `public` role has grants; CORS from `CORS_ALLOWED_ORIGINS` |
| Non-empty | Calls **without** an `Origin` header (normal Bruno/curl) need a **valid `api_key`**. Folder Bearer auth covers most requests; **List Collections Anonymous** will **403** (`Origin required or API key.`) unless you add a key or set a matching Origin for experiments |

Matching browser Origin alone still does not bypass collection/file grants. Origin is spoofable — for strong protection use key + optional IP allowlist. Full docs in **externa-docs**: [`/docs/public-cms-api-types#origin-allowlist`](https://github.com/qiick-io/externa-docs/blob/main/src/app/docs/public-cms-api-types/page.md) (Origin allowlist section). Client TypeScript / OpenAPI / JSON Schema: same docs page + `public/client-types/*`.

### Local environment notes

- `base_url` defaults to `http://externa-core.test` (Herd). No trailing slash.
- Always keep `EXTERNA_API_KEY` in collection `.env` when developing against an install with origins configured — otherwise REST/GraphQL smoke fails with 403 before permission checks run.
- Do not commit real keys. Prefer `.env` over pasting secrets into `Local.bru`.

### Auth modes in the collection

| Area | Auth in `.bru` files | What it means |
| --- | --- | --- |
| `PublicApi/` | folder-level Bearer `{{api_key}}` | Role attached to the API key |
| `PublicApi/List Collections Anonymous` | `auth: none` | Uses the `public` role |
| `PublicApi/Webhooks/Verify Webhook Signature` | `auth: none` | Docs helper only — placeholder URL 404s on externa-core; point at **your** receiver |
| `Admin/` | request says `auth: inherit`, docs require session cookie | Intended for a logged-in admin browser/session, not Public API Bearer auth |

### Permission model summary

- Public REST and GraphQL access is driven by role grants, not Spatie admin permissions.
- Collection actions are `create`, `read`, `update`, `delete`.
- File actions are `create`, `read`, `read_private`, `update`, `delete`.
- Missing grant means deny.
- Field ACL and `item_filter` still apply to REST and GraphQL reads/writes.

### Files access behavior

| Role grants | Public file | Private file |
| --- | --- | --- |
| no `read` | `403` | `403` |
| `read` only | `200` | hidden from list, `403` on direct fetch/content/transform |
| `read` + `read_private` | `200` | `200` |

## Collection structure

The collection contains 28 request files:

- `PublicApi/`: 13 requests
- `PublicApi/Files/`: 7 requests
- `PublicApi/GraphQL/`: 5 requests
- `PublicApi/Webhooks/`: 1 request
- `Admin/`: 2 requests

## Public API: collections and items

### List Collections Anonymous
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections`
- Purpose: List collections readable by the `public` role.
- Important inputs: none; this request intentionally disables auth.
- Notes: returns `200` with an empty `data` array when public read access is not granted.

### List Collections
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections`
- Purpose: List collections readable by the API key's role.
- Important inputs: `api_key`.
- Notes: same endpoint as the anonymous version, but through Bearer auth.

### Get Collection
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}`
- Purpose: Fetch one collection plus its field schema summary.
- Important inputs: `collection_slug`.
- Notes: requires collection `read`; `404` for an unknown slug.

### List Items
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items`
- Purpose: List paginated items in a collection.
- Important inputs: `collection_slug`; query params `page`, `per_page`, optional `include` (default request var `files,users`), optional `locale`, optional `include_all_translations`.
- Notes: `per_page` defaults to `15`. Public REST is **lean by default** — use `include=files` (and/or `users`, relation field names) to expand. Bruno ships `include=files` + a Chai check that any expanded file object has a non-empty `url`.

### List Items Advanced Filter
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items?filter[status][_eq]=published&page=1&per_page=15&include={{include}}`
- Purpose: Example of operator-based filtering.
- Important inputs: `collection_slug`; `include` (default `files,users`); filter params such as `_eq`, `_neq`, `_contains`, `_in`, `_null`, `_nnull`, `_gt`, `_gte`, `_lt`, `_lte`.
- Notes: good reference for the supported filter dialect; same expansion story as **List Items**.

### List Items Filter Contains
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items?filter[title][_contains]=hello&page=1&per_page=15&include={{include}}`
- Purpose: Example of explicit substring filtering.
- Important inputs: `collection_slug`; `include` (default `files,users`); `filter[title][_contains]`.
- Notes: docs say plain `filter[title]=hello` is treated the same way.

### List Items Filter In Null
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items?filter[status][_in]=published,draft&filter[body][_nnull]=1&include={{include}}`
- Purpose: Example combining `_in` and null/not-null filters.
- Important inputs: `collection_slug`; `include` (default `files,users`); comma-separated `_in` values; `_nnull` flag.
- Notes: multiple filter fields are AND-combined.

### Get Item
- Method: `GET`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items/{{item_id}}`
- Purpose: Fetch one item by id.
- Important inputs: `collection_slug`, `item_id`; request var / query `include` (default `files,users`); optional `locale`, optional `include_all_translations`.
- Notes: Without `include=files`, image/file fields stay as IDs. Tests assert expanded file objects (when present) expose `url`.

### Create Item
- Method: `POST`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items`
- Purpose: Create a new item with collection field data.
- Important inputs: `collection_slug`; JSON body in the form `{ "data": { ... } }`.
- Notes: docs show examples for simple fields, blocks fields, GeoJSON map fields, and many-to-many relations with `{ related_item_id, meta }`; empty `{}` only works when no required fields exist.

### Create Item Nested Blocks
- Method: `POST`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items`
- Purpose: Example payload for nested blocks content.
- Important inputs: `collection_slug`; JSON body with block ids/types, nested `blocks`, nested many-to-any style links, and nested many-to-many objects.
- Notes: this is a specialized create example rather than a different endpoint; adjust `related_collection_id` and `related_item_id` to real ids.

### Create Item With Junction Meta
- Method: `POST`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items`
- Purpose: Example payload for many-to-many data that carries junction metadata.
- Important inputs: `collection_slug`; JSON body using objects like `{ "related_item_id": 1, "meta": { "sort": 1 } }`.
- Notes: docs mention bare integer ids are still accepted and normalized to empty metadata.

### Update Item
- Method: `PATCH`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items/{{item_id}}`
- Purpose: Partially update an existing item.
- Important inputs: `collection_slug`, `item_id`; JSON body in the form `{ "data": { ... } }`.
- Notes: docs include a blocks-field example; validation failures return `422`.

### Delete Item
- Method: `DELETE`
- URL: `{{base_url}}/api/v1/collections/{{collection_slug}}/items/{{item_id}}`
- Purpose: Delete an item.
- Important inputs: `collection_slug`, `item_id`.
- Notes: expected success is `204`; docs note soft-delete semantics if the model uses `SoftDeletes`.

## Public API: files

### List Files
- Method: `GET`
- URL: `{{base_url}}/api/v1/files?search=&parent_id=&page=1&per_page=15`
- Purpose: List files visible to the current role.
- Important inputs: query params `search`, `parent_id`, `page`, `per_page`.
- Notes: private files are omitted from the list unless the role also has `read_private`.

### Get File
- Method: `GET`
- URL: `{{base_url}}/api/v1/files/{{file_id}}`
- Purpose: Fetch file metadata.
- Important inputs: `file_id`.
- Notes: response includes effective `access`, content `url`, and available `transforms`; private files need `read_private`.

### Get File Content
- Method: `GET`
- URL: `{{base_url}}/api/v1/files/{{file_id}}/content`
- Purpose: Stream the original file bytes.
- Important inputs: `file_id`.
- Notes: same access rules as `Get File`; request uses `Accept: */*`.

### Get File Transform
- Method: `GET`
- URL: `{{base_url}}/api/v1/files/{{file_id}}/transforms/{key}`
- Purpose: Fetch a transformed rendition of a file.
- Important inputs: `file_id`; transform key in the URL, such as `thumbnail` or legacy `size-64`.
- Notes: sample request hardcodes `size-64` in the path.

### Upload File
- Method: `POST`
- URL: `{{base_url}}/api/v1/files`
- Purpose: Upload a new file.
- Important inputs: multipart `file` is required; optional `parent_id` and `name`.
- Notes: replace only if you need a larger asset; collection ships `fixtures/sample.png` for smoke runs.

### Update File
- Method: `PATCH`
- URL: `{{base_url}}/api/v1/files/{{file_id}}`
- Purpose: Update file metadata.
- Important inputs: `file_id`; JSON body may include `title`, `description`, `location`, `download_name`, `focal_point_x`, `focal_point_y`, `name`.
- Notes: focal point values are documented as `0-1`.

### Delete File
- Method: `DELETE`
- URL: `{{base_url}}/api/v1/files/{{file_id}}`
- Purpose: Delete a file.
- Important inputs: `file_id`.
- Notes: docs say the file is soft-deleted and success returns `204`.

## Public API: GraphQL

All GraphQL requests post to `{{base_url}}/api/graphql`.

### Query Items
- Method: `POST`
- URL: `{{base_url}}/api/graphql`
- Purpose: Read collection items over GraphQL.
- Important inputs: body contains a `query` plus `variables.slug` and optional `variables.filter`.
- Notes: sample query uses `items(collection: $slug, filter: $filter)` and the filter dialect matches REST advanced filters.

### Query Collections
- Method: `POST`
- URL: `{{base_url}}/api/graphql`
- Purpose: List readable collections over GraphQL.
- Important inputs: body `query`, no variables required in the sample.
- Notes: behaves like the REST collections list for either the public role or the Bearer role.

### Create Item Mutation
- Method: `POST`
- URL: `{{base_url}}/api/graphql`
- Purpose: Create an item using `createItem`.
- Important inputs: `variables.slug`, `variables.data`.
- Notes: same validation and permission model as REST create.

### Update Item Mutation
- Method: `POST`
- URL: `{{base_url}}/api/graphql`
- Purpose: Update an item using `updateItem`.
- Important inputs: `variables.slug`, `variables.id`, `variables.data`.
- Notes: docs call out that field rules and `item_filter` apply here too.

### Delete Item Mutation
- Method: `POST`
- URL: `{{base_url}}/api/graphql`
- Purpose: Delete an item using `deleteItem`.
- Important inputs: `variables.slug`, `variables.id`.
- Notes: docs say delete behavior matches REST soft-delete semantics.

## Webhooks helper

Outbound webhooks are **Externa → your URL** (HMAC-signed). They are not part of the Public CMS API. Full guide: [externa-docs `/docs/webhooks`](https://github.com/qiick-io/externa-docs/blob/main/src/app/docs/webhooks/page.md).

### Verify Webhook Signature
- Method: `POST`
- URL: `{{base_url}}/__webhook_receiver_docs_only__` (**intentional placeholder** — not a real externa-core route; sending as-is **404s**)
- Purpose: Sample envelope + signing checklist so you can practice verifying `X-Externa-Signature` on **your** receiver.
- Important inputs: raw JSON body, `webhook_secret` (same as Project settings; via `.env` `EXTERNA_WEBHOOK_SECRET`), headers `X-Externa-Event-Id`, `X-Externa-Timestamp`, `X-Externa-Signature`.
- How to use:
  1. Replace the URL with webhook.site or your local receiver.
  2. Compute `sha256=` + HMAC-SHA256(raw body, `webhook_secret`) and set `X-Externa-Signature`.
  3. Send — Bruno acts as a **client to your receiver**, not as a caller of Externa.
- Preferred smoke test without Bruno: Settings → Project → set Webhook URL + secret → queue worker → **Send test event** (or mutate an item) → inspect delivery on your URL.
- Events Externa emits: `item.created|updated|deleted|restored`, `file.created|updated|deleted`, `collection.created|updated|deleted`, `ping`.

## Admin routes

These are admin-side routes, not Public API endpoints.

### Apply SEO Inline Field Pack
- Method: `POST`
- URL: `{{base_url}}/collections/{{collection_id}}/field-packs/seo_inline`
- Purpose: Apply the registered `seo_inline` field pack to an existing collection.
- Important inputs: `collection_id` (set it manually on the request; it is not in `environments/Local.bru`).
- Notes: requires a logged-in admin session cookie and `can-edit-collections`; response is a redirect with a flash summary of created vs skipped fields.

### Apply Articles Collection Pack
- Method: `POST`
- URL: `{{base_url}}/collections/packs/articles`
- Purpose: Apply the `articles` collection pack, auto-creating related `seo` and `categories` collections if needed.
- Important inputs: optional body values `name` and `slug` can override the target collection naming.
- Notes: docs mark it as idempotent on existing slugs; requires admin session auth and `can-create-collections`.

## Status codes seen in docs

| Code | Meaning |
| --- | --- |
| `200` / `201` / `204` | Success |
| `401` | Invalid or revoked key, IP not allowlisted, or unsupported key role |
| `403` | Missing collection/file/admin permission; or origin gate when allowlist is set (`Origin not allowed.` / `Origin required or API key.`) |
| `404` | Unknown slug, id, or route target |
| `422` | Validation failure |
| `429` | Rate limiting |

## Client types

TypeScript, OpenAPI, and JSON Schema for Public CMS responses live in **externa-docs** (single source of truth):

- Page: [`public-cms-api-types`](https://github.com/qiick-io/externa-docs/blob/main/src/app/docs/public-cms-api-types/page.md) (includes Origin allowlist)
- Artifacts: `public/client-types/types.ts`, `openapi.yaml`, `schema.json`

This collection does not duplicate those files.
