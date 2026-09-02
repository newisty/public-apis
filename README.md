<!-- PUBLIC_API_DOCS_SYNC (devs/AI):
This file is the source of truth for the PUBLIC API (routes/api/public_api.php,
its controllers and services). If you change any request/response behavior in
those files (params, validation, response shape, envelope in
app/Traits/ApiResponse.php), you MUST update this file AND regenerate the HTML
docs (keeps README.html + README-fragment.html in sync):
    php dev-assets/public-apis-docs/build-readme-html.php
-->

# Public API Documentation

Public, unauthenticated API for lightweight file processing and lookups. No API key required.

## Base URL

```
https://newisty.com/api
```

All responses are JSON unless noted (binary endpoints return raw file bytes).

## Rate Limits

Per IP address, combined with a normalized User-Agent (clients rotating
User-Agents do not share the same quota bucket):

| Tier | Limit |
|------|-------|
| **Light task** | **10 requests / minute** |
| **Heavy task** | **5 requests / minute** |

Exceeding the limit returns `429 Too Many Requests` with `Retry-After` header.

## Security Headers

Every public API response includes:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: no-referrer
Cache-Control: no-store
```

## Response Format

Standard envelope (JSON endpoints):

```json
{
  "status": true,
  "message": "Success",
  "data": { },
  "meta": null
}
```

Errors:

```json
{
  "status": false,
  "message": "Error message",
  "data": null
}
```

`422` = validation/processing error, `404` = not found, `429` = rate limited, `503` = upstream failure.

---

## Table of Contents

1. [Image](#image)
2. [PDF](#pdf)
3. [QR Code](#qr-code)
4. [Avatar](#avatar)
5. [Text & Data](#text--data)
6. [Page Generators](#page-generators)
7. [Video Downloader](#video-downloader)
8. [Lookups (Light task, 10/min)](#lookups-light-task-10min)
9. [Lookups (Heavy task, 5/min)](#lookups-heavy-task-5min)
10. [Calculators](#calculators)
11. [AI Gateway](#ai-gateway)

---

## Image

### POST `/api/image/process`

Resize / compress / convert images. Max 10 MB per file.

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `images` | file[] | one of | Image files (jpg, jpeg, png, gif, webp, jiff, bmp) |
| `urls` | string | one of | Newline-separated image URLs |
| `width` | integer | yes | Target width (min 1) |
| `height` | integer | yes | Target height (min 1) |
| `resize-unit` | string | no | `px` or `percent` |
| `quality` | integer | no | 1–100 |
| `dpi` | integer | no | Target DPI |
| `convert-to` | string | no | Target format (`jpg`, `png`, `webp`, ...) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/image/process \
  -F "images[]=@photo.jpg" \
  -F "width=800" \
  -F "height=600" \
  -F "quality=85" \
  -F "convert-to=webp"
```

**Example response:**

```json
{
  "status": true,
  "message": "Image(s) processed successfully",
  "data": [
    {
      "success": true,
      "message": "Success",
      "filename": "ab12cd34.webp",
      "extension": "webp",
      "width": 800,
      "height": 600,
      "aspect_ratio": 1.3333333,
      "size_bytes": 48621,
      "size_human": "47.48 KB",
      "sha1": "d2a4f8...",
      "unit": "px",
      "base64": "data:image/webp;base64,UklGR..."
    }
  ],
  "meta": null
}
```

Note: `images` responds per-file; `urls` (newline-separated URL list) is also accepted.

---

### POST `/api/image/convert`

Convert an image to another format (jpg, jpeg, png, webp, gif, bmp). Returns binary file.

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `image` | file | yes | Source image (max 12 MB) |
| `format` | string | yes | Output format |
| `quality` | integer | no | 1–100 (default 90) |
| `filename` | string | no | Output filename (max 180) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/image/convert \
  -F "image=@photo.png" \
  -F "format=webp" \
  -F "quality=90" \
  -o converted.webp
```

**Example response (binary):**

```
HTTP/1.1 200 OK
Content-Type: image/webp
Content-Disposition: attachment; filename="x7f3ka9b.webp"

<binary image bytes>
```

---

### POST `/api/image/metadata-clean`

Strip EXIF/metadata from an image. Returns cleaned binary image.

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `image` | file | yes | jpg, jpeg, png, webp |
| `output_format` | string | no | `auto`, `image/jpeg`, `image/png`, `image/webp` (default `auto`) |
| `quality` | integer | no | 10–100 (default 92) |
| `prefer_smallest` | boolean | no | Prefer smallest output (default false) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/image/metadata-clean \
  -F "image=@photo.jpg" \
  -F "output_format=image/webp" \
  -o clean.webp
```

**Example response (binary):**

```
HTTP/1.1 200 OK
Content-Type: image/webp
Content-Disposition: attachment; filename="photo-clean.webp"
X-Clean-Extension: webp

<binary image bytes>
```

---

### POST `/api/image/enlarge`

Increase image file size (kb/mb) without visible quality loss. Max 20 MB input.

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `image` | file | one of | Uploaded image |
| `path` | string | one of | Image URL |
| `min` | number | no | Min target size (default 10) |
| `max` | number | no | Max target size (default 50) |
| `unit` | string | no | `kb` or `mb` (default `kb`) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/image/enlarge \
  -F "image=@small.jpg" \
  -F "min=10" \
  -F "max=50" \
  -F "unit=kb"
```

**Example response:**

```json
{
  "status": true,
  "message": "Image enlarged successfully",
  "data": {
    "url": "https://newisty.com/uploads/temp/ab12cd34.jpg",
    "size_bytes": 34800,
    "size_human": "33.98 KB",
    "width": 1024,
    "height": 768,
    "mime": "image/jpeg"
  },
  "meta": null
}
```

---

## PDF

### POST `/api/pdf/generate`

Generate a PDF from **text** or **images**.

Rate: **Light task**

**Mode A — text:** JSON / form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | string | yes | Text content |
| `filename` | string | no | Output name (max 180) |

**Example request (text):**

```bash
curl -X POST https://newisty.com/api/pdf/generate \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello World\nThis is a test PDF.", "filename": "hello"}'
```

**Example response (text, binary):**

```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="x7f3ka9b.pdf"

<binary pdf bytes>
```

**Mode B — images:** `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `images` | file[] | yes | jpg, jpeg, png, gif, webp, bmp (max 12 MB each) |
| `image_data` | string | yes | JSON array, e.g. `[{"file":"photo.jpg","scale":100,"dpi":150,"fit":"contain"}]` |

**Example request (images):**

