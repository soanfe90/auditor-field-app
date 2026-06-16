# Auditor · Site Capture — PWA prototype

A self-contained, offline-first field app for building assessments, styled in the Auditor brand
(Blueprint Blue, Safety Orange, Canvas White). Pick a material, shoot photos against it, fill a
photos against it, fill a few field values, then export a zip your desktop app (Auditor /
Assets) can import and merge. No accounts, no server, no external libraries — it works on a
site with no signal.

## Files

- `index.html` — the entire app (UI + IndexedDB + camera + photo compression + GPS + zip export)
- `manifest.json` — PWA manifest (installable to the home screen)
- `sw.js` — service worker (offline app shell)
- `icon-192.png`, `icon-512.png`, `icon-512-maskable.png` — app icons
- `sample-fieldpack.json` — a sample pack you can import; also the data-contract reference

## Try it in 60 seconds

The app needs a **secure context** (https, or `localhost`) for the service worker and
home-screen install. Camera capture and storage work on localhost too.

Local test on your computer:

```bash
cd this-folder
python3 -m http.server 8000
# open http://localhost:8000
```

To use it on your **phone**, host the folder over https. Easiest options:
- Drag the folder onto https://app.netlify.com/drop (instant public URL), or
- Push to a GitHub repo and enable GitHub Pages, or
- Any static host (Cloudflare Pages, Vercel, S3 + CloudFront…).

Open the URL on your phone → browser menu → **Add to Home Screen**. Launch it from the icon
and it runs full-screen and offline.

In the app: tap **Try a sample building** (or **Import field pack** and choose
`sample-fieldpack.json`), open a material, add photos, fill fields, then **Export field data**.

## The data contract

This is the part that makes the desktop merge clean. Two JSON shapes travel between the apps.

### Field pack  (desktop → phone)

```json
{
  "format": "bca-fieldpack",
  "version": 1,
  "project": { "id": "sample-123-main", "name": "123 Main St", "details": { } },
  "capture_fields": [
    { "key": "condition", "label": "Condition", "type": "select", "options": ["Good","Fair","Poor","Failed"] },
    { "key": "rul", "label": "Est. remaining life (yrs)", "type": "number" },
    { "key": "notes", "label": "Field notes", "type": "textarea" }
  ],
  "materials": [
    { "id": "m-001", "name": "Modified bitumen — main roof", "category": "Roofing", "subcategory": "Low-slope membrane", "ref_code": "RF-01" }
  ]
}
```

Field types: `select` (renders as tap chips), `number`, `text`, `textarea`.

`materials[].id` is the **stable key** the round trip depends on. On the desktop side, generate
it from your real key — your Excel asset key if you have one, otherwise a composite like
`category|subcategory|name`, or the `df_assets` row index. Whatever you choose, the import must
match on the same id so photos never land on the wrong line item.

### Field data  (phone → desktop, inside the exported zip)

```
fielddata_<project>_<timestamp>.zip
├── data.json
└── photos/
    ├── m-001_1.jpg
    └── m-001_2.jpg
```

```json
{
  "format": "bca-fielddata",
  "version": 1,
  "project_id": "sample-123-main",
  "project_name": "123 Main St",
  "captured_at": "2026-06-16T17:40:00.000Z",
  "materials": [
    {
      "id": "m-001",
      "name": "Modified bitumen — main roof",
      "category": "Roofing",
      "ref_code": "RF-01",
      "fields": { "condition": "Fair", "rul": "8", "notes": "Ponding at NE corner." },
      "status": "in-progress",
      "photos": [
        { "file": "photos/m-001_1.jpg", "ts": "2026-06-16T17:38:11.000Z", "gps": { "lat": 43.2557, "lng": -79.8711, "acc": 6 } }
      ]
    }
  ]
}
```

`status` is one of `not-started`, `in-progress`, `completed`. Only materials that were touched
are included.

## How the desktop import would merge this

For each material in `data.json`, match by `id` to the asset, then:
1. Copy each `photos/*.jpg` into the project's photo folder, renamed per your Photo Rename Code
   convention, and append the new paths to `saved_data[name]['photos']`.
2. Write `fields` into `saved_data[name]['report_data']` (and/or `text_data`) by key.
3. Set `saved_data[name]['status']` — I'd map `completed` → assessed and leave the rest as a
   "field-captured, needs review" state so nothing auto-finalizes before the assessor looks.

## Notes / limits (it's a prototype)

- Photos are downscaled to ~2000 px, JPEG ~82%, on-device. Originals are not kept.
- All capture data lives in IndexedDB on the phone. Export regularly; iOS can evict storage
  under pressure. The header shows how many materials are waiting to export.
- GPS is best-effort per photo (8 s timeout); "no fix" is recorded if unavailable.
- Export uses the Web Share sheet where available (nice for AirDrop), otherwise a normal
  download.
- "⋯ → Reset this device" clears the pack and all captured data on the phone.
