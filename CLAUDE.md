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
4. **Fast confirm swiping race condition** — rapidly confirming items in Invoice Review can cause addstock calls to race. The addstock match logic finds blank rows when two near-simultaneous requests both look for a product that doesn't exist yet. The `.trim()` + non-empty check fix (2026-04-19) prevents matching blank rows, but two new products arriving at the same instant can still collide on Google Sheets append (same as the invoice queue append race). Potential fix: add a staggered delay in `reviewSave()` or queue confirms client-side.

## Invoice Review Screen (`screen-invoicereview`)

Launched from Inventory via "⚠ Review (n)" button (only visible when `pendingReviews.length > 0`). Card-based swipe UI, one item at a time — same screen-level swipe IIFE pattern as barcode mapping.

### Key functions
- `loadPendingReviews()` — async fetch from `N8N_INVOICE_REVIEW`; called on PIN entry and `goTo('inventory')`
- `updateReviewBadge()` — shows/hides Review button + `!` badge; counts only unconfirmed items (`irevConfirmedIds` Set)
- `startInvoiceReview()` — navigates to screen, resets `irevIndex`, `irevSavedCount`, `irevFieldCache`, `irevConfirmedIds`
- `loadReviewItem(idx)` — populates all fields from `pendingReviews[idx]`; restores `irevFieldCache` if user was here before
- `irevSaveToCache()` — snapshots all field values + `irevSizeValues` into `irevFieldCache[item.row_id]`
- `reviewSave()` — POSTs to `N8N_INVOICE_CONFIRM`, adds `row_id` to `irevConfirmedIds` (does NOT splice array), calls `irevAdvanceToNext()`
- `irevAdvanceToNext()` — finds next unconfirmed item; calls `showIrevDone()` if all confirmed
- `reviewSkip()` — saves to cache, calls `irevAdvanceToNext()`
- `reviewPrev()` — saves to cache, decrements `irevIndex`; blocked only at index 0
- `reviewTrash()` — double-tap: marks confirmed + posts `{row_id, action:'trash'}`
- `exitInvoiceReview()` — returns to inventory

### Layout
- `#irev-identity` — fixed (non-scrolling): supplier, flags, barcode (scannable), SKU, Name, lookup results
- `#irev-content` — scrollable: Category, Expiry, Qty, Cost, MRP, Min Stock, Tax Rate %, Invoice Total (readonly), Sizes (tab+grid), Colors
- `#irev-footer` — Trash, Skip, Save buttons

### Swipe gesture (screen-level IIFE)
- Left swipe → forward: `irevSaveToCache()`, then `reviewSave()` if qty/price filled, else `reviewSkip()`
- Right swipe → backward: `reviewPrev()`
- Boundary: wiggle at index 0 (right) or last unconfirmed (left)
- Threshold: 60px horizontal delta

### Field cache
- `irevFieldCache` keyed by `row_id` — preserves edits when navigating backward
- `irevConfirmedIds` (Set) — tracks confirmed row_ids in-session without splicing array; enables backward navigation

### Inventory autofill
- Barcode/SKU/Name inputs trigger `irevLookup()` (350ms debounce) → `irevShowLookupResults()` → `irevApplyProduct(idx)`
- Autofill populates: SKU, Name, Barcode, Category, MRP, Min Stock, Colors from `productCache`
- Does NOT fill Qty or Cost (those come from the invoice)

### Flag highlighting
- `DUPLICATE`: amber border + tinted card, amber pill
- Error flags (`missing_qty`, `missing_price`, `possible_hsn_as_sku`, `size_qty_mismatch`, `total_mismatch`, etc.): red border + tinted card
- `OK`: green "None" label, no tinting

### reviewSave() POST body (vasudeva-invoice-confirm)
```javascript
{
    row_id, sku, barcode, name, category, expiry, colors,
    quantity, unit_price, mrp, min_stock,
    sizes,      // object from getIrevSizes() — n8n receives as parsed object
    tax_rate,   // from irev-tax field (editable)
    line_total  // computed: qty * price * (1 + taxRate/100)
}
// OR for trash: { row_id, action: 'trash' }
```

### vasudeva-invoice-review webhook (GET)
Returns all Restock Archive rows where `invoice_flag` not in `['CONFIRMED','TRASHED']`.
Response shape: `[{row_id, sku, product_name, barcode, supplier, category, quantity, unit_price, mrp, tax_rate, line_total, sizes, colors, expiry_date, invoice_flag}]`