```bash
curl -X POST https://newisty.com/api/pdf/generate \
  -F "images[]=@cover.jpg" \
  -F "images[]=@page2.jpg" \
  -F 'image_data=[{"file":"cover.jpg","scale":100,"dpi":150},{"file":"page2.jpg","scale":100,"dpi":150}]'
```

**Example response (images):**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "location": "uploads/pdf/ab12cd34.pdf",
    "filename": "x7f3ka9b.pdf"
  },
  "meta": null
}
```

---

### POST `/api/pdf/from-html`

Convert HTML / HTML file / URL to PDF.

Rate: **Light task** — JSON / form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `source_type` | string | yes | `html`, `html_file`, `url` |
| `html` | string | cond. | HTML string (if `html`) |
| `html_file` | file | cond. | HTML file (if `html_file`, max 5 MB) |
| `url` | string | cond. | URL (if `url`) |
| `title` | string | no | PDF title (max 500) |
| `filename` | string | no | Output name (max 180) |
| `response_type` | string | no | `binary` or `link` (default `link` for JSON) |
| `render_mode` | string | no | `readability` or `full_page` |
| `allow_private_hosts` | boolean | no | Allow private network hosts |
| `wait_dynamic_content` | boolean | no | Wait for JS render |
| `dynamic_wait_ms` | integer | no | 0–30000 |
| `layout` | string | no | `portrait`, `landscape` |
| `paper_size` | string | no | e.g. `a4`, `letter` |
| `margin_top/right/bottom/left` | number | no | 0–50 (mm) |
| `scale` | number | no | 0.1–2 |
| `print_background` | boolean | no | |
| `header_footer` | boolean | no | |

**Example request:**

```bash
curl -X POST https://newisty.com/api/pdf/from-html \
  -H "Content-Type: application/json" \
  -d '{
    "source_type": "html",
    "html": "<h1>Invoice #1024</h1><p>Paid in full.</p>",
    "title": "Invoice",
    "layout": "portrait",
    "paper_size": "a4"
  }'
```

**Example response (default `link`):**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "location": "https://newisty.com/temp/upload/ab12cd34-1786-7f3ka9b.pdf",
    "filename": "ab12cd34-1786-7f3ka9b.pdf"
  },
  "meta": null
}
```

With `response_type: "binary"` you get the PDF bytes directly instead. Note: `location` can be `null` (only `filename` set) in some render modes.

---

### POST `/api/pdf/text`

Extract text from a PDF.

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `pdf_file` | file | yes | PDF (mimes:pdf) |
| `page` / `page_number` | integer | no | Single page (min 1) |
| `page_numbers` / `pages` | string/list | no | Comma-separated pages or ranges |
| `page_limit` | integer | no | Max pages to process |
| `start_page` / `end_page` | integer | no | Range (min 1) |
| `retain_image_content` | boolean | no | Keep image-layer text |
| `ignore_encryption` | boolean | no | Try to read encrypted PDFs |
| `decode_memory_limit` | integer | no | Memory limit |

**Example request:**

```bash
curl -X POST https://newisty.com/api/pdf/text \
  -F "pdf_file=@document.pdf" \
  -F "pages=1-3"
```

**Example response:**

```json
{
  "success": true,
  "text": "Chapter 1\nIntroduction to APIs\n\nAn API is a set of rules...",
  "pages": [1, 2, 3]
}
```

Note: this endpoint returns **bare JSON without the `status/message/data` envelope**. `pages` contains the processed page numbers (empty if not applicable).

---

### POST `/api/pdf/merge`

Merge multiple PDFs (with per-file page selection).

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `pdf_files` | file[] | yes | PDFs (max 50 MB each), unique filenames |
| `pdf_data` | string | yes | JSON array, e.g. `[{"file":"a.pdf","pages":"all"},{"file":"b.pdf","pages":"1-3,5"}]` |
| `filename` | string | no | Output name (max 180) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/pdf/merge \
  -F "pdf_files[]=@report-a.pdf" \
  -F "pdf_files[]=@report-b.pdf" \
  -F 'pdf_data=[{"file":"report-a.pdf","pages":"all"},{"file":"report-b.pdf","pages":"1,3-5"}]'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "location": "uploads/pdf/ab12cd34.pdf",
    "filename": "x7f3ka9b.pdf"
  },
  "meta": null
}
```

---

### POST `/api/pdf/split`

Split a PDF into page groups.

Rate: **Light task** — `multipart/form-data`

**Mode A — one file:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `pdf_file` | file | yes | PDF (max 50 MB) |
| `splits` | string | yes | Page ranges, e.g. `1-3,4,5-7` (max 1200 chars) |
| `merge_into_one` | boolean | no | Merge outputs into one PDF |

**Mode B — multiple files:** `pdf_files[]` + `pdf_data` (same shape as merge).

**Example request (mode A):**

```bash
curl -X POST https://newisty.com/api/pdf/split \
  -F "pdf_file=@document.pdf" \
  -F "splits=1-2,3,4-6"
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "location": [
      "https://newisty.com/temp/upload/ab12cd34-0.pdf",
      "https://newisty.com/temp/upload/ab12cd34-1.pdf",
      "https://newisty.com/temp/upload/ab12cd34-2.pdf"
    ],
    "filename": ["x7f3ka9b-0.pdf", "x7f3ka9b-1.pdf", "x7f3ka9b-2.pdf"],
    "merge_into_one": false
  },
  "meta": null
}
```

---

## QR Code

### POST `/api/qr-code/generate`

Generate QR code. The content field is named after `qr_type`.

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `qr_type` | string | no | `url` (default), `text`, `wifi`, `email`, `phone`, `sms`, `vcard`, `location` |
| `url` / `text` / ... | string | yes | Content field matching `qr_type` |
| `qr_size` | integer | no | 50–2000 (default 300) |
| `qr_margin` | integer | no | 0–100 (default 10) |
| `qr_color` | string | no | Hex `#000000` |
| `qr_bg_color` | string | no | Hex `#ffffff` |
| `qr_ecc` | string | no | `L`, `M`, `Q`, `H` |
| `qr_format` | string | no | `png`, `svg`, `jpg`, `gif`, `webp`, `bmp`, `eps`, `html`, `xml`, `json`, `text` |
| `quality` | integer | no | 1–100 (default 85) |
| `draw_circular` | boolean | no | Rounded modules |
| `invert_matrix` | boolean | no | Invert colors |
| `image_transparent` | boolean | no | Transparent background |
| `has_label` | boolean | no | Show text label |
| `label_text` | string | no | Label text (max 200) |
| `label_font_size` | integer | no | 6–72 |
| `has_logo` | boolean | no | Embed logo |
| `logo_predefined` | string | no | Predefined logo key |
| `logo_file` | file | no | Logo image (max 2 MB) |
| `logo_size` | integer | no | 5–100 (default 20) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/qr-code/generate \
  -H "Content-Type: application/json" \
  -d '{
    "qr_type": "url",
    "url": "https://example.com",
    "qr_size": 300,
    "qr_format": "png",
    "qr_ecc": "M",
    "has_label": true,
    "label_text": "example.com"
  }'
