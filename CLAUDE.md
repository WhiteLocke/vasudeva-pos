# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Vasudeva Shop** is a single-file, browser-based Point of Sale (POS) and Inventory Management System for retail operations. The entire application lives in `vasudeva-pos.html` — a monolithic file with ~2,691 lines of inline HTML, CSS, and JavaScript. No build system, no package manager, no external dependencies.

## Running the App

- Open `vasudeva-pos.html` directly in any modern browser (Chrome 83+ recommended)
- Or serve it via any static file server
- Default PIN: `1234`

**Browser requirements:**
- MediaDevices API (getUserMedia) for camera barcode scanning
- BarcodeDetector API (optional, falls back to manual entry)
- localStorage for custom color persistence

## Architecture

### File Structure

Everything is in `vasudeva-pos.html`:
1. `<head>` — Meta tags, Google Fonts, ~800 lines of inline CSS with CSS custom properties
2. `<body>` — Five main screens + modals + toast notification element
3. `<script>` — All application logic (~1,300 lines)

### Screen State Machine

```
PIN Entry (screen-pin) → Entry Menu (screen-entry) → Sell (screen-sell)
                                                    → Add Stock (screen-addstock)
                                                    → Inventory (screen-inventory)
                                                         └→ Barcode Mapping (screen-barcodescan)
```

Navigation is handled by `goTo(screenId)` which applies CSS transforms. `goBack()` returns to the entry menu. `exitBarcodeMapping()` returns to inventory.

### Configuration (line ~845)

```javascript
const CONFIG = {
    N8N_LOOKUP:    'https://n8n.whitelockemedia.com/webhook/vasudeva-lookup',
    N8N_ALL:       'https://n8n.whitelockemedia.com/webhook/vasudeva-all',
    N8N_SELL:      'https://n8n.whitelockemedia.com/webhook/vasudeva-sell',
    N8N_ADD_STOCK: 'https://n8n.whitelockemedia.com/webhook/vasudeva-addstock',
    OWNER_PIN:     '1234',
    CURRENCY:      '₹'
};
```

### Data Flow

All persistence goes through four N8N webhook URLs — there is no local database. Product data is fetched into `productCache` (an array) on PIN entry and reused for local search. Sales and stock updates are POSTed to separate webhooks.

```
productCache (in-memory)  ←  GET  ← N8N_ALL (full catalog)
                          ←  GET  ← N8N_LOOKUP (single product by barcode)

sale/stock data           →  POST → N8N_SELL / N8N_ADD_STOCK
barcode mapping           →  POST → N8N_ADD_STOCK (background, per-product)
```

### Global State Variables (line ~892)

```javascript
let currentScreen, currentProduct, currentQty, currentPrice;
let currentSize, currentColor, sellSizeData, currentSellSizeTab;
let sellSizeRows, sellSizeAvail, sellColorRows, sellColorAvail;
let basket = [], productCache = null, cacheLoading = false;
let stockProductFound = false, currentSuggestion = null;
let bmapQueue, bmapIndex, bmapExtraOpen, bmapSavedCount, bmapSavedIndices, bmapData;
let lastFailedBarcode, sellNameResults, sellNameTimer;
let invSortLowFirst, invExpandedSku, invProductMap;
let stockMultiSuggestions, stockNameResultsOpen, stockMultiSourceField;
```

### Key Functional Modules