### vasudeva-invoice-confirm webhook (POST)
- Receives body above
- Updates Restock Archive row (all editable fields + sets `invoice_flag = CONFIRMED`)
- Calls vasudeva-addstock with: `{sku, barcode, name, category, quantity, mrp, cost (=unit_price), expiry, colors, minStock, sizes (JSON.stringify'd), timestamp}`
- For trash: sets `invoice_flag = TRASHED`, no addstock call

### n8n confirm workflow — sizes handling
`body.sizes` arrives as a parsed JS object (n8n parses JSON bodies). In the addstock HTTP Request JSON body use:
```
"sizes": "{{ $('Webhook').item.json.body.sizes ? JSON.stringify($('Webhook').item.json.body.sizes) : '' }}",
```
In the Update Row (Restock Archive) use same expression.

### vasudeva-addstock Code node — sizes fix needed
Must handle both string and object input (invoice-confirm sends object, other callers may send string):
```javascript
const sizesStr = typeof p.sizes === 'object' && p.sizes ? JSON.stringify(p.sizes) : (p.sizes || '');
```

## Invoice Scanning Pipeline (n8n)

Telegram image → queue to Google Sheet → sequential Gemini extraction → flagging → dedup check → Restock Archive → Telegram reply → shop owner reviews in app → invoice-confirm triggers addstock

### Current n8n workflow state (as of 2026-04-19):
Two workflows: **Invoice Scanner** (intake + queue) and **vasudeva-process-queue** (sequential processing). WhatsApp workflow abandoned — Telegram only.

**Why two workflows:** Gemini can only process one invoice at a time. When users batch-send multiple images (Telegram sends each as a separate message), the intake workflow queues them and the process-queue workflow handles them one at a time via a self-trigger loop.

**Environment requirement:** `N8N_RESTRICT_FILE_ACCESS_TO=/tmp` must be set in Coolify env vars — n8n 2.0+ blocks all file system access by default.

---

### Workflow 1: Invoice Scanner (intake)

**Node chain:** Telegram Trigger → IF (image filter) → HTTP Request (getFile) → HTTP Request (download) → Code - Queue Item → Write File → Code - Add Delay → Append Row (Invoice Queue) → Reply → Get Many (Invoice Queue) → Code - Check Busy → IF (busy) → FALSE: HTTP Request (trigger process-queue)

1. **Telegram Trigger** — "On message", "Download Images/Files" ON, Image Size: Large. Binary key: `data`.
2. **IF** — `$json.message.photo` is not empty. "Convert types where required" ON.
3. **HTTP Request (getFile)** — `https://api.telegram.org/bot<TOKEN>/getFile?file_id={{ $('Telegram Trigger').item.json.message.photo[3].file_id }}`
4. **HTTP Request (download)** — `https://api.telegram.org/file/bot<TOKEN>/{{ $json.result.file_path }}`. Response Format: File.
5. **Code - Queue Item**:
```javascript
const binary_data = await this.helpers.getBinaryDataBuffer(0, 'data');
const queue_id = `INV_${Date.now()}_${Math.random().toString(36).slice(2,6).toUpperCase()}`;
const file_path = `/tmp/invq_${Date.now()}_${Math.random().toString(36).slice(2,6)}.jpg`;
const chat_id = String($('Telegram Trigger').item.json.message.chat.id);
const sender = String($('Telegram Trigger').item.json.message.from.id);
return [{
  json: { queue_id, file_path, sender, chat_id, source: 'telegram' },
  binary: { data: await this.helpers.prepareBinaryData(binary_data, 'invoice.jpg', 'image/jpeg') }
}];
```
6. **Write File** — writes binary to `{{ $json.file_path }}` on disk (`/tmp/invq_*.jpg`)
7. **Code - Add Delay** — staggered delay to prevent Google Sheets append race condition when batch-sending images:
```javascript
const msg_id = $('Telegram Trigger').item.json.message.message_id;
await new Promise(r => setTimeout(r, (msg_id % 10) * 1500));
return $input.all();
```
8. **Append Row (Invoice Queue sheet)** — columns: Queue ID, File Path, Sender, Chat ID, Source, Timestamp, Status (set to `PENDING`)
9. **Reply** — Telegram "Send a text message": `📋 Invoice queued for processing.` Chat ID: `{{ $('Code - Queue Item').item.json.chat_id }}`. Append n8n Attribution: OFF.
10. **Get Many (Invoice Queue)** — fetches all rows to check if processing is already running
11. **Code - Check Busy**:
```javascript
const rows = $input.all();
const processing = rows.find(r => r.json.Status === 'PROCESSING');
return [{ json: { busy: !!processing } }];
```
12. **IF** — `{{ $json.busy }}` equals `true`. TRUE: stop (already processing). FALSE: trigger process-queue.
13. **HTTP Request** — POST to `https://n8n.whitelockemedia.com/webhook/vasudeva-process-queue` (empty body, just kicks off processing)