```

**Example response:**

```json
{
  "status": true,
  "message": "QR Code generated successfully",
  "data": {
    "image_url": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUg...",
    "format": "png",
    "size": 300,
    "type": "url",
    "mime_type": "image/png",
    "previewable": true,
    "color": "#000000",
    "bg_color": "#ffffff"
  },
  "meta": null
}
```

---

### POST `/api/qr-code/decode`

Decode QR from image.

Rate: **Light task** — `multipart/form-data`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `image` | file | yes | png, jpg, jpeg, gif, webp, bmp (max 5 MB) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/qr-code/decode \
  -F "image=@qr.png"
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "decoded_text": "https://example.com"
  },
  "meta": null
}
```

---

## Avatar

### POST `/api/avatar/mesh-gradient`

Generate mesh-gradient avatar (SVG + base64).

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `seed` | string | no | Random seed (max 200); random if empty |
| `size` | integer | no | 64–1024 (default 512) |
| `shape` | string | no | `square`, `circle`, `squircle` |
| `base_color` | string | no | Hex color |
| `solid_color` | string | no | Hex color |
| `saturation` | number | no | 0–4 |
| `contrast` | number | no | 0–4 |
| `fade_distance` | integer | no | 40–200 |
| `text` | string | no | Initials (max 24) |
| `font_family` | string | no | Font name |
| `font_size_ratio` | number | no | 0.01–1 |
| `text_shadow` | number | no | 0–4 |
| `pixel_enabled` | boolean | no | Pixel pattern (default true) |
| `pixel_grid_size` | integer | no | 3–31 |
| `pixel_opacity` | number | no | 0–1 |
| `pixel_density` | number | no | 0–1 |
| `pixel_color_mode` | string | no | `gradient`, `monochrome`, `accent` |
| `pixel_shape` | string | no | `squares`, `circles`, `mix` |

**Example request:**

```bash
curl -X POST https://newisty.com/api/avatar/mesh-gradient \
  -H "Content-Type: application/json" \
  -d '{"seed": "my-avatar-1", "size": 256, "shape": "circle", "text": "JD"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Avatar generated successfully",
  "data": {
    "seed": "my-avatar-1",
    "svg": "<svg xmlns=\"http://www.w3.org/2000/svg\" width=\"256\" height=\"256\">...</svg>",
    "base64": "data:image/svg+xml;base64,PHN2ZyB4bWxucz0..."
  },
  "meta": null
}
```

---

## Text & Data

### POST `/api/data-converter/convert`

Convert between JSON / YAML / TOML.

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `content` | string | yes | Input data (max 200 KB) |
| `from` | string | yes | `auto`, `json`, `yaml`, `yml`, `toml` |
| `to` | string | yes | `json`, `yaml`, `yml`, `toml` |

**Example request:**

```bash
curl -X POST https://newisty.com/api/data-converter/convert \
  -H "Content-Type: application/json" \
  -d '{"content": "{\"name\": \"API\", \"version\": 1}", "from": "auto", "to": "yaml"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "location": "https://newisty.com/temp/upload/ab12cd34-1786-7f3ka9b.pdf",
    "filename": "ab12cd34-1786-7f3ka9b.pdf"
  },
  "meta": null
}
```

---

### POST `/api/text-diff/diff`

Unified diff between two texts.

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `old_text` | string | yes | Max 200 KB, ≤1500 combined lines |
| `new_text` | string | yes | Max 200 KB |

**Example request:**

```bash
curl -X POST https://newisty.com/api/text-diff/diff \
  -H "Content-Type: application/json" \
  -d '{"old_text": "hello world\nfoo", "new_text": "hello there\nfoo\nbar"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "lines": [
      { "type": "context", "prefix": " ", "old": 1, "new": 1, "text": "hello world" },
      { "type": "removed", "prefix": "-", "old": 1, "new": null, "text": "hello world" },
      { "type": "added", "prefix": "+", "old": null, "new": 2, "text": "hello there" },
      { "type": "context", "prefix": " ", "old": 2, "new": 3, "text": "foo" },
      { "type": "added", "prefix": "+", "old": null, "new": 4, "text": "bar" }
    ],
    "additions": 2,
    "deletions": 1
  },
  "meta": null
}
```

---

