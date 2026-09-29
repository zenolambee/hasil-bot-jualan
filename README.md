PHASE 24 — YOUTUBE PREMIUM INVITE DELIVERY — FINAL REPORT
Status: PASS (implemented, typecheck/build/test/prisma PASS. DB migrate belum dijalankan — no Postgres/Docker di host ini, reported honestly)

1. DeliveryMethod
prisma/schema.prisma:

enum DeliveryMethod { STOCK INVITE LICENSE VOUCHER SERIAL MANUAL }
enum DeliveryStatus { PENDING_CUSTOMER_DATA PENDING_INVITE INVITE_SENT COMPLETED FAILED }
ProductPackage.deliveryMethod DeliveryMethod @default(STOCK) — bukan hard-code YouTube. Admin packages/routes.ts support deliveryMethod enum. Seed YT-7D/14D/25D/30D di-set deliveryMethod=INVITE, Canva tetap STOCK.

2. Database Changes
OrderStatus +WAITING_CUSTOMER_DATA (PAID→WAITING_CUSTOMER_DATA→PROCESSING|FAILED)
Order.customerDataEncrypted String? @db.Text (duplicate di OrderDelivery untuk audit)
OrderDelivery baru: id, orderId @unique, method, status, customerDataEncrypted, providerReference, adminNote, createdAt, updatedAt, completedAt + @@index([status]), @@index([method]), @@index([createdAt]) — tidak duplicate model (tidak ada model Delivery sebelumnya)
prisma validate: valid, prisma generate v6.5.0: OK
3. YouTube Flow (berbeda dari STOCK)
STOCK (Canva/dll): PENDING→WAITING_PAYMENT→PAID→PROCESSING→COMPLETED (consumeReservedInventory)
INVITE (YT): PENDING→WAITING_PAYMENT→PAID→WAITING_CUSTOMER_DATA(PENDING_CUSTOMER_DATA)→email→PENDING_INVITE→INVITE_SENT→COMPLETED
API Key: PAID→PROCESSING→adapter.createKey→COMPLETED (tidak tersentuh)
delivery.ts branching isInvite = deliveryMethod===INVITE — jika INVITE, tidak consume inventory, buat OrderDelivery via upsert, notifikasi invite.waiting_email ("📧 Masukkan email Google… ⚠️ Jangan kirim password/OTP"), return inviteWaiting. STOCK tetap code path lama.

4. Bot 1 Flow + Ownership
apps/bot1/src/lib/api.ts: submitCustomerEmail, getDelivery, confirmCustomerData apps/bot1/src/handlers/menu.ts:

buy:SKU → jika deliveryMethod=INVITE hint "Pesanan INVITE — setelah pembayaran diminta email Google"
order:view → jika WAITING_CUSTOMER_DATA tombol 📧 Masukkan Email Google (invite:wait:orderId)
invite:wait → waitingEmail Map<telegramId → orderId> + prompt "📧 Masukkan email Google … Contoh: nama@gmail.com ⚠️ Jangan kirim password/OTP"
text handler → jika waitingEmail.has(uid) + mengandung @ → validateGoogleEmail(normalize), encrypt, POST /api/orders/:id/customer-data, tampilkan u***@gmail.com Paket: … Status: ⏳ Menunggu proses invite + [✅ Konfirmasi][✏️ Ganti Email][❌ Batalkan]
invite:confirm → POST /confirm → "📧 Email berhasil disimpan… menunggu proses invite"
invite:change → reset waiting
Semua api.getDelivery(orderId, telegramId) + POST customer-data verify String(order.user.telegramId)===telegramId → 403 Tidak dapat mengakses pesanan ini. (user A tidak bisa akses order B) — berlaku untuk callback & HTTP API
5. Admin Flow
apps/backend/src/modules/invite/routes.ts inviteAdminRoutes:

GET /api/admin/invites?status=PENDING_INVITE → list + maskedEmail (via maskEmail), GET /api/admin/orders/:id/delivery
POST /api/admin/orders/:id/invite/reveal-email (RBAC SUPER_ADMIN|ADMIN|OPERATOR) → revealCustomerEmail + audit ADMIN_REVEALED_CUSTOMER_EMAIL payload {masked} (tidak pernah plaintext di log)
POST /mark-sent → markInviteSent (idempotent: jika INVITE_SENT|COMPLETED return noop), transaction: delivery INVITE_SENT+completedAt + order COMPLETED + subscription ACTIVE (atau skip jika sudah ada), notifikasi ✅ YouTube Premium berhasil diproses. Berakhir: …
POST /fail → FAILED + notifikasi ⚠️ Invite belum berhasil…, POST /retry → hanya dari FAILED → PENDING_INVITE (tidak membuat order baru)
apps/admin/src/main.tsx: Nav +YouTube Invites, Dashboard Pending Invite: N (dari GET /api/admin/dashboard baru pendingInvite/failedInvite) → link ke /admin/invites, halaman Invites dengan filter, kartu ORD-… status, Email: u***@gmail.com [Show Email], buttons [✅ Tandai Invite Terkirim][❌ Gagal][🔄 Retry]
6. Subscription
Tetap engine computeEndDate/start + durationDays & computeRenewedEndDate(oldEnd>now ? oldEnd : now). markInviteSent buat subscription start=now, end=computeEndDate(now, duration) (contoh 29/09 7 hari → 06/10, renew active oldEnd 06/10 +7 → 13/10). Tidak buat subscription baru jika renewal (existing orderId unique). Bot2 tidak diubah — hanya tampil Mulai/Berakhir/Status + reminder dedupKey subscription:{id}:H-3/H-2/H-1/EXPIRED.