| Area | Key Functions |
|------|---------------|
| Navigation | `goTo()`, `goBack()`, `switchTab()` |
| Barcode | `handleBarcodeScan()`, `searchCache()`, `lookupProduct()`, `sanitizeBarcode()` |
| Sell flow | `displayProduct()`, `renderSizePicker()`, `renderColorPicker()`, `addToBasket()`, `finalizeOrder()` |
| Sell size picker | `buildSellSizeChipPicker()`, `switchSellSizeTab()`, `renderSellSizeGrid()`, `addSellSizeRow()`, `confirmSellSizeRow()` |
| Sell color picker | `renderColorRowsUI()`, `renderColorChipGrid()`, `addColorRow()`, `confirmColorRow()` |
| Sell name search | `toggleSellNameSearch()`, `onSellNameInput()`, `showSellNameResults()`, `pickSellNameResult()` |
| Stock entry | `submitStock()`, `showCurrentStockSizes()`, `getSizeBreakdown()`, `getColorBreakdown()` |
| Stock current sizes | `showCurrentStockSizes()`, `toggleCurrentStockEdit()`, `saveCurrentStockSizes()` |
| Stock current colors | `showCurrentColorStock()`, `toggleCurrentColorEdit()`, `saveCurrentColorStock()` |
| Stock color palette | `renderCqpChips()`, `applyCqpColor()`, `saveCqpColor()` (CQP = Color Quick Palette) |
| Inventory | `renderInventory()`, `editProductFromInventory()`, `toggleInvItem()`, `toggleInvSort()` |
| Barcode mapping | `startBarcodeMapping()`, `loadBmapProduct()`, `bmapNext()`, `bmapPrev()`, `bmapSaveInBackground()`, `exitBarcodeMapping()` |
| Autocomplete | `showGhosts()`, `acceptSuggestion()`, `lookupBy()` |
| UI utilities | `openNumpad()`, `openDiscount()`, `openCamera()`, `showToast()` |
| Data helpers | `computeHealedStock()`, `parseColors()`, `parseSizes()` |
| Colors | `loadGlobalColors()`, `saveCustomColors()`, `getColorHex()` |

### Data Models

**Product object** (from N8N response):
```javascript
{
    sku, barcode, name, category,
    color,    // string, JSON "{color:qty,...}", or null
    mrp, cost, stock, minStock,
    expiry, supplier,
    sizes     // JSON "{size:qty,...}" or null
}
```

**Basket item:**
```javascript
{ _key, name, sku, barcode, size, color, price, qty, total }
```

### Size Categories

Hardcoded size systems: Letter (XS–XXXL), Numeric (26–48), Bra (28A–40D), plus Free Size and Other.

### Design System

CSS custom properties (defined at top of `<style>`):
- `--bg: #FBF4EB` (cream), `--primary: #C4601D` (rust), `--secondary: #1B3A4B` (navy)
- `--accent: #D4930A` (gold), `--success: #2D6A4F`, `--danger: #B5311B`
- Fonts: Cormorant Garamond (headers), DM Sans (body), Noto Sans Devanagari (Hindi)

### Bilingual UI

All major UI elements have English + Hindi labels. Hindi text uses Devanagari script rendered via Noto Sans Devanagari.

## Development Notes

- Since the app is a single HTML file, all edits are in `vasudeva-pos.html`
- The product cache is invalidated (set to `null`) after each sell or stock submission to force a fresh fetch
- Search uses 500ms debounce timers to reduce API calls
- Custom colors are persisted in localStorage under `COLOR_STORAGE_KEY` (`'vasudeva-global-colors'`)
- The discount flow requires the owner PIN (`OWNER_PIN` in CONFIG) to unlock price override
- Stock status thresholds: green = in stock, amber = stock ≤ `minStock` (default 10), red = out of stock
- `computeHealedStock(p)` returns `max(sizeTotal, sheetStock)` — use this when displaying stock, not raw `p.stock`
- `parseColors(val)` handles both old `"Red, Black"` strings and new `{"Red":5}` JSON; returns `{name: qty|null}`
- Inventory has two sort modes toggled by `toggleInvSort()`: A–Z (default) and low-stock-first

## Multi-Match Search Pattern (Add Stock)

SKU and Name fields support multi-match autocomplete via `searchCacheMulti(value)`:
- `stockMultiSuggestions[]` holds the current match array — **must not be cleared between setting and tapping the pill**
- `clearStockNameResults()` clears `stockMultiSuggestions` — never call it immediately after setting the array; snapshot first
- Results containers (`#stock-sku-results`, `#stock-name-results`) are **block siblings inside `.form-section`**, NOT inside the `.form-field` flex containers
- They require `position: relative` and `background: var(--bg)` — without `position: relative` they render behind `position: relative` form-fields; without a contrasting background they're invisible against `var(--bg-card)`
- The pill ("+ N more →") is shown via `#hint-f-sku-more` / `#hint-f-name-more`; clicking calls `toggleStockNameResults(field)`

