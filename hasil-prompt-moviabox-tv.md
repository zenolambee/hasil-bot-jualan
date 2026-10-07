









Porting ke APK Android TV native selesai.

Yang dikerjakan:

Home: sidebar + hero cinematic + rel poster (Lanjut Menonton dengan progress, Jelajahi, Film Populer, Serial Populer)
Search overlay, Detail (backdrop + poster + info + server/episode), Player dengan panel kualitas & server
Navigasi remote TV native (focus jelas, no layout shift, focus restore)
Semua logic dipertahankan: StreamResolver, token, fallback, cache, Media3 — tanpa WebView
Verifikasi: assembleDebug BUILD SUCCESSFUL (APK 9.9 MB), 18 unit test PASS, 0 gagal.

Commit: 8afb39b dengan message persis "android: port cinematic tvplay UI to native Android TV" (terpaksa jadi 2 commit karena batas teknis ukuran payload — commit pertama yang membawa message persis itu; commit kedua resources + README).

Satu catatan jujur: testing di perangkat TV fisik tidak bisa dilakukan dari sini — checklist manual A–O (D-pad di hardware asli, dsb.) perlu kamu coba langsung di STB/TV. Kalau ada yang perlu diperbaiki setelah dicoba, kabari aku.





Redesign TVPlay selesai dan sudah di-commit.

Yang dikerjakan (murni UI, tanpa ubah logic):

Sidebar navigasi kiri (Beranda, Film, Serial, Daftar Saya, Cari, Pengaturan)
Hero fullscreen sinematik + tombol Putar / Daftar Saya
Rel horizontal: Lanjut Menonton (dengan progress), Film Populer, Serial Populer
Search overlay modern, detail film (backdrop besar, poster kiri info kanan), panel kualitas & server ramah TV
Focus D-pad jelas (outline biru + scale ringan, tanpa layout bergeser)
Semua behavior lama dipertahankan: API, token, cache, fallback, player logic, localStorage
Verifikasi: 39/39 acceptance test PASS, alur manual (Settings → connect → Search → Detail → Play → Quality → Server → Back) PASS, 0 JS error.

Commit 9f9485a dengan message persis "tvplay: redesign cinematic Netflix-style TV UI" (file tvplay/index.html + catatan redesign di tvplay/README.md).

Silakan dicek di repo. Kalau ada detail tampilan yang mau disesuaikan, tinggal bilang.




Ringkasan
Audit: seluruh source Android tetap di app/ (tidak diubah), tvplay/index.html tetap behavioral reference, API_SPEC.md tetap contract. Hanya tvplay/index.html + tvplay/README.md yang diubah — tanpa fitur di luar permintaan.

Yang ditambahkan/diubah di tvplay/index.html
Settings: Base URL, API Key, Tes Koneksi (status ● API Connected / ● API Offline), Simpan & Sambung, Hapus Konfigurasi; Settings jadi halaman pertama bila belum terkonfigurasi.
Persist localStorage tvplay_base / tvplay_key; auto-isi field & auto-connect saat refresh (tidak perlu isi ulang).
REST API TEST: Test /health (tanpa key; tampil status/version/response time), Test Search, Test Detail — stream tidak diuji di sini.
/health tanpa X-API-Key; search/detail/stream pakai X-API-Key.
Play-only resolve (katalog/search/detail/episode tidak memanggil /api/stream).
Fallback 502 maks 2, 429 backoff 20 s retry sekali, 400/401/404 tanpa retry.
Tipe stream dari bentuk URL (bukan label kualitas): master/media → HLS, direct → progressive.
Token expiry refresh −30 s; 401 media → resolve ulang; posisi dipertahankan saat ganti server/kualitas.
Cache search 10 m / detail 5 m (sessionStorage); stream tidak di-cache.
Pesan error ramah & tersanitasi (tanpa key/token/fileUrl/upstream/stack).
Verifikasi (bukan klaim)
Mock API + Chrome headless (Playwright) menjalankan 39/39 PASS, mencakup semua 20 acceptance criteria (persist config, health tanpa key, key di search/detail, play-only, mapping HLS/progressive, token expiry, 401 re-resolve, 502 fallback maks 2, 429 backoff 20 s, switch posisi, cache, tanpa secret di UI, 0 JS error). Hasil ditulis di commit message + tvplay/README.md.

Commit: ddfe681 — tvplay: REST API web tester + persist config (localStorage)


