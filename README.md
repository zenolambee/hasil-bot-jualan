PHASE 30A: PASS

Files changed

`apps/bot1/src/handlers/menu.ts`
`apps/bot1/src/keyboards/main.ts`
`tests/phase30a.test.ts`
Katalog Bot 1 kini menampilkan satu produk per baris dengan harga paket termurah yang tersedia dan status inventory aktual: 🟢 … (jumlah) atau 🔴 … (Out). Stok menghitung inventory paket serta inventory generik yang memang digunakan alur reserve. Nama panjang dipendekkan hanya pada tombol; nama database tidak diubah.

Pagination memakai tombol ◀️ [📄 halaman/total] ▶️; tombol arah yang tidak relevan disembunyikan di halaman pertama/terakhir dan indikator tidak muncul jika hanya satu halaman. Detail produk tetap memakai handler package dan tombol beli yang sudah ada. Categories tidak ditampilkan karena Bot 1 belum memiliki category flow; tombol Main Menu memakai callback yang sudah ada.

Regression dan pemeriksaan

Tests: 187 passed / 0 failed; Phase 24–29 tests tetap lulus.
Typecheck: PASS.
Lint: PASS.
Build: PASS untuk admin, backend, Bot 1, Bot 2, dan worker.
Tidak ada perubahan pada order, payment, delivery, subscription, atau integrasi Bot 2.
Tidak ada regression terdeteksi di pemeriksaan otomatis.
Pengujian langsung di Telegram dan verifikasi stok aktual tidak dijalankan karena memerlukan PostgreSQL serta sesi bot aktif. PostgreSQL/Docker integration tidak diuji pada perubahan ini.