## Barcode Mapping Mode

Launched from Inventory via the "⬛ Add barcodes" button (`startBarcodeMapping()`). Filters `productCache` to products missing barcodes, queues them in `bmapQueue[]`.

- `loadBmapProduct(idx)` displays current item: SKU large, name, optional extra fields
- `bmapNext()` / `bmapPrev()` navigate the queue; `bmapNext()` saves via `bmapSaveInBackground()` if barcode or name entered
- Saves fire POST to `N8N_ADD_STOCK` in background; progress shown as "X saved / N total"
- `saveBmapToCache()` patches `productCache` in-memory so inventory reflects changes without re-fetch
- `toggleBmapExtra()` shows/hides optional fields: category, color, sizes, qty, MRP, cost, min stock, expiry, supplier
- `exitBarcodeMapping()` returns to inventory and re-renders it

## Known Issues (do not break workarounds)

1. **Color quantities not saving to sheet** — `incoming.colors` may not be reaching vasudeva-addstock Code JS1 correctly. Needs investigation.
2. **Sell by color unconfirmed** — color deduction code exists in vasudeva-sell JS1, guarded by `colorsRaw.trim().startsWith('{')`. Won't run until color quantities are in JSON format in sheet.
3. **Size/quantity drift** — when `sizeTotal > stock`, frontend trusts sizes as ground truth. Sheet Quantity cell not auto-corrected. Manual fix needed until reconciliation workflow is built.

## Invoice Review Screen (`screen-invoicereview`)

Launched from Inventory via "⚠ Review (n)" button (only visible when `pendingReviews.length > 0`). Card-based, one item at a time — same UX pattern as barcode mapping.

- `loadPendingReviews()` — fetches from `N8N_INVOICE_REVIEW` webhook; called on PIN entry and on `goTo('inventory')`
- `updateReviewBadge()` — shows/hides the Review button in inventory header and the `!` badge on the Inventory entry button
- `startInvoiceReview()` — navigates to screen, attaches swipe gesture listeners on `#irev-card`
- `loadReviewItem(idx)` — populates card fields; highlights flagged inputs in danger color
- `reviewSave()` — POSTs to `N8N_INVOICE_CONFIRM`, splices item from `pendingReviews`, stays at same index (array shifted)
- `reviewSkip()` — advances index, wraps around; no network call
- Swipe right = save, swipe left = skip (threshold: 60px horizontal delta)
- No auto-focus on any field — keyboard stays hidden until user taps a field
- `exitInvoiceReview()` — returns to inventory screen

**Frontend flag highlighting (implemented 2026-04-13):**
- `DUPLICATE` items: amber border + tinted card background, amber pill
- Error flag items (`missing_qty`, `missing_price`, etc.): red border + tinted card background
- `OK` items: green pill, no tinting
- Name-only products (no SKU): label shows "Name / नाम", displays `product_name` in SKU spot

**n8n webhooks needed (not yet created):**
- `vasudeva-invoice-review` GET → returns all rows where `invoice_flag` != `CONFIRMED`. Shape: `[{row_id, sku, product_name, barcode, supplier, category, quantity, unit_price, mrp, sizes, colors, expiry_date, invoice_flag}]`
- `vasudeva-invoice-confirm` POST `{row_id, sku, barcode, name, category, expiry, colors, quantity, unit_price, mrp, sizes}` → updates row, sets `invoice_flag = CONFIRMED`, calls vasudeva-addstock

## Invoice Scanning Pipeline (n8n)

WhatsApp image → Gemini extraction → flagging → dedup check → Restock Archive (all items) → WhatsApp reply → shop owner reviews all items in app → invoice-confirm triggers addstock

