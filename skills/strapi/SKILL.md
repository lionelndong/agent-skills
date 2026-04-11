---
name: strapi
description: Create, update, draft, publish, and attach media to Strapi v5 Cloud content via the REST API. Use this skill whenever you are working with Strapi — creating blog posts, articles, or any content-type entries; uploading images or files; attaching media to entries; or calling any endpoint on a Strapi Cloud project (*.strapiapp.com). Strapi v5 has silent footguns that this skill prevents — drafts publish by default on REST writes, updates to drafts auto-publish them, and the upload field name is specific. Consult this skill even for tasks that look trivial, because the common mistakes produce responses that look successful until a human checks the CMS.
metadata:
  author: Lionelndong
  version: "1.0.0"
---

# Strapi v5 Cloud REST API

Use this skill whenever you call `*.strapiapp.com/api/*`. It prevents the two failure modes agents hit over and over on Strapi v5:

1. **Drafts that silently publish.** Omitting `?status=draft` publishes on create. Updating an existing draft without the param publishes it too. Neither returns an error.
2. **Uploads that 400 or never attach.** The upload field is `files` (plural), body must be multipart, and the attach step uses the numeric `id` — not `documentId`.

The rest of this doc is the commands and rules to never hit those again.

## The two rules that prevent 90% of mistakes

### Rule 1 — `?status=draft` is a URL param, and you need it on EVERY write

`publishedAt: null` in the body is a v4 trick. **v5 silently ignores it.** `status: "draft"` inside `data` is also ignored. The only thing Strapi v5 reads is the `?status` query string.

```bash
# WRONG — publishes the post, looks successful, no error
curl -X POST "$BASE/api/articles" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"title":"Hello","publishedAt":null}}'

# RIGHT — creates a draft
curl -X POST "$BASE/api/articles?status=draft" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"title":"Hello"}}'
```