### POST `/api/markdown/to-html`

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `markdown` | string | yes | Max 200 KB |
| `sanitize` | boolean | no | Sanitize HTML (default true) |
| `method` | string | no | `commonmark`, `gfm` (default `gfm`) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/markdown/to-html \
  -H "Content-Type: application/json" \
  -d '{"markdown": "# Hello\n\nThis is **bold**.", "method": "gfm", "sanitize": true}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "html": "<h1>Hello</h1>\n<p>This is <strong>bold</strong>.</p>"
  },
  "meta": null
}
```

---

### POST `/api/markdown/to-text`

Strip markdown formatting → plain text. Response data key is `text`.

Rate: **Light task** — `markdown` (string, required, max 200 KB)

**Example request:**

```bash
curl -X POST https://newisty.com/api/markdown/to-text \
  -H "Content-Type: application/json" \
  -d '{"markdown": "# Hello\n\nThis is **bold**."}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "text": "Hello\n\nThis is bold."
  },
  "meta": null
}
```

---

### POST `/api/markdown/html-to-markdown`

Rate: **Light task** — `html` (string, required, max 200 KB)

**Example request:**

```bash
curl -X POST https://newisty.com/api/markdown/html-to-markdown \
  -H "Content-Type: application/json" \
  -d '{"html": "<h1>Hello</h1><p>This is <strong>bold</strong>.</p>"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "markdown": "# Hello\n\nThis is **bold**."
  },
  "meta": null
}
```

---

### POST `/api/markdown/stats`

Rate: **Light task** — `markdown` (string, required, max 200 KB)

**Example request:**

```bash
curl -X POST https://newisty.com/api/markdown/stats \
  -H "Content-Type: application/json" \
  -d '{"markdown": "# Hello\n\nThis is a [link](https://example.com) and ![img](a.png)."}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "words": 8,
    "characters": 44,
    "characters_without_spaces": 37,
    "lines": 3,
    "paragraphs": 2,
    "headings": 1,
    "links": 1,
    "images": 1,
    "code_blocks": 0,
    "estimated_reading_time": "1 min"
  },
  "meta": null
}
```

---

### POST `/api/developer/code-formatter`

Minify or beautify code.

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `language` | string | yes | `html`, `css`, `js`/`javascript`, `sql`, `xml`, `json`, `php`, `yaml`/`yml`, `ini`, `csv`, `toml`, `env`/`dotenv` |
| `code` | string | yes | Max 200 KB |
| `mode` | string | no | `minify` (default), `pretty`, `beautify` |

**Example request:**

```bash
curl -X POST https://newisty.com/api/developer/code-formatter \
  -H "Content-Type: application/json" \
  -d '{"language": "json", "mode": "pretty", "code": "{\"name\":\"API\",\"version\":1}"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Code processed successfully.",
  "data": {
    "mode": "pretty",
    "action": "pretty",
    "language": "json",
    "output": "{\n    \"name\": \"API\",\n    \"version\": 1\n}"
  },
  "meta": null
}
```

---

### POST `/api/string/bad-word-check`

Check text for bad words.

Rate: **Light task** — form fields

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `string` | string | yes | Text (max 20 KB) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/string/bad-word-check \
  -H "Content-Type: application/json" \
  -d '{"string": "This is a perfectly clean sentence."}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Success",
  "data": {
    "is_bad_word": false,
    "first_bad_word": null,
    "bad_word_count": 0,
    "char_count": 34,
    "word_count": 6,
    "line_count": 1,
    "masked_text": "This is a perfectly clean sentence.",
    "highlighted_html": "This is a perfectly clean sentence.",
    "message": "No bad words found."
  },
  "meta": null
}
```

---

## Page Generators

Rate: **Light task** for all.

### POST `/api/about-us-page-generator/generate`

| Param | Type | Required |
|-------|------|----------|
| `site_name` | string | yes (max 120) |
| `site_url` | string | yes (max 255) |
| `business_by` | string | yes (max 30) |
| `business_type` | string | yes (max 120) |
| `contact_info` | string | yes (max 255) |
| `template` | string | yes (theme key) |
| `found_year` | string | no (max 10) |

Available templates: `agency`, `content`, `corporate`, `creative`, `default`, `ecommerce`, `minimal`, `mission`, `nonprofit`, `product`, `service`, `startup`, `team`, `template2`, `template3`, `template4`, `template5`, `template6`

**Example request:**

```bash
curl -X POST https://newisty.com/api/about-us-page-generator/generate \
  -H "Content-Type: application/json" \
  -d '{
    "site_name": "Acme Inc",
    "site_url": "https://acme.com",
    "business_by": "John Doe",
    "business_type": "Software Company",
    "contact_info": "hello@acme.com",
    "template": "default",
    "found_year": "2015"
  }'
```

**Example response:**

```json
{
  "status": true,
  "message": "Generated",
  "data": {
    "html": "<section class=\"about-us\">\n  <h1>About Acme Inc</h1>\n  ...</section>"
  },
  "meta": null
}
```

---

### POST `/api/contact-us-page-generator/generate`

| Param | Type | Required |
|-------|------|----------|
| `template` | string | no (default `default`) |
| `website_name` | string | yes (max 120) |
| `website_email` | email | yes |
| `phone_enabled` | boolean | no |
| `website_phone` | string | cond. (max 40) |
| `address_enabled` | boolean | no |
| `website_address` | string | cond. (max 255) |
| `live_chat_enabled` | boolean | no |
| `social_enabled` | boolean | no |
| `facebook_enabled` / `facebook_url` | boolean/string | cond. |
| `twitter_enabled` / `twitter_url` | boolean/string | cond. |
| `instagram_enabled` / `instagram_url` | boolean/string | cond. |
| `whatsapp_enabled` / `whatsapp_number` | boolean/string | cond. |

Available templates (default `default`): `aurora`, `canvas`, `connectly`, `default`, `float`, `harmony`, `kiosk`, `lumina`, `nexora`, `nudge`, `orbit`, `prism`, `pulse`, `serene`, `velox`, `zephyr`

**Example request:**

```bash
curl -X POST https://newisty.com/api/contact-us-page-generator/generate \
  -H "Content-Type: application/json" \
  -d '{
    "website_name": "Acme Inc",
    "website_email": "hello@acme.com",
    "phone_enabled": true,
    "website_phone": "+8801700000000",
    "address_enabled": true,
    "website_address": "Dhaka, Bangladesh",
    "whatsapp_enabled": true,
    "whatsapp_number": "+8801700000000"
  }'
```

**Example response:**

```json
{
  "status": true,
  "message": "Generated",
  "data": {
    "html": "<section class=\"contact-us\">\n  <h2>Contact Acme Inc</h2>\n  ...</section>"
  },
  "meta": null
}
```

---

### POST `/api/privacy-policy-generator/generate`

| Param | Type | Required |
|-------|------|----------|
| `update_date` | string | yes (max 40) |
| `site_name` | string | yes (max 120) |
| `site_url` | string | yes (max 255) |
| `contact_info` | string | yes (max 255) |
| `toc_page` | string | no (max 255) |
| `has_cookies` | boolean | no |
| `has_ads` | boolean | no |
| `has_adsense` | boolean | no |
| `response_time_number` | integer | no (1–365, default 30) |
| `response_time_format` | string | no (max 20, default `day`) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/privacy-policy-generator/generate \
  -H "Content-Type: application/json" \
  -d '{
    "update_date": "2026-08-11",
    "site_name": "Acme Inc",
    "site_url": "https://acme.com",
    "contact_info": "hello@acme.com",
    "has_cookies": true,
    "has_adsense": true
  }'
```

**Example response:**

```json
{
  "status": true,
  "message": "Generated",
  "data": {
    "html": "<section class=\"privacy-policy\">\n  <h1>Privacy Policy</h1>\n  ...</section>"
  },
  "meta": null
}
```

---

### POST `/api/terms-and-condition-generator/generate`

| Param | Type | Required |
|-------|------|----------|
| `update_date` | string | yes (max 40) |
| `site_name` | string | yes (max 120) |
| `site_url` | string | yes (max 255) |
| `contact_info` | string | yes (max 255) |
| `privacy_policy_url` | string | no (max 255) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/terms-and-condition-generator/generate \
  -H "Content-Type: application/json" \
  -d '{
    "update_date": "2026-08-11",
    "site_name": "Acme Inc",
    "site_url": "https://acme.com",
    "contact_info": "hello@acme.com"
  }'
```