### Current n8n workflow state (as of 2026-04-13):
Nodes in order:
1. WhatsApp Trigger
2. IF — image type filter: `$json.messages?.[0]?.type` is equal to `image` (optional chaining handles status update webhooks). "Convert types where required" ON.
3. HTTP Request — downloads image binary from `{{ $json.messages[0].image.url }}`. Response Format: File.
4. **Code in JavaScript** — generates invoiceId, extracts whatsapp sender number, converts binary to base64
5. **HTTP Request (Flask)** — POST to `http://localhost:5050/preprocess`, body Using Fields: `imageBase64 = {{ $json.imageBase64 }}`. Preprocesses image (normalize, sharpen, optional perspective correction).
6. **Code "Reconstruct Binary"** — converts Flask's returned imageBase64 back to binary for Gemini, carries sender/invoiceId forward
7. **Analyze an image** (Gemini 2.5 Pro) — primary extraction, Simplify Output ON
8. **Code in JavaScript1** — parses Gemini JSON array into individual items
9. **Code in JavaScript2** — flagging: adds `invoice_flag`, `row_id`, `invoiceId`, `whatsapp_from`, `supplier`
10. **Get Rows — Restock Archive** — fetches all rows for dedup
11. **Code "Dedup Check"** — Run once for all items. Gets invoice items from `$('Code in JavaScript2')`, sheet rows from `$input`. Re-emits invoice items with `_isDuplicate` + `_dupScore`.
12. **IF _isDuplicate** → TRUE: Code "Mark Duplicate" (sets invoice_flag = DUPLICATE) → Append to Restock Archive → WhatsApp text warning → stop. FALSE: continue.
13. **Append to Restock Archive** — one row per item, no Execute Once
14. **WhatsApp reply** (plain Send): `📋 Invoice received: {{ $('Code in JavaScript2').all().length }} items added to review queue.`
15. **Error Trigger** (free-floating) → WhatsApp error reply to hardcoded owner number

**All items go to review queue — nothing routes directly to addstock from this workflow. Addstock is triggered by vasudeva-invoice-confirm webhook when shop owner confirms in app.**

### Code in JavaScript (prep node):
```javascript
const binaryData = await this.helpers.getBinaryDataBuffer(0, 'data');
const invoiceId = `INV_${Date.now()}_${Math.random().toString(36).slice(2,6).toUpperCase()}`;
const sender = $('WhatsApp Trigger').item.json.messages[0].from;
const imageBase64 = binaryData.toString('base64');

return [{
  json: { sender, invoiceId, imageBase64 },
  binary: $input.item.binary
}];
```

### Code "Reconstruct Binary":
```javascript
const preprocessed = $json.imageBase64;
const { sender, invoiceId } = $('Code in JavaScript').first().json;
const buffer = Buffer.from(preprocessed, 'base64');

return [{
  json: { sender, invoiceId, imageBase64: preprocessed },
  binary: {
    data: await this.helpers.prepareBinaryData(buffer, 'invoice.jpg', 'image/jpeg')
  }
}];
```

### Gemini prompt (Analyze an image node):
Model: `gemini-2.5-pro`. Binary input named `data`. Simplify Output: ON.

Extract all product line items from this supplier invoice. Only extract line items explicitly present in the invoice table. Do not infer, duplicate, or create items that are not clearly printed. Compare with row numbers if available. Return a JSON array only, no explanation, no markdown. Each object must have exactly these fields:

