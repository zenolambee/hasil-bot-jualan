PHASE 27 RESULT
Repository: bot-jualan (source aktif Bot 1 apps/bot1). hasil-bot-jualan tidak ada di disk — tidak disentuh. Bot 2 tidak diubah. QRIS tetap mock, Invite tetap manual.

COST / HPP
Level: ProductPackage (paling tepat — harga & modal beda per package). Alternatif inventory-level ditolak: package adalah unit jual, inventory hanya stok fisik per item.
Prisma: ProductPackage.costPrice Int? (Int rupiah, null=belum diatur, >=0 via Zod, no float/Decimal money via float), Order.unitCost Int? snapshot. prisma validate VALID, generate v6.5.0 OK. Migration belum dijalankan (no DB) — dicatat honest.
Admin: POST /api/admin/packages / PUT /api/admin/packages/:id {costPrice} — zod min(0) (negative ditolak), cost > price diizinkan dengan margin negatif (warning), audit package.cost_update {sku, from, to} tanpa secret. Public GET /api/catalog/* tidak expose costPrice.
HISTORICAL COST
POST /api/orders snapshot unitCost = package.costPrice ?? null sekali saat create (transaction). package.costPrice diubah kemudian tidak overwrite Order.unitCost. Analytics membaca order.unitCost history, bukan live package cost.
PROFIT FORMULA (no NaN/Infinity, no dummy)
Revenue = unitPrice * qty | Cost = unitCost * qty (jika null → Historical cost unavailable) | Profit = Revenue - Cost | Margin = 0 jika Revenue==0 else profit/revenue*100.

SALES ANALYTICS UPGRADE (Phase 26 → 27, reuse struktur)
KPI Backend	Revenue	Cost	Profit	Margin	Paid/Completed/Total/AOV + change vs previous (hanya jika denominator valid)
Endpoint: GET /api/admin/analytics/overview + sales (profit trend) + products (sort revenue/profit/units/margin/orders, lowMargin filter, Top by Profit/Revenue labels) + categories (revenue/cost/profit/margin/%) + customers (orders/revenue/profit/margin/aov/last/first) + product/:id (detail trend + package breakdown) + low-margin + insights + forecast + export = 13 endpoints (12 with requireRole, export ADMIN/OPERATOR). Semua period `today	yesterday	last7	last30	thisMonth	lastMonth
Profit Trend: SalesTrend dengan 3 bars/hari Revenue/Cost/Profit + tooltip Tanggal/Revenue/Cost/Profit/Margin, legend, responsive minWidth.
Low Margin: default <10%, threshold configurable via query ?threshold=N, UI checkbox + input, label Low Margin.
Insights: hanya dari data (revenue change, top product %, best margin, low stock, payment rate). Jika kosong → Belum cukup data untuk insight.
Forecast: jika <7 valid orders → Data historis belum cukup untuk estimasi. else estimasi avgDaily * 7/30 dengan method Rata-rata harian… Estimasi saja, bukan jaminan.
Existing orders tanpa snapshot: revenue tetap, profit Historical cost unavailable (tidak difake).
Export CSV 15 kolom: Order ID, Date, Customer, Product, Package, Quantity, Unit Price, Unit Cost, Revenue, Cost, Profit, Margin %, Payment Status, Order Status, Delivery Status — period-filtered, no secret.
SECURITY
Catalog tidak expose HPP, export tanpa password/token/keyEncrypted, analytics requireRole("SUPER_ADMIN","ADMIN","OPERATOR","SUPPORT"), customer callbacks tetap telegramId == order.user.telegramId → 403.

Check	Result
Cost/HPP	PASS
Historical Cost Snapshot	PASS
Revenue	PASS
Gross Profit	PASS
Margin	PASS
Product Profit	PASS
Category Profit	PASS
Customer Analytics	PASS
Low Margin	PASS
Sales Insights	PASS
Forecast	AVAILABLE (>=7 orders) else INSUFFICIENT DATA honest
Export	PASS
Security	PASS
Responsive UI	PASS
Tests: 140 passed / 0 failed (6 files: unit 24 + inventory 4 + invite 26 + phase25 28 + analytics 23 + phase27 35)
Typecheck: PASS (tsc --noEmit)
Lint: PASS (tsc --noEmit)
Backend Build: PASS
Admin Build: PASS (41 modules → 298.27 kB gzip 91.39 kB)
Integration: SKIPPED (no PostgreSQL/Docker available → prisma validate PASS, no migrate claim)
Known limitations (nyata): DB/Postgres tidak tersedia → migrasi costPrice/unitCost belum dijalankan (apply npx prisma migrate dev saat DB ada); hasil-bot-jualan directory tidak ada (not a git repo); real QRIS gateway belum dikonfigurasi (mock only); invite tetap manual; apps/*/dist build artifacts; forecast sederhana moving average — estimasi saja.

Audit: git status → README.md modified + untracked project files (expected — repo initial hanya README); .gitignore .env, .env.* !.env.example — no .env committed; no token/password/API key in diff; no fake sales/profit; no debug files; hasil-bot-jualan not touched.

Commit siap: feat: add cost profit sales insights (tunggu git add + push ke branch source Bot 1 saat remote tersedia; jangan push ke hasil-bot-jualan).