Audit hasil
Seluruh source Android ada di app/ (25 file .kt main + 4 kelas uji). tvplay/index.html tetap behavioral reference, API_SPEC.md tetap API contract (keduanya utuh).
Tidak ada TODO/FIXME untuk fitur wajib. Tidak ada laporan/prompt yang disimpan sebagai "implementasi".
Root cause yang diperbaiki (bukan workaround)
themes.xml: Theme.MaterialComponents.Dark.NoActionBar tidak ada di Material 1.12.0 → diganti parent resmi Theme.MaterialComponents.NoActionBar (base-nya dark).
CacheManager.kt: remove()/clear() mengekspos tipe private Entry → ubah ekspresi synchronized menjadi block body.
SettingsViewModel/Activity: parameter apiKey mati pada testConnection (/health tanpa auth) → dihapus dari signature & call site.
PlayerActivity.kt: tambah suppress OVERRIDE_DEPRECATION (KT-47902).
Hasil verifikasi (clean build)
Item	Hasil
./gradlew assembleDebug	PASS
./gradlew testDebugUnitTest	PASS (18 test, 0 gagal)
Compile error / warning	tidak ada
APK	app/build/outputs/apk/debug/app-debug.apk — 10.236.215 byte (± 9,76 MB)
Test: StreamResolverTest 6, StreamResolverExpiryTest 4, CacheManagerTest 5, UrlToolsTest 3 — fallback/token-expiry/cache/resolver terverifikasi.

BuildConfig: MOVIE_API_URL/MOVIE_API_KEY tergenerate (injection via moviebox.properties/env/-P + Settings). HTTP client OkHttp, Media3 HLS/Progressive, manifest — semua lolos kompilasi.

Commit
230b95b — kode terverifikasi (build+test PASS)
7e819bf — README mencatat hasil build, nama/ukuran APK, hasil test, SHA
Working tree bersih; local.properties (path SDK temp) ter-gitignore.
Acceptance: BUILD = PASS · UNIT TEST = PASS · APK EXISTS = PASS · NO COMPILE ERROR · NO TODO (fitur wajib).

Berhenti di sini — tidak menambah fitur. Catatan: build lokal tak dapat diverifikasi tanpa toolchain; di mesin ber-SDK cukup .\gradlew.bat assembleDebug testDebugUnitTest.














Audit
Read tvplay/index.html (behavioral source of truth) and API_SPEC.md in full, and audited the existing app/. I replaced the previous loosely-organized code (core/, data/, flat ui/) with a clean modular structure, keeping only reusable behavior/logic.

What was built (Android TV, Kotlin + Media3)
Architecture (app/src/main/java/com/moviebox/tv/): api/ (ApiClient/OkHttp, ApiConfig, ApiException) · model/ · cache/ · repository/ (MovieRepository, StreamResolver) · player/PlayerManager · ui/{common,settings,catalog,detail,player} · util/UrlTools.

Key behaviors ported from tvplay:

Play-only resolve: /api/stream only fires in PlayerActivity (opened when user presses Putar); catalog/detail never call it.
Config, not hard-coded: MOVIE_API_URL/MOVIE_API_KEY via moviebox.properties/env → BuildConfig, plus a Settings screen (Base URL, API Key, Tes Koneksi, Simpan, status).
Token lifecycle: stores token/expires; schedules refresh 30 s before expiry; on player error (e.g. media 401) re-resolves — never reuses the old play_url.
Media3 mapping: master/media → HlsMediaSource; direct → ProgressiveMediaSource (via OkHttpDataSource, Range preserved). Variant type determined from URL/playlist_type, not quality name.
Fallback: p2p→turbovip→cast→hydrax, max 2 attempts; 429 → 20 s backoff (once); 400/401/404 permanent; explicit server → no auto-fallback.
Cache: search 10 min, detail 5 min (in-memory TTL).
Position-preserving switches for quality and server.
TV UX: thick focus indicators on cards/pills/episodes/menu, D-pad navigation, player auto-hide overlay (4 s), OK = pause/play, LEFT/RIGHT = seek ±10 s, back stack player→detail→catalog.
Sanitized errors: only friendly userMessage; API key, token, fileUrl, upstream hosts never shown/logged.
Tests (app/src/test): StreamResolverTest (fallback max-2, 429 backoff, permanent errors, explicit server), StreamResolverExpiryTest, CacheManagerTest, UrlToolsTest. Added org.json test dependency for real JSON parsing.

README.md updated with build, config, architecture, Media3 mapping, token lifecycle, fallback, security, and a testing checklist.

One caveat (honest)
I could not run the Gradle build — this machine has no JDK, no Android SDK, and no network to the API (verified). All XML is well-formed, all resource references (colors/strings/drawables/ids) were cross-checked, and stale old-package references are gone. Run:

$env:ANDROID_HOME = "$env:LOCALAPPDATA\Android\Sdk"
.\gradlew.bat assembleDebug ; .\gradlew.bat testDebugUnitTest
Paste any compiler output and I'll fix it immediately. No TODOs remain in the required features.
