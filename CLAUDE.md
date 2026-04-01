# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Vasudeva Shop** is a single-file, browser-based Point of Sale (POS) and Inventory Management System for retail operations. The entire application lives in `vasudeva-pos.html` — a monolithic file with ~2,050 lines of inline HTML, CSS, and JavaScript. No build system, no package manager, no external dependencies.

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
2. `<body>` — Four main screens + modals + toast notification element
3. `<script>` — All application logic (~1,100 lines)

### Screen State Machine

```
PIN Entry (screen-pin) → Entry Menu (screen-entry) → Sell (screen-sell)
                                                    → Add Stock (screen-addstock)
                                                    → Inventory (screen-inventory)
```

Navigation is handled by `goTo(screenId)` which applies CSS transforms. `goBack()` returns to the entry menu.

### Configuration (line ~720)

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
```

### Global State Variables (line ~766)

```javascript
let currentScreen, currentProduct, currentQty, currentPrice;
let sellSizeRows, sellSizeAvail, sellColorRows, sellColorAvail;
let basket = [], productCache = null, cacheLoading = false;
let stockProductFound = false, currentSuggestion = null;
```

### Key Functional Modules

| Area | Key Functions |
|------|---------------|
| Navigation | `goTo()`, `goBack()`, `switchTab()` |
| Barcode | `handleBarcodeScan()`, `searchCache()`, `lookupProduct()`, `sanitizeBarcode()` |
| Sell flow | `displayProduct()`, `renderSizePicker()`, `addToBasket()`, `finalizeOrder()` |
| Stock entry | `submitStock()`, `showCurrentStockSizes()`, `getSizeBreakdown()`, `getColorBreakdown()` |
| Inventory | `renderInventory()`, `editProductFromInventory()`, `toggleInvItem()` |
| Autocomplete | `showGhosts()`, `acceptSuggestion()`, `lookupBy()` |
| UI utilities | `openNumpad()`, `openDiscount()`, `openCamera()`, `showToast()` |
| Colors | `loadGlobalColors()`, `saveCustomColors()` |

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
- Custom colors are persisted in localStorage under a specific key
- The discount flow requires the owner PIN (`OWNER_PIN` in CONFIG) to unlock price override
- Stock status thresholds: green = in stock, amber = stock ≤ `minStock` (default 10), red = out of stock

## Known Issues (do not break workarounds)

1. **Color quantities not saving to sheet** — `incoming.colors` may not be reaching vasudeva-addstock Code JS1 correctly. Needs investigation.
2. **Sell by color unconfirmed** — color deduction code exists in vasudeva-sell JS1, guarded by `colorsRaw.trim().startsWith('{')`. Won't run until color quantities are in JSON format in sheet.
3. **Size/quantity drift** — when `sizeTotal > stock`, frontend trusts sizes as ground truth. Sheet Quantity cell not auto-corrected. Manual fix needed until reconciliation workflow is built.

## Core Validation Rules — DO NOT BREAK

- Save Product requires: barcode OR name OR SKU (any one sufficient)
- Barcode always treated as string — never coerced to number
- `sanitizeBarcode()` applied at every entry and receive point
- Basket dedup key: `sku + size + price + color`

## Backend

- Google Sheet ID: `1GYg5rJcbtQGooJe_Tnfw484YHSObYFYNQn7QqRb6P5s`
- Sizes column: JSON string e.g. `{"M":3,"L":5}`
- Color column: plain string `"Red, Black"` OR JSON `{"Red":5,"Black":3}` — frontend handles both
- n8n self-hosted: https://n8n.whitelockemedia.com
