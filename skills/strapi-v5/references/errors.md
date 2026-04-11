# Strapi v5 Cloud — Error Reference

Full error → cause → fix table. The main SKILL.md covers the most common ones; this file catches the long tail.

## HTTP status errors

| Status | Symptom / response | Cause | Fix |
|---|---|---|---|
| 400 | `ValidationError` on create/update | Body missing `data` wrapper, typo'd field name, or required field missing | Read `error.details.errors[*].path` — it tells you the exact field. Wrap body in `{"data": {...}}`. |
| 400 | `"files" is a required field` on upload | Form field named `file`, `image`, `upload`, `attachment`, or `media` | Rename form field to `files` (plural, lowercase) |
| 400 | `fileInfo` validation error | Sent as a nested object instead of a JSON string | `JSON.stringify(fileInfo)` before appending to the form |
| 400 | `"slug must be unique"` or similar uniqueness error | Collision with an existing entry | Generate a different slug, or update the existing entry instead |
| 400 | `Invalid key ...` | Field doesn't exist on the content type | Check the Content-Type Builder — field may have been renamed |
| 401 | `Missing or invalid credentials` | No `Authorization` header, or the token is expired/wrong | Add `Authorization: Bearer <API token>`. Check token in Admin → Settings → API Tokens. |
| 403 | `Forbidden` on a content-type endpoint | Token role doesn't grant `find` / `create` / `update` / `delete` on this content type | Edit the token's Custom role and enable the needed actions |
| 403 | `Forbidden` on `/api/upload` | Token missing `plugin::upload.content-api.upload` (and/or `.find`, `.destroy`) | Enable Upload plugin permissions on the token's Custom role |
| 404 | `Not Found` on `/api/{plural}` | Wrong `pluralApiId` | Check Content-Type Builder for the exact plural. `article` vs `articles`, `blog-post` vs `blog-posts`, etc. |
| 404 | `Not Found` on `/api/{plural}/{documentId}` | Wrong `documentId`, or the document exists only as draft and you're reading without `?status=draft` | Re-fetch to confirm the `documentId`. Add `?status=draft` for draft-only entries. |
| 405 | `Method Not Allowed` | Using a method the route doesn't support (e.g. PATCH instead of PUT) | Strapi REST uses GET, POST, PUT, DELETE — no PATCH |
| 413 | `Payload Too Large` on upload | File exceeds Cloud's upload size limit | Downscale / recompress. Keep images under ~25 MB for reliability. |
| 415 | `Unsupported Media Type` on upload | Body sent as JSON, or a manual `Content-Type` header overrode the multipart boundary | Use `multipart/form-data`. Do not set `Content-Type` manually when using `FormData` — let the library set it. |
| 429 | `Too Many Requests` | Hit Strapi Cloud's tier rate limit | Back off exponentially. Batch writes. |
| 500 | Generic `Internal Server Error` on upload | Occasionally a filename hash collision | Rename the file locally (append a timestamp) and retry |
| 502 / 503 / 504 | Gateway / unavailable | Transient Cloud outage or cold start | Retry with backoff; check Strapi Cloud status page if persistent |

## Behavioral gotchas (no error, wrong result)

These return 200 or 201 but produce a result that doesn't match what the agent intended. They are the reason this skill exists.

| Symptom | Cause | Fix |
|---|---|---|
| "Created a draft" but it's live on the site | Omitted `?status=draft` on the POST. `publishedAt: null` in the body is silently ignored in v5. | Re-do with `?status=draft`. To clean up: `DELETE /api/{plural}/{docId}?status=published` removes the published version and keeps the draft. |
| "Updated a draft" but now it's published | Omitted `?status=draft` on the PUT. Strapi v5 auto-publishes on updates unless the param is present (issue #24547). | Always pass `?status=draft` when editing a draft. |
| "Set `status: draft` in the body" but it got ignored | `status` is not a field on the entry — it's a URL query param | Move it to the URL: `?status=draft` |
| Upload succeeds, entry shows no image | Attached `documentId` instead of numeric `id` to a media relation | Use the numeric `id` from the upload response in the relation body |
| Upload succeeds, entry shows no image (variant 2) | Attach PUT was missing `?status=draft`, published over the draft with no media set | Re-attach with `?status=draft` and the numeric id |
| Relation appears empty in API response | Forgot `?populate=...` on the GET | Add `?populate=cover` or `?populate=*` |
| "Created" but nothing returned except status 204 | You called DELETE by accident, or hit a webhook endpoint | Re-check the method and path |
| Entry looks right in API, empty in admin | You wrote to the published version only; admin shows drafts by default | Write to draft with `?status=draft`, or toggle the admin view |

## Diagnostic recipe

When something looks off, run these in order:

```bash
BASE="https://your-project.strapiapp.com"
DOCID="kq9r8x2n4p7"
AUTH="Authorization: Bearer $TOKEN"

# 1. Does the draft exist?
curl -s "$BASE/api/articles/$DOCID?status=draft&populate=*" -H "$AUTH" | jq

# 2. Does the published version exist?
curl -s "$BASE/api/articles/$DOCID?populate=*" -H "$AUTH" | jq

# 3. List your token's permissions — Admin → Settings → API Tokens → click token

# 4. For a failing upload, verify the form shape:
curl -v -X POST "$BASE/api/upload" -H "$AUTH" -F "files=@./cover.jpg"
#   Look at the request line in -v output. It must be:
#     Content-Type: multipart/form-data; boundary=...
#   If you see application/json, your client is wrong.
```

## Upstream references

- Strapi v5 REST API: https://docs.strapi.io/cms/api/rest
- Draft & Publish in v5: https://docs.strapi.io/cms/features/draft-and-publish
- Upload plugin: https://docs.strapi.io/cms/plugins/upload
- API tokens: https://docs.strapi.io/cms/features/api-tokens
- Issue #24860 (`publishedAt: null` ignored): https://github.com/strapi/strapi/issues/24860
- Issue #24547 (PUT on draft auto-publishes): https://github.com/strapi/strapi/issues/24547
