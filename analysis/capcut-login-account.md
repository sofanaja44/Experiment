# CapCut 19.7.0 — Analisis Fitur Login & Akun

> Dari kode hasil decompile `jadx` (paket `com.lemon.account`, 337 kelas).
> Analisis statis. Nama field sebagian ter-obfuscate (a/b/c, f56692a) tapi
> struktur & alur terbaca jelas.

## Ringkasan

Login CapCut **bukan sistem buatan sendiri** — ini pembungkus (facade) di atas
**SDK akun ByteDance "BDAccount" + sistem "Passport"**, infrastruktur yang sama
dipakai TikTok. Lapisan `com.lemon.account` hanya menyediakan API tingkat-aplikasi
dan UI; autentikasi, token, dan sesi dikelola SDK ByteDance.

## Komponen inti

| Lapisan | Teknologi | Bukti |
|---|---|---|
| Auth engine | `com.bytedance.sdk.account.api.IBDAccount` (BDAccount SDK) | import di `AccountFacade.java` |
| Backend auth | **Passport** @ `passport-api.capcutapi.com` | string DEX |
| HTTP client | Fork Retrofit ByteDance `com.bytedance.retrofit2` | `AccountService.java:1-6` |
| Penyimpanan | **Keva** (KV-store ala MMKV) via AOP hook `SharedPreferencesLancet` → `KevaSpAopHook` | `VeryImportantConfig.java:244-317` |
| Monitor privasi | **Helios** `com.bytedance.helios.statichook` (hook akses API sensitif) | import di `AccountFacade.java` |
| Routing/deep-link | `com.bytedance.router.SmartRouter` | import di `AccountFacade.java` |
| Risk control | **BdTuring** (`BdTuringServiceImp`) — CAPTCHA/verifikasi ByteDance | kelas di paket akun |

### Di mana sesi/token disimpan
Semua panggilan `getSharedPreferences` di-redirect ke **Keva** lewat bytecode-AOP
(`SharedPreferencesLancet`/`KevaSpAopHook`), jadi **bukan file XML
SharedPreferences biasa** — tersimpan di format Keva (lebih sulit diinspeksi
tanpa root). Session token (`x-tt-token`/`sessionid`) dipegang BDAccount SDK.
Contoh data lain yang disimpan: `region_info` → `store_region`
(`VeryImportantConfig.java:642-644`).

`AccountFacade.isLogin()` hanya meneruskan ke `IBDAccount.isLogin()`; data user
(`getUserId/getUserName/getUserEmail/getUserMobile`) diambil dari SDK yang sama.

## Metode login yang didukung

**Native (Passport):**
- Email — `/passport/email/register_verify/`, `/email/bind_and_verify/`, `/email/change_password/`
- SMS/No. HP — `/passport/mobile/sms_login_only/`, `/account_lookup/mobile/`
- Lookup perangkat — `/passport/account_lookup/device/`

**Pihak ketiga / OAuth** (subpaket: `ttbind`, `youtube`, `bind`):
- **TikTok** (terbanyak, 25 ref) — quick-login `/passport/can_aweme_quick_login/`,
  binding via `ttbind/TiktokBindProcess`, `CreateLinkSessionStep` ("Aweme" = Douyin/TikTok)
- **Google / YouTube** (25) — `youtube/YoutubeAuthService.requestAccessTokenByGMS`,
  OAuth `authorization_code`
- **Facebook** (18) — `com.facebook.AccessTokenSource` (FB Login SDK)
- **Apple** (2) — Sign in with Apple
- Backend tautan: `/lv/v1/user/client/bind_third_party`, `/lv/v1/user/third_party_info`

**Multi-akun:** subpaket `multilogin` (`SwitchAccountConfigManager`) — ganti akun
tanpa logout, konfigurasi disimpan di SharedPreferences/Keva.

## Kepatuhan & kontrol usia (sebelum/sesudah login)

Login dipagari pengecekan usia/kepatuhan (CoPPA-style):
- `/lv/v1/user/age_gate` — "age door" (`AccountService.isPassAgeDoor()`, `AccountService.java:11-13`)
- Subpaket `agegate`, kelas `ComplianceAgeCheckHelper`, `BirthdayActivity`,
  `ChildCheckData/Result`
- `/lv/v1/sc/compliance_popup/submit_consent` — persetujuan popup kepatuhan
- `/lv/v1/punish/get_punish_status` + `BanFunctionHelper/BanNotifyManager/BanReason`
  — status hukuman/pemblokiran akun

## Endpoint backend CapCut terkait akun (`/lv/v1/...`)

`/user/age_gate`, `/user/generate_unique_id`, `/user/third_party_info`,
`/user/client/bind_third_party`, `/homepage/profile`, `/authority/get_creator_authority`,
`/punish/get_punish_status`, `/sc/compliance_popup/submit_consent`

## Catatan untuk analisis dinamis

- Trafik di-HTTPS, NSC hanya percaya CA sistem → **MITM proxy biasa gagal**.
  Butuh patch network-security-config + kemungkinan bypass BdTuring/signing.
- Token di Keva (bukan XML) → inspeksi butuh root atau hook runtime (Frida).
- Request Passport ditandatangani SDK (parameter `X-Argus`/`X-Gorgon`/`X-Ladon`
  khas ByteDance) — perlu ditelusuri di native lib bila ingin replay request.

## Keterbatasan

- `BdTuringServiceImp.java` belum ter-decompile saat analisis (job jadx OOM di
  ~88%). Logika pasti signing request ada di native `.so` (di luar dex).
- Nama field token di `AccessTokenData` ter-obfuscate (4 String: a/b/c/d).
