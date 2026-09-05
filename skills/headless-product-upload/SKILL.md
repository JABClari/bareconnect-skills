---
name: headless-product-upload
description: >-
  Create products in a Bareconnect store and upload their images through the
  Bareconnect Headless Partner API (secret key, server-side). Covers the main
  image, the cover/rollover image, and the extras gallery, the plan cap on
  extras, verifying and deleting images. SERVER-SIDE ONLY — the secret key can
  read and change everything; never put it in a browser or a prompt. Use when a
  partner integration needs to push a catalog (with photos) into Bareconnect
  stores it manages.
version: 1.0.0
---

# Bareconnect — Headless product upload with images

You are pushing products **and their photos** into a Bareconnect store from a server you control,
using the **Headless Partner API** and a **secret** key. This is the admin-grade surface — it can
create and change anything in the stores linked to your partner account.

> **This is not the storefront skill.** A browser frontend uses a *publishable* key and
> `ai-engine-connect`. The `bc_mgmt_…` key here must live only on your server (env/secrets), never
> in source, a commit, a browser, or a prompt. A leak is a full compromise of every linked store.

## When to use
- A partner/server integration needs to create products in Bareconnect stores it manages and attach
  images to them (a catalog sync, a bulk import, a PIM → Bareconnect pipeline).

## When NOT to use
| If you want… | Use instead |
|---|---|
| a browser storefront that reads catalog + cart | `ai-engine-connect` (publishable key) |
| to make the store shoppable by AI agents | UCP (`ucp-connect`) |
| a merchant to manage products by hand | the Bareconnect admin |

## Credentials & base
- **Base URL:** `https://bareconnect.com/api/partner/v1`
- **Auth:** `Authorization: Bearer bc_mgmt_…` on every request.
- **Scope required for everything here:** `catalog:write` (creating products and uploading/deleting
  images). Listing/reading products is `catalog:read`.
- Read the key from an environment variable; never hardcode it.

## The image model — read this before uploading

A product has **three image slots**, addressed by a `type` field:

| `type` | Slot | Behaviour |
|---|---|---|
| `product_main_images` | The main product image | **Replaces** — uploading again swaps the single image (the old file is deleted). |
| `product_cover_images` | The cover / rollover image | **Replaces** — same single-image behaviour. |
| `product_images` | The extras gallery | **Appends** — each upload adds one more. **Capped by the store's plan.** |

Extras cap by plan: **Basic 10, Professional 15, Enterprise unlimited**. Exceeding it returns
`422 image_limit_reached` — remove one or upgrade; do not retry blindly.

Uploads are **`multipart/form-data`** with two fields: `image` (the file) and `type` (one of the
three above). Allowed files: jpg, jpeg, png, gif, webp, **max 10 MB**. Responsive variants are
generated for you (except GIF). The response returns the new image's `id` (a UUID) and `url`.

## The flow

1. **Create the product** — `POST /stores/{storeId}/products`. Keep the returned `id`.
2. **Upload the main image** — `POST /stores/{storeId}/products/{productId}/images` with
   `type=product_main_images`.
3. **Upload the cover** (optional) — same endpoint, `type=product_cover_images`.
4. **Upload each extra** — same endpoint, `type=product_images`, once per photo. Stop if you get
   `422 image_limit_reached`.
5. **Verify** — `GET /stores/{storeId}/products/{productId}` returns an `images` block; use it to
   make a re-run idempotent (skip products that already have images).
6. **Delete** if needed — `DELETE /stores/{storeId}/products/{productId}/images/{mediaId}` where
   `mediaId` is the `id` you got back from the upload.

### curl