### Invoice Queue sheet columns:
| Column | Value |
|--------|-------|
| Queue ID | `{{ $json.queue_id }}` |
| File Path | `{{ $json.file_path }}` |
| Sender | `{{ $json.sender }}` |
| Chat ID | `{{ $json.chat_id }}` |
| Source | `{{ $json.source }}` |
| Timestamp | `{{ new Date().toISOString() }}` |
| Status | `PENDING` |

---

### Workflow 2: vasudeva-process-queue

**Node chain:** Webhook → Get Many (Invoice Queue) → Code - Route → IF (action=process) →
- TRUE: Update Row (mark PROCESSING) → Read File → Code - Prep for Gemini → Gemini → Parse → Flag → Get Rows (Restock Archive) → Dedup → IF dup → Append to Archive → Reply → Update Row (mark DONE) → HTTP Request (self-trigger)
- FALSE: IF (action=cleanup) → TRUE: Clear sheet (keep headers). FALSE: stop.

**Self-trigger loop:** After marking DONE, the workflow POSTs to its own webhook URL. Code - Route picks the next PENDING item or runs cleanup if none remain.

1. **Webhook** — POST `https://n8n.whitelockemedia.com/webhook/vasudeva-process-queue`
2. **Get Many (Invoice Queue)** — all rows
3. **Code - Route**:
```javascript
const rows = $input.all();
const processing = rows.find(r => r.json.Status === 'PROCESSING');
if (processing) return [{ json: { action: 'skip' } }];
const pending = rows.filter(r => r.json.Status === 'PENDING')
  .sort((a, b) => new Date(a.json.Timestamp) - new Date(b.json.Timestamp));
if (!pending.length) return [{ json: { action: 'cleanup' } }];
const next = pending[0].json;
return [{ json: { action: 'process', ...next } }];
```
4. **IF** — `{{ $json.action }}` equals `process` (must be toggled to expression mode)
5. **Update Row** — matches Queue ID = `{{ $json['Queue ID'] }}`, sets Status = `PROCESSING`
6. **Read File** — reads from `{{ $('Code - Route').item.json['File Path'] }}`
7. **Code - Prep for Gemini**:
```javascript
const route = $('Code - Route').item.json;
const binaryData = await this.helpers.getBinaryDataBuffer(0, 'data');
const imageBase64 = binaryData.toString('base64');
const buffer = Buffer.from(imageBase64, 'base64');
return [{
  json: {
    sender: route.Sender,
    invoiceId: route['Queue ID'],
    chatId: route['Chat ID'],
    source: route.Source,
    filePath: route['File Path'],
    imageBase64
  },
  binary: { data: await this.helpers.prepareBinaryData(buffer, 'invoice.jpg', 'image/jpeg') }
}];
```
8. **Analyze an image** (Gemini 2.5 Pro) — Simplify Output ON, binary input named `data`
9. **Parse Gemini Response** (Code) — parses Gemini JSON array into individual items (same as below)
10. **Flag Invoice Items** (Code) — references `$('Code - Prep for Gemini')` for sender, invoiceId, chatId, source, filePath
11. **Get Rows — Restock Archive** — fetches all rows for dedup
12. **Dedup Check** (Code) — Run once for all items. Gets items from `$('Flag Invoice Items')`, sheet rows from `$input`.
13. **IF _isDuplicate** → TRUE: **Mark Duplicate** → Append to Restock Archive → Telegram duplicate warning → stop. FALSE: continue.
14. **Append to Restock Archive** — one row per item, no Execute Once
15. **Reply** — Telegram "Send a text message": `📋 Invoice received: {{ $('Parse Gemini Response').all().length }} items added to review queue.` Chat ID: `{{ $('Code - Prep for Gemini').item.json.chatId }}`. Run once for all items. Append n8n Attribution: OFF.
16. **Update Row (mark DONE)** — matches Queue ID = `{{ $('Code - Prep for Gemini').item.json.invoiceId }}`, sets Status = `DONE`
17. **HTTP Request (self-trigger)** — POST to `https://n8n.whitelockemedia.com/webhook/vasudeva-process-queue` (loops back to process next item)
18. **Error Trigger** → hardcoded owner chat ID

