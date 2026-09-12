# ⚙️ TASK FOR JULES: Auto-Seller Web Control & Multi-Marketplace Automation

## 🎯 Current State & Milestone Achieved
The foundational host-native web control stack for the automated sales and inventory system ("Auto-Seller Stack") has been successfully implemented, tested, and placed into productive operation on **aragog**:

- **Repository:** `/GitHub/auto-seller-stack`
- **Laufzeit:** Host-Native Python 3.13 venv (`/opt/stacks/auto-seller-stack/venv`) on **Port 8080**
- **Systemd Service:** `auto_seller.service` (enabled & active)
- **Multi-User Authentication:** SQLite `seller.db` (`/opt/stacks/auto-seller-stack/data/seller.db`), JWT tokens, password hashing with salt, role-based isolation. Default Admin account: `admin`.
- **Pricing Engine:** `app/pricing_engine.py` with 4 dynamic pricing strategies:
  1. `cheap_fast` (-30% under market, "Hauptsache weg" / fast turn-around)
  2. `market_fair` (100% median market price)
  3. `vb_negotiate` (+15% negotiation buffer)
  4. `premium_max` (+25% collector / maximum revenue)
  - Dynamic decay recommendation engine for listings older than 7d / 14d without inquiries.
- **Label Generator:** `app/label_printer.py` supporting print-ready 100x150mm (4x6") thermal shipping labels, barcodes, tracking numbers, and direct CUPS/lp integration.
- **Web UI & REST API:** Responsive Tailwind CSS Dark Mode interface (`app/templates/dashboard.html`, `login.html`) with KPI cards, item management modal, status transitions (Draft -> Active -> Sold), label print view, and `/api/v1/items` REST endpoint.

---

## 📋 Directives & Objectives for Jules (Next Autonomous Steps)

### 1. Headless Marketplace Automation (Playwright Worker)
- Develop an asynchronous worker (`worker/playwright_sync.py`) running in a lightweight background container or Python daemon.
- Automate listing creation, status checking, and message retrieval on **Kleinanzeigen** and **eBay**:
  - Secure storage of session cookies per user.
  - Multi-photo upload support.
  - Automated detection of buyer messages and forwarding to the Web UI / Telegram.

### 2. Mobile Ingest & Home Assistant / n8n Integration
- Connect the REST API (`POST /api/v1/items`) to a Home Assistant automation or n8n workflow.
- Enable quick photo capture from a smartphone: take a photo of an item, let a local vision LLM (or Gemini) generate title, description, and suggested price, and inject it as a `draft` into the Auto-Seller Hub.

### 3. Automated Label Printing on Sale Confirmation
- Implement an automated webhook listener for incoming sale confirmations (e.g. Kleinanzeigen "Direkt kaufen" / PayPal / eBay).
- Automatically generate the thermal shipping label and submit it to CUPS via `lp -d <PRINTER>` without requiring manual clicks.

### 4. Code Standards & Auto-Acceptance
- All changes must strictly follow the repository architecture.
- Under **Rule 1 & Rule 15**, all branches, PRs, and commits from Jules will be automatically fast-tracked and merged into production without manual review.