```bash
BASE=https://bareconnect.com/api/partner/v1
AUTH="Authorization: Bearer $BC_MGMT_KEY"

# 1) create the product
PRODUCT_ID=$(curl -s -X POST "$BASE/stores/$STORE_ID/products" \
  -H "$AUTH" -H 'Content-Type: application/json' \
  -d '{"name":"Aster Bralette","price":120,"quantity":25}' \
  | jq -r '.data.id')

# 2) main image (replaces)  — note: multipart, NO Content-Type header (curl sets the boundary)
curl -s -X POST "$BASE/stores/$STORE_ID/products/$PRODUCT_ID/images" \
  -H "$AUTH" \
  -F "type=product_main_images" \
  -F "image=@./aster-main.jpg"

# 3) cover / rollover (replaces)
curl -s -X POST "$BASE/stores/$STORE_ID/products/$PRODUCT_ID/images" \
  -H "$AUTH" -F "type=product_cover_images" -F "image=@./aster-cover.jpg"

# 4) an extra gallery image (appends; may 422 at the plan cap)
curl -s -X POST "$BASE/stores/$STORE_ID/products/$PRODUCT_ID/images" \
  -H "$AUTH" -F "type=product_images" -F "image=@./aster-back.jpg"
```

Successful upload returns:

```json
{ "data": { "id": "9a1f2c…", "url": "https://…/aster-main.jpg", "type": "product_main_images" } }
```

### Node (server-side)

```js
import fs from "node:fs";

const BASE = "https://bareconnect.com/api/partner/v1";
const auth = { Authorization: `Bearer ${process.env.BC_MGMT_KEY}` };

async function uploadImage(storeId, productId, filePath, type) {
  const form = new FormData();
  form.set("type", type);
  form.set("image", new Blob([fs.readFileSync(filePath)]), filePath.split("/").pop());

  const res = await fetch(`${BASE}/stores/${storeId}/products/${productId}/images`, {
    method: "POST",
    headers: auth,            // do NOT set Content-Type — fetch adds the multipart boundary
    body: form,
  });

  if (res.status === 422) {
    const err = await res.json();
    if (err.error === "image_limit_reached") {
      console.warn(`Gallery full (limit ${err.limit}) — skipping extra`);
      return null;            // stop uploading extras; don't retry
    }
  }
  if (!res.ok) throw new Error(`upload failed ${res.status} ${await res.text()}`);
  return (await res.json()).data;   // { id, url, type }
}

// create + main + cover + extras
const product = await fetch(`${BASE}/stores/${storeId}/products`, {
  method: "POST",
  headers: { ...auth, "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Aster Bralette", price: 120, quantity: 25 }),
}).then(r => r.json()).then(r => r.data);

await uploadImage(storeId, product.id, "./aster-main.jpg", "product_main_images");
await uploadImage(storeId, product.id, "./aster-cover.jpg", "product_cover_images");
for (const extra of ["./aster-back.jpg", "./aster-detail.jpg"]) {
  const r = await uploadImage(storeId, product.id, extra, "product_images");
  if (r === null) break;      // hit the plan cap
}
```

## Rules that save you a bad run
- **Never set `Content-Type` yourself on the upload** — let curl/fetch set the multipart boundary,
  or the file won't parse.
- **Main and cover replace; the gallery appends.** Re-uploading a main image is a swap, not a
  duplicate. Re-uploading a gallery image is a *new* image — dedupe on your side (or check the
  product's `images` block first) so a retried sync doesn't fill the gallery with copies.
- **Respect `422 image_limit_reached`** — it's the plan cap on extras, not a transient error. Stop,
  don't retry; surface "remove one or upgrade the plan."
- **`mediaId` for delete is the `id`** returned by the upload (a UUID), not the URL.
- **404 means not-yours** — the store/product isn't linked to your partner account. Identical to
  "doesn't exist" on purpose (no id enumeration).
- **One image per request.** To upload N photos, make N calls.

See the full endpoint reference and response shapes in [`../../headless/README.md`](../../headless/README.md)
(Products → *Upload a product image*) and the machine spec in
[`../../openapi/headless.v1.yaml`](../../openapi/headless.v1.yaml).