**FALSE branch of action=process IF:**
- **IF (action=cleanup)** — `{{ $json.action }}` equals `cleanup`
- TRUE: **Clear sheet** — clear operation on Invoice Queue sheet, keepFirstRow: true (preserves headers, deletes all DONE rows)
- FALSE: stop (action was `skip`)

**All items go to review queue — nothing routes directly to addstock. Addstock is triggered by vasudeva-invoice-confirm when shop owner confirms in app.**

### Temp file cleanup
Queue images are saved to `/tmp/invq_*.jpg`. These are cleaned up when the queue sheet is cleared (action=cleanup). For additional safety, a cron job on the VPS can clean old files: `find /tmp -name 'invq_*' -mmin +1440 -delete` (not yet set up).

---

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

### Parse Gemini Response (Code):
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

### Flag Invoice Items (Code):
```javascript
const whatsapp_from = $('Code - Prep for Gemini').first().json.sender;
const invoiceId = $('Code - Prep for Gemini').first().json.invoiceId;

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

  if (p.line_total && p.unit_price && p.quantity) {
    const lt = parseFloat(p.line_total);
    const preTax = p.unit_price * p.quantity;
    const withTax = p.tax_rate != null
      ? p.unit_price * p.quantity * (1 + p.tax_rate / 100)
      : null;
    const tol = n => Math.max(1, n * 0.02); // 2% or ₹1, whichever larger
    const okPreTax  = Math.abs(lt - preTax) <= tol(preTax);
    const okWithTax = withTax !== null && Math.abs(lt - withTax) <= tol(withTax);
    if (!okPreTax && !okWithTax) issues.push('total_mismatch');
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
const items = $('Flag Invoice Items').all().map(i => i.json);

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
**Critical:** `items` comes from `$('Flag Invoice Items')` not `$input` — Get Rows replaces $input with sheet rows. This is intentional.

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

### Reply messages:
- **Intake (Invoice Scanner)**: `📋 Invoice queued for processing.`
- **Success (process-queue)**: `📋 Invoice received: {{ $('Parse Gemini Response').all().length }} items added to review queue.`
- **Duplicate detected** (after Mark Duplicate): `⚠️ Duplicate invoice detected ({{ Math.round($json._dupScore * 100) }}% match). Items added to review queue in the app.`
- **Error** (Error Trigger → hardcoded owner chat ID): `❌ Could not process invoice. Please enter manually.`

### invoice_flag values:
`OK`, `missing_sku`, `missing_price`, `missing_qty`, `possible_hsn_as_sku`, `size_qty_mismatch`, `sizes_parse_error`, `total_mismatch`, `CONFIRMED`, `DUPLICATE`, `TRASHED`

### Known issues (as of 2026-04-19):
1. **Gemini accuracy** — sizes still occasionally wrong; I/slash confusion in product names; category sometimes missed. Prompt tuned but may need further work.
2. **Flask preprocessing** — script at `/opt/invoice_preprocess.py` (port 5050) on Hetzner VPS preserved but not used. Removed from active workflow to simplify pipeline.
3. ~~**vasudeva-addstock Code node — sizes handling**~~ — FIXED (2026-04-19). Parse body Code node now handles both object and string for `sizes` field.
4. **addstock HTTP Request JSON body** — numeric fields need `|| 0` fallbacks. Do NOT use `={{ }}` syntax in JSON body — use `{{ }}` only. String fields: `"{{ expr }}"`. Numeric: `{{ expr }}`.
5. **n8n expressions in HTTP Request JSON body** — line breaks inside `{{ }}` cause the expression to be written as literal text. Must be single-line. Copy-paste from this doc will mangle them — type manually.
6. **Google Sheets append race condition** — batch-sent Telegram images arrive as separate messages processed concurrently. Two simultaneous Append Row calls target the same "next empty row". Fixed in Invoice Scanner with staggered delay based on `message_id % 10 * 1500ms`.
7. **Temp file cleanup** — `/tmp/invq_*.jpg` files accumulate. Queue sheet cleared on cleanup action but files on disk not deleted by n8n. Suggested cron: `find /tmp -name 'invq_*' -mmin +1440 -delete` (not yet set up).

## vasudeva-addstock Workflow Structure

**Node chain:** Webhook → Code in JavaScript (parse body) → Get Rows (inventory sheet) → Code in JavaScript1 (match product) → IF (found) → Update Row / Append Row

### Code in JavaScript (parse body)
```javascript
const b = $input.item.json.body;
return [{
  json: {
    barcode: b.barcode,
    sku: b.sku,
    name: b.name,
    category: b.category,
    mrp: parseFloat(b.mrp) || 0,
    cost: parseFloat(b.cost) || 0,
    quantity: parseInt(b.quantity) || 0,
    minStock: parseInt(b.minStock) || 10,
    expiry: b.expiry || '',
    supplier: b.supplier || '',
    color: b.color || '',
    colorsOverride: b.colorsOverride || null,
    sizesOverride: b.sizesOverride || null,
    sizes: typeof b.sizes === 'object' && b.sizes ? JSON.stringify(b.sizes) : (b.sizes || ''),
    timestamp: b.timestamp
  }
}];
```

### Code in JavaScript1 (match product)
Matches incoming product against inventory sheet rows by barcode, SKU, or name. All values are trimmed and lowercased before comparison. Both sides must be non-empty to match — prevents false matches against blank rows.
```javascript
const incoming = $('Code in JavaScript').item.json;
const rows = $input.all();