- "sku": the supplier's article/item code. NOT the HSN/SAC code. HSN codes are 4–8 digit numbers (e.g. 6109, 610910) — if the only code visible is numeric and looks like an HSN, return null. Copy SKU codes exactly character by character — do not abbreviate, skip, or correct any characters. The numeric product family prefix (e.g. 1722, 1723) is critical — read each digit carefully and do not confuse similar digits. Do not confuse capital letter I with slash / in SKU codes or product names.
- "product_name": only return a value if there is an explicit human-readable product title separate from the SKU. Do NOT infer names from SKU codes. Do NOT use fabric type, category, or HSN descriptions. If no distinct product name is visible, return null.
- "category": the product description, type, or category column. Read the full column value — do not leave blank if text is present in that column. Otherwise null.
- "sender": supplier company name from the invoice header, otherwise null.
- "quantity": total units from the Qty/Quantity column only. Do not sum size columns.
- "unit_price": supplier cost price per unit (rate column)
- "mrp": maximum retail price if visible, otherwise null
- "expiry_date": expiry date if visible, otherwise null
- "tax_rate": the GST/tax percentage for this line (e.g. 12 for 12%). Read from the tax rate column if present, otherwise null.
- "line_total": the actual printed total/amount value for this line item as it appears in the invoice — this is typically unit_price × qty × (1 + tax_rate/100) and represents what the supplier billed including tax. Do not compute it yourself; read the printed value. Null if no total column is visible.
- "sizes": read each size column header and its corresponding cell value for this row independently. Do not skip columns. Do not borrow values from adjacent rows or columns. Size labels must be read in full. Use the sequence of column headers as context — if columns read S, M, L, XL then the fourth column is XL not L. Common sequences: XS/S/M/L/XL/2XL/3XL or numeric 28/30/32/34/36/38/40. Never assign the same label twice in one row. After reading, verify your size quantities sum to the total Qty — if they don't, re-read the row. Return as object e.g. {"32":2,"34":1,"36":2}. Return null if no size columns visible.
- "colors": if color breakdown with quantities return {"Red":2,"Blue":3}; if colors listed without quantities return "Red, Blue"; otherwise null
- "barcode": EAN/UPC barcode if visible, otherwise null

### Code in JavaScript1 (Gemini parser):
```javascript
const text = $input.item.json.content.parts[0].text;
const cleaned = text.replace(/```json|```/g, '').trim();
const products = JSON.parse(cleaned);

return products.map(product => ({
  json: {
    sku: product.sku,
    product_name: product.product_name || '',
    category: product.category || '',
    sender: product.sender || '',
    quantity: product.quantity,
    min_stock: 10,
    unit_price: product.unit_price,
    mrp: product.mrp || '',
    expiry_date: product.expiry_date || '',
    sizes: product.sizes ? JSON.stringify(product.sizes) : '',
    barcode: product.barcode ? String(product.barcode) : '',
    colors: product.colors
      ? (typeof product.colors === 'object' ? JSON.stringify(product.colors) : product.colors)
      : '',
    tax_rate: product.tax_rate ?? null,
    line_total: product.line_total ?? null
  }
}));
```

### Code in JavaScript2 (flagging node):
```javascript
const whatsapp_from = $('Code in JavaScript').first().json.sender;
const invoiceId = $('Code in JavaScript').first().json.invoiceId;

return $input.all().map(item => {
  const p = item.json;
  const issues = [];

  const skuStr = String(p.sku || '');
  if (p.sku && /^\d{4,8}$/.test(skuStr)) issues.push('possible_hsn_as_sku');
  if (!p.sku && !p.product_name) issues.push('missing_sku');
  if (!p.unit_price) issues.push('missing_price');
  if (!p.quantity) issues.push('missing_qty');

  if (p.quantity && p.sizes) {
    try {
      const sizesObj = typeof p.sizes === 'string' ? JSON.parse(p.sizes) : p.sizes;
      const sizeTotal = Object.values(sizesObj).reduce((a, b) => a + b, 0);
      if (Math.abs(sizeTotal - p.quantity) > 0) issues.push('size_qty_mismatch');
    } catch(e) {
      issues.push('sizes_parse_error');
    }
  }

  return { json: {
    ...p,
    supplier: p.sender || null,
    sender: undefined,
    row_id: `${Date.now()}_${Math.random().toString(36).slice(2,7)}`,
    whatsapp_from,
    invoiceId,
    invoice_flag: issues.length ? issues.join(',') : 'OK'
  }};
});
```

### Dedup Check code (Run once for all items):
```javascript
const items = $('Code in JavaScript2').all().map(i => i.json);

const identifiers = [
  ...items.map(i => i.sku).filter(Boolean),
  ...items.map(i => i.product_name).filter(Boolean)
];
const currentSet = new Set(identifiers);

const twoMonthsAgo = Date.now() - (60 * 24 * 60 * 60 * 1000);
const rows = $input.all().filter(row => {
  const ts = new Date(row.json['Timestamp']).getTime();
  return ts > twoMonthsAgo;
});

const invoiceMap = {};
for (const row of rows) {
  const invId = row.json['Invoice ID'];
  if (!invId) continue;
  if (!invoiceMap[invId]) invoiceMap[invId] = [];
  const id = row.json['SKU'] || row.json['Product Name'];
  if (id) invoiceMap[invId].push(id);
}

let bestMatch = 0;
for (const pastIds of Object.values(invoiceMap)) {
  const overlap = pastIds.filter(id => currentSet.has(id)).length;
  const pct = overlap / Math.max(currentSet.size, pastIds.length);
  if (pct > bestMatch) bestMatch = pct;
}

return items.map(item => ({ json: {
  ...item,
  _dupScore: bestMatch,
  _isDuplicate: bestMatch > 0.8
}}));
```
**Critical:** `items` comes from `$('Code in JavaScript2')` not `$input` — Get Rows replaces $input with sheet rows. This is intentional.

