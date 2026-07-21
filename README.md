# Externa Bruno collection

Classic Bruno collection for the **Public CMS API** (`/api/v1`) — same layout style as Klover (`bruno.json` + folders + `.bru`).

Full product docs: sibling repo **externa-docs** → page `/docs/public-cms-api`.

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

## How access works (short)

| Call style | Auth in Bruno | Permissions |
| --- | --- | --- |
| Anonymous | `List Collections Anonymous` (`auth: none`) | Role **`public`** → Collection access matrix |
| With key | Other requests in `PublicApi` (`auth: inherit` → Bearer `{{api_key}}`) | Role linked to that API key |

**Missing grant = deny** (403). Spatie admin permissions (`can-edit-collections`, etc.) do **not** apply to `/api/v1`.

### Typical setup

1. Admin → **Roles** → edit **`public`** → enable **Read** (✓) on the collections you want public → Save  
2. Same role form → **Files access** matrix → enable **Read** (and Create/Update/Delete as needed) → Save  
3. (Optional) Create a custom role with Create/Update/Delete → **API Keys** → create key bound to that role → paste into `api_key`  
4. Run requests in Bruno

## PublicApi requests

| Request | Method | Path | Auth | Required action |
| --- | --- | --- | --- | --- |
| List Collections Anonymous | `GET` | `/api/v1/collections` | none | *(lists only readable)* |
| List Collections | `GET` | `/api/v1/collections` | Bearer | *(lists only readable)* |
| Get Collection | `GET` | `/api/v1/collections/{{collection_slug}}` | Bearer | `read` |
| List Items | `GET` | `/api/v1/collections/{{collection_slug}}/items` | Bearer | `read` |
| Get Item | `GET` | `/api/v1/collections/{{collection_slug}}/items/{{item_id}}` | Bearer | `read` |
| Create Item | `POST` | `/api/v1/collections/{{collection_slug}}/items` | Bearer | `create` |
| Update Item | `PATCH` | `/api/v1/collections/{{collection_slug}}/items/{{item_id}}` | Bearer | `update` |
| Delete Item | `DELETE` | `/api/v1/collections/{{collection_slug}}/items/{{item_id}}` | Bearer | `delete` |
| List Files | `GET` | `/api/v1/files` | Bearer / none | `file:read` |
| Get File | `GET` | `/api/v1/files/{{file_id}}` | Bearer / none | `file:read` |
| Get File Content | `GET` | `/api/v1/files/{{file_id}}/content` | Bearer / none | `file:read` |
| Get File Transform | `GET` | `/api/v1/files/{{file_id}}/transforms/{key}` | Bearer / none | `file:read` |
| Upload File | `POST` | `/api/v1/files` | Bearer / none | `file:create` |
| Update File | `PATCH` | `/api/v1/files/{{file_id}}` | Bearer / none | `file:update` |
| Delete File | `DELETE` | `/api/v1/files/{{file_id}}` | Bearer / none | `file:delete` |

Each `.bru` file has a **docs** panel with expected status codes and body shape.

Folder auth: `PublicApi/folder.bru` sets Bearer `{{api_key}}` for inherited requests.

## Expected status codes

| Code | Meaning |
| --- | --- |
| `200` / `201` / `204` | Success |
| `401` | Invalid/revoked key, IP not allowlisted, or super-admin role on key |
| `403` | Role lacks the collection or file action |
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
    └── Files/
        ├── folder.bru
        └── *.bru
```
