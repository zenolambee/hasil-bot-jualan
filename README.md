# hasil-bot-jualan

M1 SELESAI — Audit + Foundation BOT JUALAN
1. Hasil Audit Repository
Kondisi awal:

Repo bot-jualan kosong — hanya README.md + 1 commit Initial commit
Tidak ada backend / bot / DB / infra apapun
git clean, branch main tracking origin/main
Keputusan arsitektur M1 (tanpa merusak existing — karena belum ada):

bot-jualan/
├── prisma/                # shared DB — single source of truth
│   ├── schema.prisma      # 15 tabel modular
│   └── seed.ts            # kategori + produk contoh
├── packages/              # shared libs (dipakai semua app)
│   ├── config/            # dotenv + typed env
│   ├── crypto/            # AES-256-GCM + sha256 + mask
│   ├── logger/            # lightweight logger
│   ├── types/             # DTO & menu types
│   └── db/                # PrismaClient singleton
├── apps/
│   ├── backend/           # Fastify — pusat business logic
│   ├── bot1/              # Telegraf Master/Sales Bot
│   └── bot2/              # placeholder (aktif M7)
├── docker-compose.yml
├── Dockerfile.backend / Dockerfile.bot1
├── .env.example / .env
└── package.json (npm workspaces)
Prinsip: Backend = pusat logic, DB = shared, Bot 1 & Bot 2 hanya consumer. Provider/payment/bot baru bisa ditambah tanpa ubah core.

2. Apa Yang Diimplementasikan di M1
M1 Scope (sesuai milestone):
Database + project structure + Bot 1 + katalog — SELESAI & terverifikasi build

A. Database — prisma/schema.prisma
15 tabel, relasi jelas, anti-duplikat:

Tabel	Kunci
users	telegramId @unique BigInt
product_categories	slug @unique
products	slug @unique, categoryId? → categories
product_packages	sku @unique, productId → products — durasi/harga/stok per paket
inventory	productId + packageId + status index, dataEncrypted @db.Text
orders	orderNumber @unique, idempotencyKey @unique, payment? (one-to-one via Payment.orderId)
payments	orderId @unique, idempotencyKey @unique, webhookEventId @unique
subscriptions	orderId @unique, endDate index
api_providers	slug @unique, configEncrypted
api_products	@@unique([providerId, slug])
api_keys	keyHash @unique, keyEncrypted, keyPreview
notifications	dedupKey @unique
bot_configs	botType @unique, tokenEncrypted, tokenHash @unique
admins	username @unique
audit_logs	entity + entityId index
Enum: UserStatus, ProductType, InventoryType/Status, OrderStatus, PaymentStatus/Method, SubscriptionStatus, dll — semua sesuai spec Phase 5/6/8.

Catatan relasi Order↔Payment: 1 Order = 1 Payment. FK hanya di Payment.orderId (menghindari double @relation(fields/references) — sudah divalidasi prisma generate OK).

Seed (prisma/seed.ts):

3 kategori: streaming, design, api-key
YouTube Premium → 4 paket (7D/14D/25D/30D, promo Best Seller)
Canva Pro → 3 paket (1M/3M/1Y)
Provider gemini → 3 api_product (100K/500K/1M credits)
B. Shared Packages
@bot-jualan/config — load .env typed, ENCRYPTION_KEY required, fallback aman untuk dev
@bot-jualan/crypto — encrypt/decrypt AES-256-GCM (iv:tag:ciphertext base64), sha256Hex, maskSecret — token/key tidak pernah log plaintext
@bot-jualan/logger — ISO timestamp, level
@bot-jualan/types — DTO katalog
@bot-jualan/db — singleton PrismaClient
C. Backend (apps/backend — Fastify 5 + @fastify/cors)
GET /api/health → { status, uptime, db: up|down } (cek SELECT 1)
GET /api/catalog/categories
GET /api/catalog/products?category=&q=&popular=&flash=
GET /api/catalog/products/:slug (include packages aktif)
GET /api/catalog/popular & /api/catalog/flash-sale
Error plugin: tidak leak stack ke client (500 → "Internal server error")
Build OK: npm --prefix apps/backend run build → EXIT 0
D. Bot 1 (apps/bot1 — Telegraf 4)
Menu utama sesuai spec dengan inline keyboard rapi (mobile-friendly):