const match = rows.find((item, index) => {
  const r = item.json;
  const qname = (incoming.name || '').trim().toLowerCase();
  const rname = (r['Product Name'] || '').trim().toLowerCase();
  const qsku = (incoming.sku || '').trim().toLowerCase();
  const rsku = (r.SKU || '').trim().toLowerCase();
  const qbarcode = String(incoming.barcode || '').trim();
  const rbarcode = String(r.Barcode || '').trim();
  return (
    (qbarcode.length > 0 && rbarcode.length > 0 && qbarcode === rbarcode) ||
    (qsku.length > 0 && rsku.length > 0 && (rsku.includes(qsku) || qsku.includes(rsku))) ||
    (qname.length > 2 && rname.length > 2 && (rname.includes(qname) || qname.includes(rname)))
  );
});

if (!match) return [{ json: { found: false } }];

const rowIndex = rows.indexOf(match);
const r = match.json;
return [{ json: { found: true, row_number: rowIndex + 2, ...r } }];
```

### Key behaviors
- If found: updates existing row (by row_number)
- If not found: appends new row
- `sizes` field: Code in JavaScript handles both object and string input (invoice-confirm sends string, other callers may send either)
- Race condition fix (2026-04-19): added `.trim()` and non-empty checks on both sides to prevent matching blank rows when two near-simultaneous requests both find the same empty row

## vasudeva-invoice-confirm Workflow Structure

**Node chain:** Webhook → IF (trash check) → Update Rows (Restock Archive) → Add Stock HTTP Request (calls vasudeva-addstock)

### HTTP Request body (JSON — Using Fields mode)
All fields reference `$('Webhook').item.json.body.*`. Key mapping:
- `cost` ← `unit_price`
- `minStock` ← `min_stock`
- `sizes` arrives as a pre-stringified JSON string from the frontend (as of 2026-04-19)

### n8n expression gotchas
- `JSON.stringify()` inside HTTP Request JSON body mode causes "JSON parameter needs to be valid JSON" validation error — n8n validates the template before evaluating expressions
- `toJsonString()` also rejected
- Raw body mode makes the entire body arrive as a string at the target webhook
- Solution: stringify sizes in the **frontend** before sending, so all downstream nodes just pass a plain string

## Upcoming Features (planned, not yet built)

1. **Stock Movements tab** — append logic needed in both vasudeva-addstock AND vasudeva-sell workflows.
2. **Slow mover scheduled workflow** — not yet built.
3. **Expiry batch deduction in vasudeva-sell** — not yet built.

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
