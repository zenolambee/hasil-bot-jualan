



























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