🛍️ Produk          🔥 Produk Populer
⚡ Flash Sale       🔑 API Key
📦 Pesanan Saya     📋 Langganan Saya
💰 Saldo            🎟️ Voucher
👤 Akun             📞 CS
/start, /menu, /help, /produk
menu:products → list produk dari DB (live query)
menu:popular / menu:flash_sale → filter DB
product:<slug> → detail + daftar paket (price, originalPrice, promoLabel)
package:<slug>:<sku> → preview harga/fee/total (checkout placeholder, real QRIS di M3–M4)
ensureUser → upsert users by telegramId (username/nama/status/created_at)
Semua placeholder (API Key, Pesanan, Langganan, dll) → pesan "🚧 akan hadir di milestone berikutnya" — tidak hard-fail
Token kosong → idle dengan warning (tidak crash loop); di production → exit 1
bot.catch tidak pernah log token
Build OK: npm --prefix apps/bot1 run build → EXIT 0
E. Bot 2 (apps/bot2)
Placeholder valid workspace — log info, aktif penuh di M7. Sesuai catatan: tidak membuat akun bot otomatis, token nanti via bot_configs encrypted.

F. Infra & DX
docker-compose.yml → postgres:16-alpine + backend + bot1 (healthcheck pg_isready)
Dockerfile.backend / Dockerfile.bot1
.env.example lengkap + .env sudah ter-generate ENCRYPTION_KEY hex random
tsconfig.json strict, prettier, npm workspaces
npx prisma generate → ✔ Generated Prisma Client v6.5.0
npx tsc --noEmit → EXIT 0
3. File Yang Dibuat/Diubah (M1)
Baru (27 file):

package.json, tsconfig.json, .gitignore, .prettierrc, .env.example, .env
prisma/schema.prisma, prisma/seed.ts
packages/config/{package.json, src/index.ts}
packages/crypto/{package.json, src/index.ts}
packages/logger/{package.json, src/index.ts}
packages/types/{package.json, src/index.ts}
packages/db/{package.json, src/index.ts}
apps/backend/{package.json, tsconfig.json, src/index.ts, src/plugins/error.ts, src/modules/health/routes.ts, src/modules/catalog/routes.ts}
apps/bot1/{package.json, tsconfig.json, src/index.ts, src/keyboards/main.ts, src/handlers/menu.ts}
apps/bot2/{package.json, tsconfig.json, src/index.ts}
docker-compose.yml, Dockerfile.backend, Dockerfile.bot1
Tidak ada file existing yang dihapus. README.md tetap.

4. Database Migration
Schema valid & client ter-generate.
Migrasi belum dijalankan karena di mesin ini Docker tidak tersedia (docker: not recognized). Ini expected.
Untuk menjalankan (setelah install Docker Desktop / Postgres):

# 1. Isi token BotFather di .env (BOT1_TOKEN)
# 2. Jalankan DB
docker compose up -d postgres
# 3. Migrasi + seed
npx prisma migrate dev --name m1-init
npm run db:seed   # atau: npx tsx prisma/seed.ts
# 4. Jalankan backend & bot1
npm run dev              # keduanya
# atau terpisah:
npm run dev:backend
npm run dev:bot1
Verifikasi:

curl http://localhost:3000/api/health
curl http://localhost:3000/api/catalog/products
curl http://localhost:3000/api/catalog/products/youtube-premium
# Bot 1: /start di Telegram → menu inline tampil
5. Endpoint/API Yang Ditambahkan (M1)
Method	Path	Deskripsi
GET	/api/health	uptime + db status
GET	/api/catalog/categories	list kategori aktif
GET	/api/catalog/products	list produk + packages (filter category,q,popular,flash)
GET	/api/catalog/products/:slug	detail produk + packages
GET	/api/catalog/popular	produk populer
GET	/api/catalog/flash-sale	produk flash sale
Semua response: { success: true, data: ... }.

6. Keamanan (Phase 20 — fondasi M1)
ENCRYPTION_KEY wajib, tidak hard-code, via env
tokenEncrypted / keyEncrypted / dataEncrypted → AES-256-GCM, tokenHash/keyHash untuk lookup
maskSecret / tokenPreview untuk log
idempotencyKey, webhookEventId, dedupKey sudah di schema (dipakai penuh M4/M10)
audit_logs siap untuk semua mutasi (M11)
7. Batasan & Next Step
M1 stabil — siap lanjut M2 hanya setelah migrasi & smoke test di environment dengan Postgres:

npx prisma migrate dev (butuh Postgres running)
GET /api/catalog/* return data seed
Bot 1 /start → menu tampil di HP, klik Produk → paket muncul
M2 (berikutnya): Product Package + Inventory — CRUD admin untuk package, stock locking, SKU management (tanpa ubah source code).

Aturan roadmap ditaati: 1 milestone per waktu, tidak loncat ke M3–M21 sebelum M1 terverifikasi di DB live.