7. Worker Changes
apps/worker/src/jobs/index.ts runInviteQueueJob() + runAllJobs include it. apps/backend/src/jobs/scheduler.ts runInviteQueueJob interval 15m: PENDING_INVITE >30m → notifikasi admin invite.pending_admin, FAILED → notifikasi user ⚠️ Invite belum berhasil… (dedup hari-an) — TIDAK melakukan login/invite Google apapun.

8. API Changes
POST /api/orders/:id/customer-data { googleEmail|email, telegramId } → PENDING_INVITE
GET  /api/orders/:id/delivery?telegramId=… → {delivery, maskedEmail, orderStatus}
POST /api/orders/:id/customer-data/confirm {telegramId}
GET  /api/admin/invites?status=&limit=
GET  /api/admin/orders/:id/delivery
POST /api/admin/orders/:id/invite/reveal-email (RBAC ADMIN/OPERATOR, audit masked)
POST /api/admin/orders/:id/invite/mark-sent
POST /api/admin/orders/:id/invite/fail {reason}
POST /api/admin/orders/:id/invite/retry
GET  /api/admin/dashboard → +pendingInvite, +failedInvite
Tidak ada endpoint duplicate (reuse /api/orders/:id existing).

9. Tests
tests/invite.test.ts 26 tests (18 requirement coverage) + unit.test.ts 24 + inventory.test.ts 4:

Test Files 3 passed | Tests 54 passed
Ph24 1: schema DeliveryMethod INVITE & default STOCK — PASS
Ph24 2: STOCK tidak minta email — PASS
Ph24 3: PAID→WAITING_CUSTOMER_DATA + WAITING→PROCESSING — PASS
Ph24 4: invalid email ditolak — PASS
Ph24 5: trim/lowercase/≤320 + user@gmail.com — PASS
Ph24 6: encrypt tidak plaintext + roundtrip — PASS
Ph24 7: ownership 111≠222 rejected — PASS
Ph24 8: DeliveryStatus PENDING_INVITE — PASS
Ph24 9: OrderDelivery fields — PASS
Ph24 10: subscription 29/09→06/10 — PASS
Ph24 11: duplicate mark-sent idempotent — PASS
Ph24 12: FAILED→retry only — PASS
Ph24 13: retry tidak buat order baru — PASS
Ph24 14: renewal ACTIVE extends oldEnd — PASS
Ph24 15: maskEmail — PASS
Ph24 16: audit sanitize email — PASS (SECRET_KEYS + ADMIN_REVEALED…)
Ph24 17: RBAC reveal/mark — PASS
Ph24 18: tidak ada QRIS/Google password/access_token — PASS (MANUAL)
npm test PASS (7.6s).

10. Typecheck
npm run typecheck → tsc --noEmit EXIT:0 (fix PkgT typing untuk prisma.productPackage).

11. Build
npm run build --workspaces:

admin vite 6.4.3 → 272.20 kB (gzip 85.91kB) PASS
backend tsc PASS
bot1 tsc PASS
bot2 tsc PASS
worker tsc PASS
12. Remaining Limitations
docker compose config → DOCKER: NOT AVAILABLE di host ini; compose file valid (5 services), tidak diklaim.
DATABASE_URL Postgres tidak ada → prisma migrate dev & db:seed NOT RUN (hanya validate/generate). Perlu npx prisma migrate dev --name phase24-invite di env dengan DB.
Payment tetap MockPaymentProvider (MOCK-PAYMENT:…:NOT_A_REAL_QRIS), tidak ada QRIS.
Invite MANUAL — tidak ada akses akun Google, tidak ada password/OTP/cookie/session/access_token, tidak ada automation (dijaga di invite.ts comment + test).
bot-jualan/.env berisi change-me-* placeholder (development).
Security audit: customerDataEncrypted AES-256-GCM, maskEmail u***r@gmail.com, RBAC reveal/mark, ownership telegramId, audit.ts SECRET_KEYS include email/googleEmail/customerDataEncrypted, auditLog payload sanitize ***, tidak ada log plaintext.

Next: git add + commit ("feat: Phase24 YouTube INVITE manual …") saat DB tersedia → migrate → seed → curl /api/health → Bot1 /start → checkout YT → pay mock POST /api/payments/:id/check?mockPaid=1 → bot minta email → admin /admin/invites mark-sent smoke.