And the worse one — **updating a draft without `?status=draft` auto-publishes it** (Strapi issue #24547):

```bash
# WRONG — this PUT publishes the draft you were trying to edit
curl -X PUT "$BASE/api/articles/$DOCID" -d '{"data":{"title":"edit"}}'

# RIGHT — stays a draft
curl -X PUT "$BASE/api/articles/$DOCID?status=draft" -d '{"data":{"title":"edit"}}'
```

Default every Strapi write to `?status=draft`. Only drop it when a human has explicitly said "publish it".

### Rule 2 — The upload field is `files`, multipart only, and attach uses the numeric `id`

```bash
# WRONG — 400 "files field is required"
curl -X POST "$BASE/api/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@cover.jpg"

# WRONG — 415 "Unsupported Media Type"
curl -X POST "$BASE/api/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"files":"cover.jpg"}'

# RIGHT
curl -X POST "$BASE/api/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "files=@cover.jpg"
```

Response is a **bare array**, not `{data: ...}`:

```json
[{"id": 42, "documentId": "kq9...","url": "https://...media.strapiapp.com/cover_abc.jpg", ...}]
```

The field is `files` — not `file`, `image`, `upload`, `attachment`, or `media`. The `id` you get back is a **number** — that's the one you pass when attaching to a content-type relation. Do not pass `documentId` for media relations; it will silently create an empty relation.

## Quick reference card

```bash
# --- ENV ---
BASE="https://your-project.strapiapp.com"
TOKEN="..."                                  # API token, NOT a JWT
AUTH="Authorization: Bearer $TOKEN"

# --- DRAFTS ---
# Create draft
curl -X POST "$BASE/api/articles?status=draft" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"data":{"title":"Hello","content":"..."}}'

# Update draft (MUST include ?status=draft or it auto-publishes)
curl -X PUT "$BASE/api/articles/$DOCID?status=draft" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"data":{"title":"Edited"}}'

# Publish
curl -X PUT "$BASE/api/articles/$DOCID?status=published" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"data":{}}'

# Unpublish (no dedicated endpoint — delete the published version, draft remains)
curl -X DELETE "$BASE/api/articles/$DOCID?status=published" -H "$AUTH"

# --- READS ---
curl "$BASE/api/articles?status=draft"                       # list drafts
curl "$BASE/api/articles?status=published"                   # list published (default)
curl "$BASE/api/articles/$DOCID?status=draft&populate=cover" # one entry with media
curl "$BASE/api/articles?filters[slug][\$eq]=hello"          # filter

# --- UPLOADS ---
# Upload
curl -X POST "$BASE/api/upload" -H "$AUTH" -F "files=@cover.jpg"

# Attach to draft (two-step, preferred)
curl -X PUT "$BASE/api/articles/$DOCID?status=draft" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"data":{"cover": 42}}'    # numeric id from upload response

# Delete media
curl -X DELETE "$BASE/api/upload/files/42" -H "$AUTH"
```

## Part 1 — Drafts and published content

### Endpoint shape

| Action | Method | Path | Query | Body |
|---|---|---|---|---|
| Create draft | POST | `/api/{plural}` | `?status=draft` | `{"data": {...}}` |
| Create published | POST | `/api/{plural}` | *(omit)* or `?status=published` | `{"data": {...}}` |
| Update draft | PUT | `/api/{plural}/{documentId}` | `?status=draft` (**required**) | `{"data": {...}}` |
| Publish a draft | PUT | `/api/{plural}/{documentId}` | `?status=published` | `{"data": {}}` |
| Read draft | GET | `/api/{plural}/{documentId}` | `?status=draft` | — |
| Read published | GET | `/api/{plural}/{documentId}` | *(omit)* | — |
| Delete published version | DELETE | `/api/{plural}/{documentId}` | `?status=published` | — |
| Delete entirely | DELETE | `/api/{plural}/{documentId}` | *(omit)* | — |

`{plural}` is the pluralApiId from Content-Type Builder (e.g. `articles`, `blog-posts`). `{documentId}` is the v5 string id, not the numeric one.

### Full create-a-draft example

```bash
BASE="https://your-project.strapiapp.com"
TOKEN="..."

curl -X POST "$BASE/api/articles?status=draft" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "title": "How we ship on Fridays",
      "slug": "how-we-ship-on-fridays",
      "content": "...",
      "category": "engineering"
    }
  }'
```

Response (flat, no `attributes` wrapper):

```json
{
  "data": {
    "id": 137,
    "documentId": "kq9r8x2n4p7",
    "title": "How we ship on Fridays",
    "slug": "how-we-ship-on-fridays",
    "content": "...",
    "publishedAt": null,
    "createdAt": "2026-02-14T09:12:33.000Z",
    "updatedAt": "2026-02-14T09:12:33.000Z"
  },
  "meta": {}
}
```

`publishedAt: null` confirms it's a draft. Save the `documentId` — you'll use it on every subsequent call.

### Publishing the draft later

```bash
curl -X PUT "$BASE/api/articles/kq9r8x2n4p7?status=published" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data": {}}'
```

An empty `data` is fine — the state transition is what you want. You can also pass field updates in the same call.

### Unpublishing

There is no `POST /unpublish` endpoint in v5. To take a published post back to draft-only state:

```bash
curl -X DELETE "$BASE/api/articles/kq9r8x2n4p7?status=published" \
  -H "Authorization: Bearer $TOKEN"
```

This deletes the **published version** only. The draft remains intact and editable. Omit `?status=published` and you delete the document entirely — draft and all.

### v4 → v5 diff cheat sheet

| Thing | v4 | v5 |
|---|---|---|
| Unique identifier in paths | numeric `id` | string `documentId` |
| Response shape | `{data: {id, attributes: {...}}}` | `{data: {id, documentId, ...fields}}` |
| Relation in responses | `{data: {id, attributes}}` | plain object |
| Draft on create | `publishedAt: null` in body | `?status=draft` in URL |
| Publish | `POST /{plural}/{id}/actions/publish` | `PUT /{plural}/{docId}?status=published` |
| List drafts | `publicationState=preview` | `status=draft` |

## Part 2 — Uploading and attaching media

### Endpoint

`POST /api/upload`, `multipart/form-data`.

| Form field | Required | Type | Notes |
|---|---|---|---|
| `files` | yes | File (or array of files) | **Plural**, lowercase. Not `file`, `image`, `upload`. |
| `fileInfo` | no | **JSON string** | Per-file metadata (alt text, caption, name). Must be `JSON.stringify`d — not a nested object. |
| `ref` | no (only for 1-step attach) | string | e.g. `"api::article.article"` |
| `refId` | no (only for 1-step attach) | string/number | Numeric `id` of the target entry |
| `field` | no (only for 1-step attach) | string | Name of the media field on the entry |

**Do not set the `Content-Type` header yourself** when using undici / fetch / form-data. The library sets the correct `multipart/form-data; boundary=...` header. If you override it, the server can't parse the body and you get 415.

Response is a bare array:

```json
[
  {
    "id": 42,
    "documentId": "md_kq9r8x2n4p7",
    "name": "cover.jpg",
    "alternativeText": null,
    "caption": null,
    "width": 1600,
    "height": 900,
    "hash": "cover_abc123",
    "ext": ".jpg",
    "mime": "image/jpeg",
    "size": 182.4,
    "url": "https://your-project.media.strapiapp.com/cover_abc123.jpg",
    "provider": "strapi-provider-upload-aws-s3"
  }
]
```

Note the URL host is `<project>.media.strapiapp.com` — **different** from the API host.

### Two-step flow (preferred)

Upload first, then attach. You get cleaner error handling and the upload is reusable.

**curl**

```bash
# 1. Upload
UPLOAD=$(curl -s -X POST "$BASE/api/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "files=@./cover.jpg" \
  -F 'fileInfo={"alternativeText":"Cover image","name":"cover.jpg"}')

MEDIA_ID=$(echo "$UPLOAD" | jq '.[0].id')

# 2. Attach to existing draft (numeric id, not documentId)
curl -X PUT "$BASE/api/articles/$DOCID?status=draft" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"data\":{\"cover\": $MEDIA_ID}}"
```

**Node (undici)**

```js
import { fetch, FormData } from 'undici'
import { openAsBlob } from 'node:fs'

const BASE = 'https://your-project.strapiapp.com'
const TOKEN = process.env.STRAPI_TOKEN

// 1. Upload
const form = new FormData()
form.set('files', await openAsBlob('./cover.jpg'), 'cover.jpg')
form.set('fileInfo', JSON.stringify({ alternativeText: 'Cover image' }))

const upRes = await fetch(`${BASE}/api/upload`, {
  method: 'POST',
  headers: { Authorization: `Bearer ${TOKEN}` }, // no Content-Type
  body: form,
})
if (!upRes.ok) throw new Error(`upload failed: ${upRes.status} ${await upRes.text()}`)
const [media] = await upRes.json() // bare array
const mediaId = media.id            // numeric

// 2. Attach to draft
const attachRes = await fetch(
  `${BASE}/api/articles/${documentId}?status=draft`,
  {
    method: 'PUT',
    headers: {
      Authorization: `Bearer ${TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ data: { cover: mediaId } }),
  },
)
if (!attachRes.ok) throw new Error(`attach failed: ${attachRes.status} ${await attachRes.text()}`)
```

For a multi-image field (repeatable media), pass an array of numeric ids: `{ data: { gallery: [42, 43, 44] } }`.

### One-step flow (upload + attach in one call)

Works, but the `refId` must be the **numeric `id`** of the entry, not the `documentId`. That means you have to fetch the entry first to get its numeric id, which defeats most of the point.

```bash
curl -X POST "$BASE/api/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "files=@./cover.jpg" \
  -F "ref=api::article.article" \
  -F "refId=137" \
  -F "field=cover"