**Example response:**

```json
{
  "status": true,
  "message": "Generated",
  "data": {
    "html": "<section class=\"terms\">\n  <h1>Terms and Conditions</h1>\n  ...</section>"
  },
  "meta": null
}
```

---

## Video Downloader

Download videos from supported sites (YouTube, etc.) via the download service. Flow: `info` → `start` → poll `progress/{jobId}` → `download/{jobId}`.

Rate: **Heavy task** for `info`/`start` (external engine), **Light task** for `progress`/`download` (local reads).

### POST `/api/video-downloader/info`

Fetch video metadata (title, thumbnail, available formats) for a URL. Result is cached forever per URL.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | Video URL (public http/https, max 500 chars) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/video-downloader/info \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.youtube.com/watch?v=jNQXAC9IVRw"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Video information fetched.",
  "data": {
    "id": "jNQXAC9IVRw",
    "title": "Me at the zoo",
    "thumbnail": "https://i.ytimg.com/vi/jNQXAC9IVRw/hqdefault.jpg",
    "uploader": "jawed",
    "duration": 19,
    "duration_text": "0:19",
    "formats": []
  },
  "meta": null
}
```

Note: `formats` entries are flattened when the server does not expose the full format list; download quality is handled by `format` in `/start`. Info is cached forever per URL.

---

### POST `/api/video-downloader/start`

Queue a download for the given URL + quality. Starts a transient queue worker automatically; returns a `job_id` to poll. One active download per IP at a time (429 if another is still in progress).

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | Video URL (public http/https, max 500 chars) |
| `format` | string | yes | `best-mp4`, `1080p`, `720p`, `480p`, `audio-m4a`, `audio-mp3` (availability depends on server ffmpeg) |

**Example request:**

```bash
curl -X POST https://newisty.com/api/video-downloader/start \
  -H "Content-Type: application/json" \
  -d '{"url": "https://www.youtube.com/watch?v=jNQXAC9IVRw", "format": "best-mp4"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Download started.",
  "data": { "job_id": "34326ec8-c670-4408-a191-37e343a3609d" },
  "meta": { "tier": "best-mp4" }
}
```

Possible errors: `429` `"You already have a download in progress. Please wait for it to finish."` or `"The downloader is busy. Please try again in a moment."` — wait a few seconds and retry `start`.

---

### GET `/api/video-downloader/progress/{jobId}`

Poll download state. Poll every 2–5 seconds; the job stays in the queue for a few seconds to a minute before a worker picks it up.

**Example request:**

```bash
curl "https://newisty.com/api/video-downloader/progress/34326ec8-c670-4408-a191-37e343a3609d"
```

**Example response (queued — worker not started yet):**

```json
{
  "status": true,
  "message": "Download queued.",
  "data": {
    "status": "queued",
    "message": "Preparing download…"
  },
  "meta": null
}
```

**Example response (downloading):**

```json
{
  "status": true,
  "message": "Download status.",
  "data": {
    "status": "downloading",
    "message": "Downloading…",
    "percent": 42.5,
    "speed": "1.2MiB/s",
    "eta": "00:12"
  },
  "meta": null
}
```

Status values: `queued` → `starting` → `downloading` / `processing` / `merging` → `done` | `failed`. On `done` the payload also includes `filename` + `size`.

Retry behavior for API clients:
- `404` → job no longer exists (never dispatched, or already consumed and no state file was kept). The early polls right after `start` can return 404 if the worker has not picked the job up yet and the job is not visible in the queue — clients should tolerate a few 404s (poll at least 10 times / ~20–30 seconds) before giving up.
- `failed` status → the video/URL itself failed (private, removed, region-blocked, or temporary server issue). Retry the whole flow by calling `/start` again, ideally after a short delay.

---

### GET `/api/video-downloader/download/{jobId}`

Download the finished file (binary). Only available when progress reported `done`.

**Example request:**

```bash
curl "https://newisty.com/api/video-downloader/download/34326ec8-c670-4408-a191-37e343a3609d" \
  -o video.mp4
```

**Example response (binary):**

```
HTTP/1.1 200 OK
Content-Type: video/mp4
Content-Disposition: attachment; filename="Me at the zoo [jNQXAC9IVRw].mp4"