### Restock Archive sheet columns:
| Column | Expression |
|--------|-----------|
| Invoice ID | `{{ $json.invoiceId }}` |
| Row ID | `{{ $json.row_id }}` |
| Timestamp | `{{ new Date().toISOString() }}` |
| Supplier | `{{ $json.supplier }}` |
| WhatsApp From | `{{ $json.whatsapp_from }}` |
| SKU | `{{ $json.sku }}` |
| Product Name | `{{ $json.product_name }}` |
| Category | `{{ $json.category }}` |
| Barcode | `{{ $json.barcode }}` |
| Sizes | `{{ $json.sizes }}` |
| Colors | `{{ $json.colors }}` |
| Quantity | `{{ $json.quantity }}` |
| Unit Price | `{{ $json.unit_price }}` |
| MRP | `{{ $json.mrp }}` |
| Tax Rate | `{{ $json.tax_rate }}` |
| Line Total (incl. tax) | `{{ $json.line_total ?? ($json.unit_price * $json.quantity) }}` |
| Expiry Date | `{{ $json.expiry_date }}` |
| Invoice Flag | `{{ $json.invoice_flag }}` |

### addstock HTTP Request JSON body:
```json
{
  "sku": "{{$json.sku}}",
  "barcode": "{{$json.barcode}}",
  "name": "{{$json.product_name}}",
  "category": "{{$json.category}}",
  "quantity": {{$json.quantity}},
  "mrp": {{$json.mrp || 0}},
  "cost": {{$json.unit_price}},
  "expiry": "{{$json.expiry_date}}",
  "sizes": "{{$json.sizes}}",
  "colors": "{{$json.colors}}",
  "sizesOverride": null,
  "colorsOverride": null,
  "timestamp": "{{new Date().toISOString()}}"
}
```
Note: "Never Error" must be ON — addstock returns plain text, not JSON.

### WhatsApp reply nodes:
- **Duplicate detected** (plain Send after Mark Duplicate): `⚠️ Duplicate invoice detected ({{ Math.round($json._dupScore * 100) }}% match). Items added to review queue in the app.`
- **Success** (after Append to Restock Archive): `📋 Invoice received: {{ $('Code in JavaScript2').all().length }} items added to review queue.`
- **Error** (Error Trigger → hardcoded owner number): `❌ Could not process invoice. Please enter manually.`

### invoice_flag values:
`OK`, `missing_sku`, `missing_price`, `missing_qty`, `possible_hsn_as_sku`, `size_qty_mismatch`, `sizes_parse_error`, `CONFIRMED`, `DUPLICATE`

### Known issues (as of 2026-04-13):
1. **Gemini accuracy** — sizes still occasionally wrong; I/slash confusion in product names; category sometimes missed. Prompt updated to address these. May need further tuning.
2. **Flask preprocessing** — Flask image preprocessing node (`/opt/invoice_preprocess.py`, port 5050) exists on the Hetzner VPS but is **no longer used in the active invoice scanning workflow** (removed to simplify the n8n pipeline). The script and its service file (`/tmp/invoice-preprocess.service` on Steam Deck) are preserved in case a future workflow needs preprocessing.
3. **vasudeva-invoice-confirm body expanded (2026-04-15)** — frontend now POSTs `{row_id, sku, barcode, name, category, expiry, colors, quantity, unit_price, mrp, min_stock, sizes}`. The n8n confirm workflow needs updating to pass the new fields through to the vasudeva-addstock call.
4. **addstock JSON body** — numeric fields need `|| 0` fallbacks or they produce invalid JSON when null. Current fixed body is in the addstock HTTP Request JSON body section above.