```

Prefer the two-step flow in 99% of cases.

### Common upload errors

| Symptom | Cause | Fix |
|---|---|---|
| 400 "files field is required" | Field named `file` / `image` / `upload` | Use `files` (plural) |
| 400 on `fileInfo` | Sent as a nested object, not a JSON string | `JSON.stringify(fileInfo)` |
| 415 Unsupported Media Type | JSON body, or manual `Content-Type` header | Use multipart, don't set the header |
| 403 on upload | API token missing `plugin::upload.content-api.upload` | Enable in Custom token permissions |
| 413 Payload Too Large | File > Cloud limit | Downscale (keep under ~25 MB) and retry |
| 500 with hash collision | Rare duplicate filename hash | Rename the file and retry |
| Upload succeeds, entry shows no image | You passed `documentId` to the media relation, or forgot `?status=draft` on the PUT | Use the numeric `id`, include `?status=draft` |

See [references/errors.md](references/errors.md) for the full error → cause → fix table.

## Part 3 — Strapi Cloud specifics

### Base URLs

- **API**: `https://<project-slug>.strapiapp.com/api`
- **Admin panel**: `https://<project-slug>.strapiapp.com/admin`
- **Media (CDN)**: `https://<project-slug>.media.strapiapp.com`

The media subdomain is set by Cloud — don't try to rewrite URLs to point at the API host.

### API tokens vs JWTs

**Agents always use API tokens**, never JWTs.

| | API token | JWT |
|---|---|---|
| Who | Server-to-server, agents, scripts | End users in a frontend |
| Header | `Authorization: Bearer <token>` | `Authorization: Bearer <jwt>` |
| Lifetime | Unlimited / 7d / 30d / 90d | Short-lived, per-user |
| Permissions | Read-only, Full access, or Custom role | The authenticated user's permissions |
| Where | Admin → Settings → API Tokens | `/api/auth/local` |