<binary video bytes>
```

Note: the download `job_id` becomes invalid after the job finishes or the file expires; a `404` means the file is gone.

---

### DELETE `/api/video-downloader/cancel/{jobId}`

Cancel a queued/in-flight download and delete its partial files. Idempotent: safe to call when the job already finished or expired.

**Example request:**

```bash
curl -X DELETE "https://newisty.com/api/video-downloader/cancel/34326ec8-c670-4408-a191-37e343a3609d"
```

**Example response:**

```json
{
  "status": true,
  "message": "Download job canceled.",
  "data": null,
  "meta": null
}
```

Retry behavior: after `404` on `/progress/{jobId}` for ~1 minute (job still waiting/unknown), cancel it and start a new download via `/start`. The cancel endpoint deletes both the queued job (queue + Redis/DB) and the partial `storage/app/ytdlp/{jobId}` folder.

---

## Lookups (Light task, 10/min)

Rate: **Light task**.

### GET `/api/ip`

Your current IP.

**Example request:**

```bash
curl https://newisty.com/api/ip
```

**Example response:**

```json
{
  "status": true,
  "message": "Current IP resolved",
  "data": {
    "ip": "103.55.22.11"
  },
  "meta": null
}
```

---

### GET `/api/domain/whois?domain=example.com`

WHOIS data.

**Example request:**

```bash
curl "https://newisty.com/api/domain/whois?domain=example.com"
```

**Example response:**

```json
{
  "status": true,
  "message": "WHOIS lookup completed.",
  "data": {
    "success": true,
    "type": "domain",
    "raw-data": "Connecting to COM.whois-servers.net...\r\nWHOIS Server: whois.markmonitor.com...",
    "whois-data": {},
    "sort-data": {
      "Domain Status": ["clientDeleteProhibited", "clientTransferProhibited"],
      "Name Server": ["NS1.GOOGLE.COM", "NS2.GOOGLE.COM"]
    },
    "sanitized": "example.com",
    "raw": "..."
  },
  "meta": null
}
```

Note: uses raw WHOIS protocol sockets — can take several seconds; some TLDs or timeouts return an error envelope instead.

---

### GET `/api/domain/age?domain=example.com`

Domain age.

**Example request:**

```bash
curl "https://newisty.com/api/domain/age?domain=example.com"
```

**Example response:**

```json
{
  "status": true,
  "message": "Domain age lookup completed.",
  "data": {
    "success": true,
    "type": "domain",
    "sanitized": "example.com",
    "sort-data": {
      "Domain Age": "30 years, 11 months, 28 days - (11311 days)",
      "Domain Age Years, Months, Days": "30 years, 11 months, 28 days",
      "Last Updated": "1 year, 0 months, 9 days - (368 days)",
      "Domain Will Expire": "1 year, 2 months, 1 days - (422 days)",
      "Name Server": "NS1.GOOGLE.COM, NS2.GOOGLE.COM",
      "Domain Status": "clientDeleteProhibited, clientTransferProhibited"
    }
  },
  "meta": null
}
```

---

### GET `/api/domain/expiry?domain=example.com`

Domain expiry date. **Note:** same payload as `/api/domain/age` (controller reuses the age computation); expiry data lives in `sort-data` → `Domain Will Expire`.

**Example request:**

```bash
curl "https://newisty.com/api/domain/expiry?domain=example.com"
```

**Example response:**

```json
{
  "status": true,
  "message": "Domain expiry lookup completed.",
  "data": {
    "success": true,
    "type": "domain",
    "sanitized": "example.com",
    "sort-data": {
      "Domain Will Expire": "1 year, 2 months, 1 days - (422 days)",
      "Domain Will Expire Formatted": "1 year, 2 months, 1 days, 15 hours, 13 minutes and 52 seconds"
    }
  },
  "meta": null
}
```

---

### GET `/api/country/list?search=&by=`

Country list. `by` options: `name` (default), `alpha`, `capital`, `region`.

**Example request:**

```bash
curl "https://newisty.com/api/country/list?search=ban&by=name"
```

**Example response:**

```json
{
  "status": true,
  "message": "Country list loaded",
  "data": {
    "countries": [
      {
        "name": { "common": "Bangladesh", "official": "People's Republic of Bangladesh" },
        "flags": { "svg": "https://.../bgd.svg", "alt": "flag of Bangladesh" },
        "cca3": "BGD",
        "region": "Asia"
      }
    ],
    "count": 1
  },
  "meta": null
}
```

---

### GET `/api/country/{by}/{value}`

Single country by field. **Valid `by` values: `name`, `alpha`, `capital`, `region`** (`name` matches the common name; for alpha codes pass the 3-letter code).

**Example request:**

```bash
curl "https://newisty.com/api/country/alpha/bgd"
curl "https://newisty.com/api/country/name/bangladesh"
curl "https://newisty.com/api/country/capital/dhaka"
```

**Example response:**

```json
{
  "status": true,
  "message": "Country found",
  "data": {
    "country": {
      "name": { "common": "Bangladesh", "official": "People's Republic of Bangladesh" },
      "cca3": "BGD",
      "capital": ["Dhaka"],
      "region": "Asia",
      "flag": { "url_svg": "https://.../bgd.svg", "description": "flag of Bangladesh" }
    }
  },
  "meta": null
}
```

---

### GET `/api/fake-credit-card/generate?issuer=&name=`

Fake (Luhn-valid) test card. `issuer`: `random` (default) or an issuer key. `name`: cardholder (max 80).

Issuers: `visa`, `mastercard`, `amex`, `discover`, `switch`, `jcb`, `unionpay`, `diners`, `diners_16`, `interpayment`, `instapayment`, `dankort`, `solo`, `laser`, `maestro`, `maestro_uk`, `mir`, `rupay`.

**Example request:**

```bash
curl "https://newisty.com/api/fake-credit-card/generate?issuer=visa&name=John+Doe"
```

**Example response:**

```json
{
  "status": true,
  "message": "Generated successfully.",
  "data": {
    "issuer": "visa",
    "issuer_name": "Visa",
    "visual_brand": "visa",
    "number_raw": "4111111111111111",
    "number_formatted": "4111 1111 1111 1111",
    "exp": "08/29",
    "exp_month": "08",
    "exp_year": "2029",
    "cvv": "321",
    "bank_name": "Chase",
    "card_product": "Signature"
  },
  "meta": null
}
```

---

### GET `/api/fake-credit-card/verify?number=&issuer=`

Luhn + issuer validation. `issuer`: `auto` (default) or specific.

**Example request:**

```bash
curl "https://newisty.com/api/fake-credit-card/verify?number=4111111111111111&issuer=auto"
```

**Example response:**

```json
{
  "status": true,
  "message": "Verification complete.",
  "data": {
    "valid": true,
    "luhn_valid": true,
    "issuer": "visa",
    "issuer_name": "Visa",
    "length": 16,
    "issuer_match": true,
    "message": "Card number passes all validation checks."
  },
  "meta": null
}
```

---

### GET `/api/dca-calculator/assets`

Available DCA investment assets. Asset ids are short codes (`btc`, `eth`, `sol`, ...) — use them as `asset_id` in `/calculate`.

**Example request:**

```bash
curl https://newisty.com/api/dca-calculator/assets
```

**Example response:**

```json
{
  "status": true,
  "message": "DCA calculator assets loaded",
  "data": {
    "assets": [
      { "id": "btc", "name": "Bitcoin", "symbol": "BTC", "type": "crypto" },
      { "id": "eth", "name": "Ethereum", "symbol": "ETH", "type": "crypto" }
    ]
  },
  "meta": null
}
```

---

## Lookups (Heavy task, 5/min)

### GET `/api/ip/lookup?ip=`

Geolocation for an IP (defaults to caller IP).

**Example request:**

```bash
curl "https://newisty.com/api/ip/lookup?ip=8.8.8.8"
```

**Example response:**

```json
{
  "status": true,
  "message": "IP lookup completed",
  "data": {
    "continent": "NA - North America",
    "country": "US - United States",
    "region": "Virginia",
    "city": "Ashburn",
    "district": "",
    "zip": "20149",
    "lat": 39.03,
    "lon": -77.5,
    "timezone": "America/New_York",
    "isp": "Google LLC",
    "organization": "Google Public DNS",
    "isMobileData": false,
    "isProxy": false,
    "isHosting": true,
    "isTor": null,
    "isVpn": null,
    "dnsGeo": "India - Google LLC",
    "dnsIp": "172.253.211.219"
  },
  "meta": null
}
```

---

### GET `/api/dns/lookup`

DNS + geo info for current connection.

**Example request:**

```bash
curl https://newisty.com/api/dns/lookup
```

**Example response:**

```json
{
  "status": true,
  "message": "DNS lookup completed",
  "data": {
    "dns": {
      "geo": "Bangladesh - GreenHost",
      "ip": "103.55.22.11"
    },
    "ipLookup": {
      "continent": "AS - Asia",
      "country": "BD - Bangladesh",
      "region": "Dhaka Division",
      "city": "Dhaka",
      "district": "",
      "zip": "",
      "lat": 23.79,
      "lon": 90.4,
      "timezone": "Asia/Dhaka",
      "isp": "GreenHost",
      "organization": "GreenHost",
      "isMobileData": false,
      "isProxy": false,
      "isHosting": true,
      "isTor": null,
      "isVpn": null,
      "dnsGeo": "Bangladesh - GreenHost",
      "dnsIp": "103.55.22.11"
    }
  },
  "meta": null
}
```

---

### GET `/api/subdomain-finder/search?apex=`

Enumerate subdomains of an apex domain (eTLD+1) from certificate transparency logs. Results are cached 24h per domain. On rate-limit the service falls back to proxy rotation (when enabled) and then to `crt.sh`.

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `apex` | string (required) | Registrable domain, e.g. `namecheap.com` |
| `dates` | bool (optional) | `1` to include first-seen dates |
| `limit` | int (optional) | Cap on returned subdomains (1–5000) |

**Example request:**

```bash
curl "https://newisty.com/api/subdomain-finder/search?apex=namecheap.com&dates=1"
```

**Example response:**

```json
{
  "status": true,
  "message": "Subdomains found",
  "data": {
    "success": true,
    "apex": "namecheap.com",
    "source": "crt.name",
    "quota_remaining": 998,
    "count": 2,
    "subdomains": [
      {
        "sub": "api.namecheap.com",
        "first_seen": "2024-04-21T04:56:29Z"
      },
      {
        "sub": "www.namecheap.com",
        "first_seen": "2026-01-05T00:00:00Z"
      }
    ],
    "message": "Subdomains found for namecheap.com"
  },
  "meta": null
}
```

> When every source is rate-limited the endpoint returns HTTP 429 with `"success": false` and an empty `subdomains` array.

---

### GET `/api/currency/list?search=`

Currency codes (cached 24h).

**Example request:**

```bash
curl "https://newisty.com/api/currency/list?search=USD"
```

**Example response:**

```json
{
  "status": true,
  "message": "Currency list loaded",
  "data": {
    "currencies": [
      { "code": "USD", "name": "US Dollar" }
    ],
    "count": 1
  },
  "meta": null
}
```

---

### GET `/api/currency/convert?amount=&base=&target=&date=`

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | number | yes | Amount |
| `base` | string | yes | Base code, e.g. `USD` |
| `target` | string | yes | Target code, e.g. `BDT` |
| `date` | string | no | `latest` (default) or `YYYY-MM-DD` |

**Example request:**

```bash
curl "https://newisty.com/api/currency/convert?amount=100&base=USD&target=BDT"
```

**Example response:**

```json
{
  "status": true,
  "message": "Conversion completed",
  "data": {
    "amount": 100,
    "base": "USD",
    "target": "BDT",
    "rate": 109.42,
    "converted_amount": 10942,
    "requested_date": "latest",
    "resolved_date": "2026-08-10",
    "is_latest": true,
    "used_latest_fallback": false
  },
  "meta": null
}
```

---

### GET `/api/currency/table?base=&date=&search=`

Rate table for one base currency.

**Example request:**

```bash
curl "https://newisty.com/api/currency/table?base=USD&search=eur"
```

**Example response:**

```json
{
  "status": true,
  "message": "Exchange rate table loaded",
  "data": {
    "base": "USD",
    "requested_date": "latest",
    "resolved_date": "2026-08-10",
    "is_latest": true,
    "used_latest_fallback": false,
    "rows": [
      { "code": "EUR", "name": "Euro", "rate": 0.9214, "display_rate": "0.9214" },
      { "code": "GBP", "name": "British Pound", "rate": 0.7841, "display_rate": "0.7841" }
    ]
  },
  "meta": null
}
```

---

### POST `/api/readability/extract`

Fetch URL and extract readable article.

| Param | Type | Required |
|-------|------|----------|
| `url` | string | yes |

**Example request:**

```bash
curl -X POST https://newisty.com/api/readability/extract \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/blog/post"}'
```

**Example response:**

```json
{
  "status": true,
  "message": "Readable article extracted successfully.",
  "data": {
    "article": {
      "url": "https://example.com/blog/post",
      "title": "Post Title",
      "author": "Author Name",
      "site_name": "Example",
      "excerpt": "Short summary of the article...",
      "image": "https://example.com/cover.jpg",
      "content_html": "<div><p>Clean readable content...</p></div>"
    },
    "pretty_html": "<!DOCTYPE html><html lang=\"en\"><head>...</head><body>...</body></html>"
  },
  "meta": null
}
```

Note: `article` can be `null` for pages where no article is detected (e.g. single-purpose landing pages).

---

## AI Gateway

Unified, authenticated proxy for `chat_completions`, `responses`, `messages`, `gemini` and raw `custom` upstreams. One client base URL, routed by exact `model` id with priority-ordered fallback. Upstream bodies are relayed verbatim with gateway metadata merged.

**Base URL:** `https://newisty.com/api/ai` — route names `api.ai-gateway.*` (use `route('api.ai-gateway.chat-completions')` etc., not hard-coded).

