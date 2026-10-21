# Analisis Reverse Engineering — CapCut 19.7.0

> Analisis **statis** untuk tujuan edukasi/riset. Aplikasi tidak dijalankan.
> Tanggal: 2026-10-21.

## 1. Identitas sampel

| Atribut | Nilai |
|---|---|
| Nama | CapCut - Video Editor |
| Package | `com.lemon.lvoverseas` |
| Versi | 19.7.0 (versionCode 19700200) |
| Ukuran | 320.939.435 byte (320,94 MB) |
| MD5 | `3ac0b7b7ee7edca247447ab7fc247484` |
| SHA256 | `f9c40e89ecdf306ba649f191f6181c67901a4967dd85299f6925906feb2c59fd` |
| Sumber | Aptoide (store "bjwawesome") — rank TRUSTED |
| minSdk / targetSdk | 23 / 36 |

### Keaslian — TERVERIFIKASI ASLI
Sertifikat penandatangan:
- **Owner/Issuer:** `CN=lemon, O=bytedance` (self-signed, 2019–2044)
- **SHA256 cert:** `6F:33:FD:44:6F:6E:83:A7:67:D4:FF:F9:C1:9C:EC:F7:60:59:81:4E:A6:A7:EA:71:DF:1D:0D:EF:2D:75:DB:31`
- Signature: SHA256withRSA

"lemon" adalah codename internal CapCut; "bytedance" penerbit resminya. Aplikasi
yang di-repackage/mod akan ditandatangani kunci berbeda, sehingga sertifikat ini
memastikan APK **belum dimodifikasi** — ini build CapCut resmi.

## 2. Struktur

| Komponen | Jumlah |
|---|---|
| Total entri | 27.976 |
| DEX (kode) | 39 (multidex) |
| Native lib (.so) | 345 |
| Arsitektur | arm64-v8a, armeabi-v7a |
| Assets | 1.170 |
| Uncompressed | ~850 MB |

Kode utama: `com.vega.*` (nama internal editor "Vega"), `com.xt.*` (retouch/edit).

## 3. Permission (40 total)

**Sensitif:** `CAMERA`, `RECORD_AUDIO`, `READ/WRITE_CALENDAR`,
`READ_MEDIA_IMAGES/VIDEO/AUDIO`, `ACCESS_MEDIA_LOCATION`,
`READ/WRITE_EXTERNAL_STORAGE`.

Wajar untuk editor video (kamera, mikrofon, galeri). Yang perlu dicatat:
`READ/WRITE_CALENDAR` kurang lazim untuk editor video — kandidat untuk ditelusuri.

**Custom/vendor:** `com.android.vending.BILLING` (pembelian in-app),
`com.applovin.*` (SDK iklan), permission launcher Oppo/Samsung (shortcut/widget).

## 4. Komponen (attack surface)

| Tipe | Total | Exported |
|---|---|---|
| activity | 493 | 22 |
| service | 51 | 9 |
| receiver | 30 | 5 |
| provider | 21 | 4 |

Permukaan besar tapi wajar untuk aplikasi sebesar ini. 22 activity + 4 provider
exported adalah kandidat utama audit IPC/deep-link.

## 5. Keamanan jaringan (hardened)

`res/xml/z.xml`:
- `cleartextTrafficPermitted="false"` (base & debug) — tidak ada HTTP polos.
- Trust anchor **hanya CA sistem** (bukan CA user), termasuk di debug-overrides.

Artinya aplikasi **menolak MITM proxy biasa** (sertifikat user tidak dipercaya).
Untuk analisis dinamis perlu patch network-security-config atau Frida.

## 6. SDK pihak ketiga

- **ByteDance core** (`com.ss.android`, `com.bytedance.*`) — analitik AppLog,
  jaringan, monitoring.
- **Pangle** (`com.bytedance.sdk.openadsdk`) — jaringan iklan ByteDance.
- **AppLovin MAX** (`com.applovin.mediation`) — mediasi iklan.
- **Google** — Play Services, Firebase, AdMob/DoubleClick.

## 7. Endpoint jaringan (dari string dalam DEX; 210 host unik)

**Telemetri/Logging ByteDance:**
`mon.isnssdk.com`, `i.isnssdk.com`, `log.isnssdk.com`, `log.sgsnssdk.com`,
`log.byteoversea.com`, `mon.byteoversea.com`, `log.tiktokv.com`

**API CapCut:**
`www.capcut.com`, `editor-api.capcutapi.com`, `feed-api.capcutapi.com`,
`aivideo-normal.capcutapi.com`, `feed-api-va.hypic.com`, `www.capcut.net`

**Iklan:**
`developers.applovin.com` + banyak `*.applovin.com`, `googleads.g.doubleclick.net`,
`pagead2.googlesyndication.com`, `lf-static.tiktokpangle-cdn-us.com`

**CDN:** `sf16-sg.tiktokcdn.com`, `*.ibytedtos.com`, `*.ibyteimg.com`

Pola khas: endpoint produk sendiri + telemetri ByteDance intensif + beberapa
jaringan iklan. (CapCut pakai domain `isnssdk`/`byteoversea` yang sama dengan
ekosistem TikTok/ByteDance.)

## 8. Catatan & langkah lanjut

Belum dilakukan (butuh waktu/arahan):
- Decompile penuh via `jadx` (39 DEX, besar) untuk logika fitur spesifik.
- Audit 22 activity + 4 provider exported (deep-link, IPC).
- Telusuri alasan permission CALENDAR.
- Analisis native `.so` (libcutsame, dll.) jika logika inti ada di native.
- Uji payload yang dikirim ke endpoint telemetri (butuh analisis dinamis + bypass pinning/NSC).

---
*File sampel ada di scratchpad sesi (tidak di-commit — 320 MB).*