### Creating an API token for an agent

1. Admin panel → **Settings** → **Global Settings** → **API Tokens** → **Create new API Token**
2. Name: something like `agent-blog-writer`
3. Token duration: 30 days (or unlimited for long-running agents; rotate on a schedule)
4. Token type: **Custom** (recommended — least privilege)
5. Permissions to enable for a blog-writing agent:
   - `Article`: `find`, `findOne`, `create`, `update`, `delete`
   - `Upload`: `plugin::upload.content-api.upload`, `plugin::upload.content-api.find`, `plugin::upload.content-api.destroy`
6. Save and copy the token immediately — it's only shown once.

If uploads return 403, the Upload permissions weren't enabled on this token.

### CORS and rate limits

- CORS: configurable in Admin → Settings → Global Settings. For server-to-server (agents), CORS doesn't apply — it's browser-only.
- Rate limits: Strapi Cloud applies tier-dependent rate limits per project. Batch writes; don't hammer. On 429, back off.

## Part 4 — v5 response format

### Flattened shape

v4 wrapped every field in `attributes`. v5 does not.

```json
// v5
{
  "data": {
    "id": 137,
    "documentId": "kq9r8x2n4p7",
    "title": "How we ship on Fridays",
    "slug": "how-we-ship-on-fridays",
    "publishedAt": null,
    "cover": {
      "id": 42,
      "documentId": "md_kq9r8x2n4p7",
      "url": "https://your-project.media.strapiapp.com/cover_abc.jpg"
    }
  },
  "meta": {}
}
```

Lists use `data: [...]` plus `meta.pagination`.

### `id` vs `documentId`

| | `id` | `documentId` |
|---|---|---|
| Type | number | string |
| Scope | one document version (draft or published) | the document as a whole |
| Stable across publish? | no — draft and published have different `id`s | yes |
| Use in URL path params | no | **yes** |
| Use in media relation bodies | **yes** (the numeric upload id) | no |
| Use in content-type relation bodies | works, but prefer `documentId` | **yes** |

Rule of thumb: URL paths and content-type relations → `documentId`. Media relations → numeric `id` from the upload response.

### Error envelope

```json
{
  "data": null,
  "error": {
    "status": 400,
    "name": "ValidationError",
    "message": "2 errors occurred",
    "details": {
      "errors": [
        { "path": ["title"], "message": "title must be a `string` type", "name": "ValidationError" },
        { "path": ["slug"], "message": "slug must be unique",            "name": "ValidationError" }
      ]
    }
  }
}
```

When you hit a 400, always read `error.details.errors[*].path` and `message` — they tell you exactly which field broke.

## Part 5 — Decision flow for "create a blog post" requests

```
User says "create a blog post" / "add an article" / "post this to Strapi"
                             │
                             ▼
            Did they explicitly say "publish"?
                /                           \
              NO                            YES
               │                              │
               ▼                              ▼
    POST /api/articles?status=draft     Ask once: "Publish now, or
     body: {data: {...}}                  save as draft first?"
               │                              │
               ▼                              ▼
    Do they need a cover image?       If draft → same flow as left.
        /            \                 If publish now →
      NO            YES                  POST /api/articles
       │              │                   body: {data: {...}}
       │              ▼                   (no ?status, defaults to published)
       │    POST /api/upload (files=@)
       │              │
       │              ▼
       │    PUT /api/articles/{docId}?status=draft
       │     body: {data: {cover: <numeric id>}}
       │              │
       ▼              ▼
    Return the documentId + admin preview URL, tell them
    "Saved as draft. Reply 'publish' to push it live."
```

The default is always draft. Publishing is a separate, explicit step.

## Checklist before sending any Strapi write request

- [ ] Is this a write? If yes and the user did not explicitly say "publish", my URL has `?status=draft`.
- [ ] Am I PUT-ing an existing draft? If yes, the URL **still** has `?status=draft` (updates auto-publish without it).
- [ ] Is my body wrapped in `{"data": {...}}`?
- [ ] Path param is `documentId` (string), not numeric `id`?
- [ ] If uploading: form field is `files` (plural), body is multipart, no manual `Content-Type` header?
- [ ] If attaching media: am I passing the **numeric `id`** from the upload response, not the media's `documentId`?
- [ ] `Authorization: Bearer <API token>` header present? (Not a JWT.)