**Auth:** `Authorization: Bearer sk-gw-...` or `x-api-key: sk-gw-...` (Anthropic SDKs). `GET /models` also requires auth and is filtered by key `allowed_models`.

**Rate:** global `throttle:60,1` on `api/ai` + per-key `rate_limit_per_minute` (if set). Exceeding returns `429` with OpenAI error shape.

**Error envelope (always OpenAI-shaped, not `status/message/data`):**
```json
{ "error": { "message": "...", "type": "invalid_request_error", "code": "model_not_found" } }
```
`401 invalid_api_key`, `403 model_not_allowed`, `404 model_not_found`, `400 stream_not_supported|missing_model|unsupported_gemini_method`, `502 all_providers_failed`, `429 rate_limited`.

**Endpoints:**

| Method | Path | Inbound shape | Upstream mapping |
|---|---|---|---|
| GET | `/api/ai/models` | — | OpenAI list, filtered by `allowed_models` |
| POST | `/api/ai/chat/completions` | OpenAI Chat | translated per provider `api_compatible` |
| POST | `/api/ai/responses` | OpenAI Responses | `→ chat_completions/messages/gemini` |
| POST | `/api/ai/messages` | Anthropic Messages | `x-api-key` + `anthropic-version` upstream |
| POST | `/api/ai/v1beta/models/{model}:generateContent` | Google Gemini (`contents[]`, `systemInstruction`, `generationConfig`) | `?key=` or `x-goog-api-key` upstream |
| POST | `/api/ai/raw` | Raw passthrough (body relayed verbatim) | `custom` upstream at its `base_url`; model from `body.model` or `X-Gateway-Model` header; `400 missing_model` if absent; `stream:true` still `400` |

