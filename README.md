# Externa Bruno collection

Classic Bruno collection for the **Public CMS API** (`/api/v1`) and **GraphQL** (`/api/graphql`) — same layout style as Klover (`bruno.json` + folders + `.bru`).

Full product docs: sibling repo **externa-docs** → `/docs/public-cms-api`, `/docs/security-checklist`.

## Open in Bruno

1. Install [Bruno](https://www.usebruno.com/)
2. **Open Collection** → select this folder (`externa-bruno`; must contain `bruno.json`)
3. Select environment **Externa Local**
4. Set variables:

| Variable | Example | Purpose |
| --- | --- | --- |
| `base_url` | `http://externa-core.test` | App origin (no trailing slash) |
| `api_key` | `ek_…` | From Admin → **API Keys** (shown once at create) |
| `collection_slug` | `posts` | Collection slug |
| `item_id` | `1` | Existing item id for get/update/delete |
| `file_id` | `1` | Existing file id for get/content/transform/patch/delete |
| `webhook_secret` | *(optional)* | Project webhook secret for `Webhooks/Verify Webhook Signature` HMAC helper |

## How access works (short)

| Call style | Auth in Bruno | Permissions |
| --- | --- | --- |
| Anonymous | `List Collections Anonymous` (`auth: none`) | Role **`public`** → Collection access matrix |
| With key | Other requests in `PublicApi` (`auth: inherit` → Bearer `{{api_key}}`) | Role linked to that API key |

**Missing grant = deny** (403). Spatie admin permissions (`can-edit-collections`, etc.) do **not** apply to `/api/v1` or `/api/graphql`.

Field ACL + `item_filter` on the role also apply to REST and GraphQL responses/writes.

### Roles matrix (Files)

| Files access | Public file | Private file (effective) |
| --- | --- | --- |
| no `read` | **403** | **403** |
| `read` only | **200** | list omit / get+content **403** |
| `read` + `read_private` | **200** | **200** |

Item field expand: private without `read_private` → `null` / omitted (no id). See externa-docs `/docs/public-cms-api#public-vs-private-files`.

### Typical setup

1. Admin → **Roles** → edit **`public`** → enable **Read** (✓) on the collections you want public → Save  
2. Same role form → **Files access** → enable **Read** (✓); leave **Read private** denied (✕) unless the anonymous site must see private assets → Save  
3. (Optional) Create a custom role with Create/Update/Delete and/or **Read private** → **API Keys** → create key bound to that role → paste into `api_key`  
4. Run requests in Bruno — try a public `file_id` (200) vs a private one without `read_private` (403)

## PublicApi requests

| Request | Method | Path | Auth | Required action |
| --- | --- | --- | --- | --- |
| List Collections Anonymous | `GET` | `/api/v1/collections` | none | *(lists only readable)* |
| List Collections | `GET` | `/api/v1/collections` | Bearer | *(lists only readable)* |
| Get Collection | `GET` | `/api/v1/collections/{{collection_slug}}` | Bearer | `read` |
| List Items | `GET` | `/api/v1/collections/{{collection_slug}}/items` | Bearer | `read` |
| List Items Advanced Filter | `GET` | `…/items?filter[status][_eq]=…` | Bearer | `read` |
| List Items Filter Contains | `GET` | `…/items?filter[title][_contains]=…` | Bearer | `read` |
| List Items Filter In Null | `GET` | `…/items?filter[status][_in]=…` | Bearer | `read` |
| Get Item | `GET` | `/api/v1/collections/{{collection_slug}}/items/{{item_id}}` | Bearer | `read` |
| Create Item | `POST` | `/api/v1/collections/{{collection_slug}}/items` | Bearer | `create` |
| Create Item With Junction Meta | `POST` | same | Bearer | `create` (M2M `{ related_item_id, meta }`) |
| Update Item | `PATCH` | `/api/v1/collections/{{collection_slug}}/items/{{item_id}}` | Bearer | `update` |
| Delete Item | `DELETE` | `/api/v1/collections/{{collection_slug}}/items/{{item_id}}` | Bearer | `delete` |
| GraphQL / Query Items | `POST` | `/api/graphql` | Bearer / none | `read` (+ filter JSON) |
| GraphQL / Query Collections | `POST` | `/api/graphql` | Bearer / none | *(readable collections)* |
| GraphQL / Create Item Mutation | `POST` | `/api/graphql` | Bearer | `create` |
| GraphQL / Update Item Mutation | `POST` | `/api/graphql` | Bearer | `update` |
| GraphQL / Delete Item Mutation | `POST` | `/api/graphql` | Bearer | `delete` |
| List Files | `GET` | `/api/v1/files` | Bearer / none | `file:read` (+ `read_private` to include private) |
| Get File | `GET` | `/api/v1/files/{{file_id}}` | Bearer / none | `file:read` (+ `read_private` if private → else **403**) |
| Get File Content | `GET` | `/api/v1/files/{{file_id}}/content` | Bearer / none | same as Get File |
| Get File Transform | `GET` | `/api/v1/files/{{file_id}}/transforms/{key}` | Bearer / none | same as Get File |
| Upload File | `POST` | `/api/v1/files` | Bearer / none | `file:create` |
| Update File | `PATCH` | `/api/v1/files/{{file_id}}` | Bearer / none | `file:update` |
| Delete File | `DELETE` | `/api/v1/files/{{file_id}}` | Bearer / none | `file:delete` |
| Webhooks / Verify Webhook Signature | `POST` | *(your receiver — docs helper)* | none | HMAC checklist for outbound webhooks |

Admin-only (session cookie, not in this collection): items list **Filters** popover (same `filter[field][_op]` dialect as the requests above), item **History** at `/collections/{id}/items/{item}/revisions`, Jobs monitor at `/settings/jobs`. Outbound webhook config: Settings → Project (see externa-docs `/docs/webhooks`). See externa-docs.

Each `.bru` file has a **docs** panel with expected status codes and body shape.

Folder auth: `PublicApi/folder.bru` sets Bearer `{{api_key}}` for inherited requests.

## Expected status codes

| Code | Meaning |
| --- | --- |
| `200` / `201` / `204` | Success |
| `401` | Invalid/revoked key, IP not allowlisted, or super-admin role on key |
| `403` | Role lacks the collection or file action (includes private file without `read_private`) |
| `404` | Unknown slug or item |
| `422` | Validation error on create/update |
| `429` | Rate limit |

## Structure

```
externa-bruno/
├── bruno.json
├── README.md
├── environments/
│   └── Externa Local.bru
└── PublicApi/
    ├── folder.bru
    ├── *.bru
    ├── GraphQL/
    │   ├── folder.bru
    │   └── *.bru
    ├── Files/
    │   ├── folder.bru
    │   └── *.bru
    └── Webhooks/
        ├── folder.bru
        └── Verify Webhook Signature.bru
```