## Upcoming Features (planned, not yet built)

1. **vasudeva-invoice-review webhook** (GET) — returns all rows from Restock Archive where `invoice_flag` is not `CONFIRMED` or `TRASHED`. Response shape: `[{row_id, sku, product_name, barcode, supplier, category, quantity, unit_price, mrp, sizes, colors, expiry_date, invoice_flag}]`. Frontend calls this on PIN entry and on `goTo('inventory')`.
2. **vasudeva-invoice-confirm webhook** (POST) — body: `{row_id, sku, barcode, name, category, expiry, colors, quantity, unit_price, mrp, sizes}` OR `{row_id, action:'trash'}`. For confirm: updates Restock Archive row to `invoice_flag = CONFIRMED`, calls vasudeva-addstock with all confirmed values. For trash: sets `invoice_flag = TRASHED`, no addstock call.

## Core Validation Rules — DO NOT BREAK

- Save Product requires: barcode OR name OR SKU (any one sufficient)
- Barcode always treated as string — never coerced to number
- `sanitizeBarcode()` applied at every entry and receive point
- Basket dedup key: `sku + size + price + color`

## Backend

- Google Sheet ID: `1GYg5rJcbtQGooJe_Tnfw484YHSObYFYNQn7QqRb6P5s`
- Sizes column: JSON string e.g. `{"M":3,"L":5}`
- Color column: plain string `"Red, Black"` OR JSON `{"Red":5,"Black":3}` — frontend handles both
- Expiry column: plain date string `"2027-01-01"` OR JSON `{"2027-01-01":10,"2027-06-01":5}` — `parseExpiry()` handles both. Null qty in parsed object means old plain-string format (qty unknown). FEFO: sell from earliest date first.
- Slow Mover column: sheet header must be named `slow_mover` (underscore) so n8n maps it correctly. Values: `TRUE`/`FALSE`. Set by scheduled n8n workflow reading Stock Movements tab.
- Last Sale Date column: sheet header `last_sale_date`. Updated by vasudeva-sell webhook on every sale.
- n8n self-hosted: https://n8n.whitelockemedia.com
- n8n runs on a Hetzner CPX22 VPS (95.217.216.83), managed via **Coolify** — use Coolify UI to set environment variables, not the server console or docker-compose directly

## Image Preprocessing Service (in progress as of 2026-04-13)

Goal: preprocess WhatsApp invoice photos (perspective correction, sharpen, normalize) before sending to Gemini to improve table reading accuracy.

### What was tried and failed:
- Python Code node in n8n — sandbox blocks all imports. `N8N_RUNNERS_STDLIB_ALLOW` and `N8N_RUNNERS_EXTERNAL_ALLOW` env vars did not work. Requires custom task runner Docker image + n8n-task-runners.json config file — too complex for Coolify setup.
- Execute Command node — disabled by default in n8n 2.0+. Needs `NODES_INCLUDE=n8n-nodes-base.executeCommand` env var. Added to Coolify but node still didn't appear — may need full redeploy.

### Current approach: Flask API on the server
- cv2, numpy, flask installed on host system (python3.12.3)
- Script written to `/opt/invoice_preprocess.py` — exposes POST `/preprocess` on port 5050
- Accepts `{"imageBase64": "..."}`, returns `{"imageBase64": "..."}` with corrected image
- n8n calls it via HTTP Request node between HTTP Request (image download) and Code in JavaScript (prep node)
- Script created but had syntax errors from Hetzner web console paste mangling special characters

### Blocked on: SSH access
- SSH from Steam Deck (Konsole) failing with `Permission denied (publickey,password)`
- Password auth appears enabled in sshd_config but password not being accepted
- Next step: either fix SSH (try adding SSH key via Coolify, or generate key on Steam Deck with `ls ~/.ssh/`) or find another way to write the script cleanly to the server
- Hetzner web console (`>_`) works but mangles special characters on paste (`:` → `;` etc.)