### GET `/api/ai/models`

List distinct routable model ids (active provider + active model, `allowed_models` filtered).

**Example:**
```bash
curl https://newisty.com/api/ai/models \
  -H "Authorization: Bearer sk-gw-..."
```
**Response:**
```json
{ "object": "list", "data": [{ "id": "gpt-4o", "object": "model", "owned_by": "mySiteName" }] }
```

### POST `/api/ai/chat/completions`

**Params:** `model` (string, required), `messages` (array), `system` (string), `max_tokens`, `temperature`, `top_p`, `stop`, `stream` (refused).

**Example:**
```bash
curl https://newisty.com/api/ai/chat/completions \
  -H "Authorization: Bearer sk-gw-..." -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"ping"}]}'
```

### POST `/api/ai/messages` (Anthropic)

```bash
curl https://newisty.com/api/ai/messages \
  -H "x-api-key: sk-gw-..." -H "Content-Type: application/json" \
  -d '{"model":"claude-3-5-sonnet-20241022","max_tokens":1024,"messages":[{"role":"user","content":"ping"}]}'
```

### POST `/api/ai/v1beta/models/{model}:generateContent` (Gemini)

Path captures `{model}` whole; SDK glues method onto model. `:generateContent` dispatches, `:streamGenerateContent` → `400 stream_not_supported`.

```bash
curl https://newisty.com/api/ai/v1beta/models/gemini-1.5-flash:generateContent \
  -H "Authorization: Bearer sk-gw-..." -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"ping"}]}]}'
```

### POST `/api/ai/raw`

```bash
curl https://newisty.com/api/ai/raw \
  -H "Authorization: Bearer sk-gw-..." -H "X-Gateway-Model: gpt-4o" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"ping"}]}'
```

**Success response:** upstream JSON object verbatim + injected `provider`/`gateway` (unless `inject_metadata=false` per key or `setting('ai_gateway_inject_metadata')` off). Arrays/scalars/non-JSON pass through untouched; existing upstream keys never overwritten. Headers always include `X-Gateway-*`:

```jsonc
{
  // ...upstream...
  "provider": "Newisty", // setting('ai_gateway_provider_label')
  "gateway": {
    "upstream_shape": "chat_completions",
    "inbound_shape": "chat_completions",
    "model_requested": "gpt-4o",
    "model_used": "gpt-4o",
    "attempts": 1,
    "latency_ms": 812
  }
}
```
Cross-shape fallback returns upstream's shape; `gateway.upstream_shape` tells which. Constrain `allowed_models` to same-shape if client cannot handle.

---

## Calculators

### POST `/api/dca-calculator/calculate`

Dollar-cost-average backtest (fetches historical prices).

Rate: **Heavy task**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `asset_id` | string | yes | Asset id from `/assets` |
| `amount` | number | yes | 1–1000000 |
| `frequency` | string | yes | `daily`, `weekly`, `monthly` |
| `start_date` | string | yes | `Y-m-d`, ≥ 2017-01-01, ≤ today |
| `end_date` | string | no | `Y-m-d`, ≥ start, ≤ today |
| `currency` | string | no | 3-letter code |

**Example request:**

```bash
curl -X POST https://newisty.com/api/dca-calculator/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "asset_id": "btc",
    "amount": 100,
    "frequency": "monthly",
    "start_date": "2023-01-01",
    "currency": "USD"
  }'
```

**Example response:**

```json
{
  "status": true,
  "message": "DCA result ready",
  "data": {
    "asset": { "id": "btc", "name": "Bitcoin", "symbol": "BTC", "type": "crypto" },
    "currency": "USD",
    "rate": 1,
    "frequency": "monthly",
    "start_date": "2023-01-01",
    "end_date": "2026-08-01",
    "history": {
      "from": "2017-01-01",
      "to": "2026-08-10",
      "latest_price": 61750.5,
      "stale": false
    },
    "summary": {
      "purchases": 44,
      "total_invested": 4400,
      "total_units": 0.0921,
      "current_value": 5685.12,
      "profit": 1285.12,
      "roi": 29.21,
      "avg_buy_price": 47774.16,
      "latest_price": 61750.5
    },
    "compare": {
      "dca_value": 5685.12,
      "lump_value": 6042.33,
      "lump_profit": 1642.33,
      "difference": 357.21,
      "better": "lump"
    },
    "best_month": { "date": "2023-11-01", "change": 21.4 },
    "worst_month": { "date": "2022-06-01", "change": -18.2 },
    "series": [
      { "date": "2023-01-01", "invested": 100, "value": 105.2 }
    ]
  },
  "meta": null
}
```

---

### GET `/api/crypto-gas-fee-calculator/market-data?currency=USD`

Live gas fees + market data.

Rate: **Heavy task** — `currency` must be 3 uppercase letters (default `USD`).

**Example request:**

```bash
curl "https://newisty.com/api/crypto-gas-fee-calculator/market-data?currency=USD"
```

**Example response:**

```json
{
  "status": true,
  "message": "Crypto gas market data loaded",
  "data": {
    "currency": "USD",
    "fiat_rate": 1,
    "updated_at": "2026-08-11 10:00:00",
    "live": true,
    "partial": false,
    "stale": false,
    "live_count": 10,
    "total_count": 10,
    "chains": [
      {
        "id": "ethereum",
        "name": "Ethereum",
        "layer": "L1",
        "website": "https://ethereum.org",
        "token": "ETH",
        "token_price_usd": 3250.8,
        "gas": { "standard": 8.5, "fast": 10.2, "instant": 12.1 },
        "live": true,
        "updated_at": "2026-08-11 10:00:00"
      },
      {
        "id": "bsc",
        "name": "BNB Smart Chain",
        "layer": "L1",
        "website": "https://www.bnbchain.org",
        "token": "BNB",
        "token_price_usd": 585.4,
        "gas": { "standard": 3.2, "fast": 4.0, "instant": 5.1 },
        "live": true,
        "updated_at": "2026-08-11 10:00:00"
      }
    ]
  },
  "meta": null
}
```
