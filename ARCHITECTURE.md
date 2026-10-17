# ARCHITECTURE.md — Kontrak Teknis

> Ini kontrak antar-agent. Server dikerjakan **dua agent** (Claude Code dan Codex),
> frontend oleh Cline, jadi **bentuk request/response dan fungsi di §12 adalah
> hukum**. Kalau butuh field atau fungsi baru, ubah file ini dulu lewat PR
> terpisah (oleh owner), baru koding.

---

## 1. Struktur Folder (kepemilikan per agent)

```
monsaham-project/
├── server/
│   ├── config/
│   │   ├── config.js           ← Claude Code
│   │   ├── constants.js        ← OWNER: kamu (manusia). Agent hanya membaca
│   │   └── marketHolidays.js   ← OWNER: kamu (manusia). Diperbarui tiap tahun
│   │
│   ├── models/                 ← Claude Code (Sequelize models)
│   ├── migrations/             ← Claude Code
│   ├── seeders/                ← Claude Code
│   ├── telegram/               ← Claude Code
│   │   ├── client.js           (satu-satunya file yang memanggil Bot API)
│   │   ├── router.js           (route update: command / text / callback)
│   │   └── formatter.js        (menyusun teks balasan + inline keyboard)
│   ├── services/               ← Claude Code
│   │   ├── entitlementService.js   (satu-satunya penentu tier)
│   │   ├── onboardingService.js
│   │   ├── alertService.js         (buat / batal / aktifkan lagi / tandai kena)
│   │   ├── stockService.js         (cari ticker di tabel Stocks)
│   │   └── subscriptionService.js
│   ├── controllers/            ← Claude Code
│   ├── routes/                 ← Claude Code
│   ├── middlewares/            ← Claude Code (authentication, entitlement, telegramAuth, errorHandler)
│   ├── helpers/                ← Claude Code
│   │   ├── dateHelper.js
│   │   ├── jwt.js, bcrypt.js
│   │   ├── price.js            (parsePrice, formatRupiah, tick size, bps — fungsi murni)
│   │   ├── alertParser.js      (regex: teks → perintah alert, SEMUA tier)
│   │   └── stateStore.js       (Map in-memory + TTL)
│   │
│   ├── market/                 ← Codex
│   │   ├── marketClient.js     (satu-satunya file yang memanggil API data saham)
│   │   ├── quoteService.js     (cache harga in-memory + sanity check)
│   │   ├── historyService.js   (isi & perbarui PriceDailies)
│   │   ├── metricsService.js   (hitung angka dari PriceDailies, TANPA AI)
│   │   ├── marketCalendar.js   (hari bursa & jam sesi — fungsi murni)
│   │   └── alertRules.js       (apakah alert kena? — fungsi murni)
│   ├── ai/                     ← Codex
│   │   ├── aiClient.js         (satu-satunya file yang memanggil API AI)
│   │   ├── contextService.js   (metrics + cache Analyses + panggil AI)
│   │   └── complianceFilter.js (buang output AI yang terdengar seperti saran)
│   ├── jobs/                   ← Codex
│   │   ├── alertCheckJob.js    (tiap 5 menit selama sesi)
│   │   ├── eodJob.js           (setelah bursa tutup: harga harian + ringkasan)
│   │   ├── expiryJob.js        (00:05: subscription & trial)
│   │   └── stockSyncJob.js     (mingguan: daftar ticker)
│   ├── bin/
│   │   ├── www.js              ← Claude Code
│   │   └── cron.js             ← Codex (proses terpisah, bukan di dalam web server)
│   │
│   ├── __tests__/              (lihat TESTING.md §4 untuk pembagian per agent)
│   ├── app.js                  ← Claude Code
│   ├── jest.config.js          ← OWNER: kamu (manusia). Ambang coverage
│   └── package.json            ← Claude Code (perubahan dependency = izin owner)
│
├── client/                     ← OWNER: Cline
│   ├── src/
│   │   ├── components/
│   │   ├── pages/              (Login, Overview, Alerts, StockDetail, Upgrade)
│   │   ├── services/api.js
│   │   ├── context/
│   │   ├── App.jsx
│   │   └── main.jsx
│   └── vite.config.js
│
├── .github/workflows/          ← OWNER: Antigravity
├── docs/                       ← OWNER: Antigravity
├── SPEC.md / PRD.md / ARCHITECTURE.md / ERD.md / STYLE.md / TESTING.md /
│   AGENTS.md / WORKFLOW.md     ← OWNER: kamu (manusia)
└── README.md                   ← OWNER: Antigravity
```

**Aturan:** agent hanya boleh mengubah file di folder miliknya. **Claude Code dan
Codex sama-sama di `server/`, tapi foldernya tidak tumpang tindih.** Kalau
butuh perubahan di file milik agent lain → buka issue, jangan edit sendiri.
Memanggil fungsi milik agent lain **boleh**, asal fungsinya tercantum di §12.

**Kenapa pembagiannya begini:** Claude Code memegang semua yang menerima input
dari luar (Telegram, REST, pembayaran) dan semua yang menulis data user. Codex
memegang semua yang berbicara ke pihak ketiga yang berbayar (provider data,
provider AI) dan semua yang berjalan terjadwal. Batasnya adalah fungsi di §12.

## 2. Schema Database

Relasi antar tabel, constraint, dan index ada di `ERD.md`. Di sini kolomnya.

### Users
| kolom | tipe | keterangan |
|---|---|---|
| id | integer PK | |
| telegramId | bigint | unique, not null — identitas utama |
| username | string | nullable, `@username` Telegram |
| name | string | dari first_name + last_name profil Telegram |
| email | string | nullable, unique, dipakai kalau login dashboard pakai password |
| password | string | hashed bcrypt, nullable |
| trialStartedAt | date | diisi sekali saat `/start` pertama |
| trialEndsAt | date | `trialStartedAt + 7 hari`, **tidak pernah diubah** |
| timezone | string | default `Asia/Jakarta` |
| dailySummaryEnabled | boolean | default `true`, diubah lewat `/ringkasan on\|off` |

> **Tidak ada kolom `plan`.** Tier dihitung, lihat `SPEC.md §4`.

### Subscriptions
| kolom | tipe | keterangan |
|---|---|---|
| id | integer PK | |
| UserId | integer FK | not null |
| status | string | enum: `pending_payment` / `active` / `expired` / `cancelled` |
| price | integer | rupiah penuh, snapshot harga saat beli |
| paymentRef | string | nullable, referensi dari payment gateway, unique |
| startedAt | date | |
| expiresAt | date | `startedAt + 30 hari` |

### Stocks (daftar ticker, diisi `stockSyncJob`)
| kolom | tipe | keterangan |
|---|---|---|
| id | integer PK | |
| symbol | string | unique, not null. Saham: `^[A-Z]{4}$`. Indeks: `IHSG` |
| name | string | nama emiten, mis. "Bank Central Asia Tbk" |
| isIndex | boolean | default `false`. `true` hanya untuk `IHSG` (dipakai sebagai pembanding konteks, tidak bisa dipasang alert) |
| isActive | boolean | default `true`. `false` kalau delisting/suspensi → alert-nya dilewati |
| lastSyncedAt | date | |

### Alerts
| kolom | tipe | keterangan |
|---|---|---|
| id | integer PK | |
| UserId | integer FK | not null |
| StockId | integer FK | not null |
| type | string | enum: `price_below` / `price_above` / `pct_drop` / `pct_rise` / `below_avg_30d` |
| targetPrice | integer | rupiah. **Null hanya untuk `below_avg_30d`**. Untuk `pct_*` dihitung saat dibuat (SPEC §8) |
| targetPctBps | integer | basis poin. **Terisi hanya untuk `pct_*`**, untuk ditampilkan |
| basePrice | integer | harga pasar saat alert dibuat, not null |
| basePriceAsOf | date | jam data `basePrice` |
| status | string | enum: `active` / `triggered` / `cancelled` |
| triggeredAt | date | nullable |
| triggeredPrice | integer | nullable, harga yang membuat alert kena |
| triggeredPriceAsOf | date | nullable, jam data harga tersebut |
| cancelledAt | date | nullable |
| source | string | enum: `text` / `command` / `callback` / `dashboard` |
| rawMessage | text | pesan asli user — untuk audit parser. **Tidak boleh masuk log** |
| reactivatedFromId | integer FK | nullable, rujuk alert lama saat [Aktifkan lagi] / [Geser target] |
| createdAt | date | dipakai untuk urutan "paling lama dibuat" (batas free) |

### PriceDailies (harga penutupan harian)
| kolom | tipe | keterangan |
|---|---|---|
| id | integer PK | |
| StockId | integer FK | not null |
| tradeDate | dateonly | tanggal bursa `YYYY-MM-DD` (WIB) |
| open, high, low, close | integer | rupiah |
| volume | bigint | lembar |

Hanya diisi untuk saham yang **pernah punya alert aktif**, plus `IHSG`. Tidak
menyimpan seluruh bursa.

### Analyses (cache konteks AI, biar tidak panggil model berulang)
| kolom | tipe | keterangan |
|---|---|---|
| id | integer PK | |
| StockId | integer FK | not null |
| scope | string | enum: `alert_context` (v1.0 hanya ini) |
| tradeDate | dateonly | tanggal bursa. **Satu konteks per saham per hari** |
| inputHash | string | hash dari metrics — untuk audit |
| metrics | jsonb | angka mentah hasil `metricsService.forContext` |
| insight | text | narasi AI yang **sudah lolos** `complianceFilter` |
| tokensUsed | integer | untuk kontrol biaya |

Kalau output AI ditolak `complianceFilter`, baris tetap disimpan dengan
`insight = null` supaya hari itu AI tidak dipanggil ulang untuk saham yang sama.

### Cache harga — BUKAN tabel
Harga intraday **tidak disimpan di DB**. Disimpan di `Map` in-memory di
`market/quoteService.js`, TTL `QUOTE_CACHE_TTL_SECONDS` (300 detik).

### State input menggantung — BUKAN tabel
`helpers/stateStore.js`, TTL `CONFIRMATION_TTL_SECONDS` (300 detik). Dipakai
untuk [Geser target]: bot menunggu user mengetik harga baru.
```js
stateStore.set(telegramId, { action: 'shift_target', alertId: 12 }, 300);
stateStore.get(telegramId);   // null kalau sudah lewat TTL
stateStore.del(telegramId);
```

### update_id yang sudah diproses — BUKAN tabel
`Set` in-memory di `telegram/router.js` (sama seperti Mon Mon). Cukup untuk
satu instance di v1.0.

## 3. Format Response Standar

Sukses:
```json
{ "message": "Alert created", "data": { } }
```

List dengan pagination:
```json
{
  "data": [],
  "meta": { "page": 1, "limit": 10, "totalItems": 57, "totalPages": 6 }
}
```

Error (SEMUA error lewat `errorHandler` middleware):
```json
{ "message": "Target Rp 3.003 tidak sesuai fraksi harga" }
```

Error karena tier (field tambahan supaya frontend bisa tampilkan upsell):
```json
{ "message": "Alert persen hanya untuk Member", "requiredTier": "member" }
```

Kode status: `200`, `201`, `400` (validasi / business rule), `401`, `403` (tier
tidak berwenang **atau** batas alert tercapai), `404`, `429` (OTP), `500`,
`503` (provider data tidak tersedia — `name: 'MarketUnavailable'`).

## 4. Daftar Endpoint

### 4a. Webhook (dipanggil Telegram & payment gateway, bukan client)
| Method | Path | Keterangan |
|---|---|---|
| POST | `/webhook/telegram` | Inbound update. Divalidasi `X-Telegram-Bot-Api-Secret-Token` |
| POST | `/webhook/payment` | Notifikasi pembayaran. Divalidasi signature gateway |
| GET | `/health` | Untuk uptime monitor & smoke test deploy |

### 4b. REST API Dashboard
| Method | Path | Tier | Keterangan |
|---|---|---|---|
| POST | `/api/auth/request-otp` | publik | body: `telegramId` → kode 6 digit dikirim ke Telegram |
| POST | `/api/auth/verify-otp` | publik | return `{ access_token, user }` |
| POST | `/api/auth/login` | publik | alternatif email + password |
| GET | `/api/me` | semua | profil + `{ tier, expiresAt, activeAlerts, alertLimit }` |
| GET | `/api/alerts` | trial, member | query: `status`, `symbol`, `page`, `limit` |
| POST | `/api/alerts` | trial, member | body: `{ symbol, type, targetPrice?, targetPctBps? }`, `source` = `dashboard` |
| DELETE | `/api/alerts/:id` | trial, member | status → `cancelled`. Baris tidak dihapus |
| POST | `/api/alerts/:id/reactivate` | trial, member | buat alert baru, `reactivatedFromId` = `:id` |
| GET | `/api/stocks` | trial, member | query: `q` (cari symbol/nama), maks. 20 hasil |
| GET | `/api/stocks/:symbol/quote` | trial, member | harga terakhir (dari `quoteService`) |
| GET | `/api/stocks/:symbol/history` | trial, member | query: `days` (maks. 90), dari `PriceDailies` |
| GET | `/api/dashboard/summary` | trial, member | alert aktif + jarak ke target, alert kena 30 hari terakhir |
| POST | `/api/subscriptions/checkout` | semua | buat `pending_payment` + link bayar |

Tier `free` yang memanggil endpoint mana pun selain `/api/auth/*`, `/api/me`,
dan `/api/subscriptions/checkout` → **403** dengan `requiredTier`.

### Contoh payload webhook masuk (Telegram Update object)
```json
{
  "update_id": 100001,
  "message": {
    "message_id": 42,
    "from": { "id": 123456789, "first_name": "Andi", "username": "andi" },
    "chat": { "id": 123456789, "type": "private" },
    "date": 1758594900,
    "text": "kabarin bbca kalau 3000"
  }
}
```
Callback tombol: `callback_query.data` berformat `alert:<aksi>:<alertId>`, dengan
`<aksi>` = `reactivate` / `shift` / `cancel`. Contoh `alert:shift:12`. **Wajib
cek alert itu milik `callback_query.from.id`** sebelum diproses.

### Contoh output parser (kontrak internal, wajib bentuk ini)

`helpers/alertParser.js` → `parse(text)`. **Fungsi murni, tidak menyentuh DB
atau harga.** Validasi yang butuh harga sekarang dilakukan `alertService`.
```json
{ "intent": "create_alert", "symbol": "BBCA", "type": "price_below", "targetPrice": 3000, "targetPctBps": null }
{ "intent": "create_alert", "symbol": "BBCA", "type": "price_auto",  "targetPrice": 3000, "targetPctBps": null }
{ "intent": "create_alert", "symbol": "TLKM", "type": "pct_drop",    "targetPrice": null, "targetPctBps": 1000 }
{ "intent": "create_alert", "symbol": "BBRI", "type": "below_avg_30d", "targetPrice": null, "targetPctBps": null }
{ "intent": "quote",        "symbol": "BBCA" }
{ "intent": "advice_request" }
{ "intent": "unknown" }
```
`price_auto` hanya ada di output parser. `alertService` mengubahnya jadi
`price_below` / `price_above` setelah membandingkan dengan harga sekarang.
**`price_auto` tidak pernah masuk DB.**

### Contoh quote (kontrak internal `quoteService.getQuote`)
```json
{
  "symbol": "BBCA",
  "lastPrice": 2990,
  "prevClose": 3100,
  "changeBps": -355,
  "asOf": "2026-09-23T03:15:00.000Z",
  "isDelayed": true
}
```

### Contoh input AI (metrics — dihitung `metricsService.forContext`, BUKAN AI)
Hanya memakai data sampai **penutupan hari bursa sebelumnya**, supaya satu
konteks berlaku sepanjang hari itu.
```json
{
  "symbol": "BBCA",
  "tradeDate": "2026-09-23",
  "prevClose": 3100,
  "change5dBps": -620,
  "change30dBps": -410,
  "avg30Close": 3210,
  "high30": 3400,
  "low30": 3050,
  "downDays5": 4,
  "downDays30": 18,
  "ihsgChange5dBps": 40,
  "ihsgChange30dBps": 120
}
```
AI menerima objek ini dan **hanya** menyusun narasi 1–2 kalimat. Dilarang
mengirim deret harga mentah ke AI untuk dihitung. Prompt wajib melarang saran
beli/jual dan prediksi harga.

### Contoh angka baris 📊 (`metricsService.forAlert`, BUKAN AI, selalu segar)
```json
{ "change5dBps": -830, "avg30Close": 3210 }
```
`change5dBps` = harga terakhir dibanding penutupan 5 hari bursa sebelumnya.

## 5. Header Autentikasi

```
Authorization: Bearer <access_token>
```
Payload JWT hanya berisi: `{ id, telegramId }`. **Tier tidak masuk token** —
selalu dihitung ulang dari DB.

Bot Telegram tidak pakai JWT. Identitas dari `message.from.id` /
`callback_query.from.id`, divalidasi lewat tabel `Users`.

## 6. Alur Kritis: Pesan Masuk → Alert Dibuat (Claude Code)

```
 1. Validasi header X-Telegram-Bot-Api-Secret-Token. Tidak cocok → 401, stop.
 2. update_id sudah pernah diproses? → 200, stop (idempoten).
 3. res.status(200).send() SEKARANG. Sisanya diproses setelah ini.
 4. Cari User by telegramId. Belum ada → buat User, trialStartedAt = now,
    trialEndsAt = now + 7 hari.
 5. tier = entitlementService.resolve(user, activeSubscription, now)
 6. Route:
    - command        → commandHandler
    - callback_query → callbackHandler (cek kepemilikan alert)
    - text + stateStore berisi 'shift_target' → perlakukan text sebagai harga baru
    - text lainnya   → alertParser.parse(text)
 7. intent 'quote'          → quoteService.getQuote → balas harga. Stop.
    intent 'advice_request' → balas netral (SPEC §8 baris terakhir). Stop.
    intent 'unknown'        → balas panduan format. Stop.
 8. intent 'create_alert':
    a. stockService.findActiveBySymbol(symbol) → tidak ada → tolak
    b. entitlementService.can(tier, capabilityUntuk(type)) → tidak → upsell, stop
    c. hitung alert aktif user ≥ entitlementService.maxActiveAlerts(tier) → upsell, stop
    d. alertService.create(...)  (langkah 9)
 9. alertService.create:
    a. quote = quoteService.getQuote(symbol, now)   ← MarketUnavailable → balas
       "data harga sedang tidak tersedia, coba lagi sebentar", JANGAN simpan
    b. price_auto → price_below / price_above (target sama dengan harga → tolak)
    c. pct_* → hitung targetPrice (SPEC §8) dengan pembulatan tick
    d. below_avg_30d → historyService.ensureHistory, lalu
       metricsService.avgClose(StockId, 30, tradeDate). Data < 30 hari → tolak
    e. validasi tick (price.isValidTick) dan "belum terlewati" (SPEC §5 no. 9)
    f. insert Alert: status 'active', basePrice = quote.lastPrice
    g. historyService.ensureHistory(StockId, now) kalau belum dipanggil
10. Balas lewat telegram/client.js (formatter.alertCreated)
```

## 7. Alur Kritis: Cek Alert (Codex — `jobs/alertCheckJob.js`, tiap 5 menit)

```
 1. marketCalendar.isInSession(now) false → stop (tidak memanggil provider).
 2. rows = alertService.findCheckable(now)
    → sudah menerapkan batas free, isActive saham, dan status 'active'.
 3. symbols = daftar unik dari rows. Kosong → stop.
 4. quotes = quoteService.getQuotes(symbols, now)
    → satu panggilan batch ke provider, quote mencurigakan sudah dibuang (SPEC §5 no. 13).
 5. Untuk alert below_avg_30d: avg30 = metricsService.avgClose(StockId, 30, tradeDate)
    (sekali per saham per putaran, bukan per alert).
 6. Untuk setiap row: kalau alertRules.isHit(alert, quote, avg30):
    a. isFirst = alertService.markTriggered(alert.id, { price, asOf }, now)
    b. isFirst false → lewati (sudah diproses putaran/instance lain).
    c. tier punya 'alert.context'?
       - ya  → numbers = metricsService.forAlert(StockId, quote)
               context = contextService.getContext(StockId, tradeDate, tier)
               (gagal/timeout → context = null, tetap lanjut)
       - tidak → numbers = null, context = null
    d. kirim formatter.alertTriggered({...}) lewat telegram/client.js
 7. Kegagalan satu alert (mis. Telegram error) tidak menghentikan alert lain.
    Log: { event: 'alert.notify.failed', alertId }. Alert tetap 'triggered'.
```

`alertRules.isHit`:
```
price_below, pct_drop  → quote.lastPrice <= alert.targetPrice
price_above, pct_rise  → quote.lastPrice >= alert.targetPrice
below_avg_30d          → quote.lastPrice <  avg30
```

## 8. Alur Kritis: Setelah Bursa Tutup (Codex — `jobs/eodJob.js`, 16:30 WIB)

```
1. marketCalendar.isTradingDay(today) false → stop.
2. symbols = alertService.activeSymbols() + 'IHSG'
3. historyService.saveDailyBars(symbols, today)  → upsert PriceDailies
4. users = trial/member dengan dailySummaryEnabled = true dan ≥1 alert aktif
   (lewat alertService.findForDailySummary(now) — sudah memakai entitlementService)
5. Untuk tiap user: formatter.dailySummary(items, today) → kirim.
   TANPA AI. Semua angka dari PriceDailies hari ini.
```

## 9. Jam Bursa & Kalender (Codex — `market/marketCalendar.js`)

Semua dalam **WIB (Asia/Jakarta)**. Nilainya di `config/constants.js`
→ `MARKET_SESSIONS`. **Owner wajib cek ulang ke jadwal resmi IDX sebelum rilis.**

| Hari | Sesi I | Sesi II |
|---|---|---|
| Senin – Kamis | 09:00 – 12:00 | 13:30 – 16:00 |
| Jumat | 09:00 – 11:30 | 14:00 – 16:00 |

- Istirahat siang: `isInSession` → `false`, tidak memanggil provider.
- Karena data tertunda ±15 menit, `isInSession` memperpanjang tiap sesi
  `DATA_DELAY_MINUTES` (15 menit) supaya harga penutupan sesi tetap dicek.
- Hari libur bursa: `config/marketHolidays.js` (array `YYYY-MM-DD`), diisi owner
  tiap awal tahun dari kalender IDX.
- `MARKET_FORCE_OPEN=true` di `.env` membuat `isInSession` selalu `true`.
  **Hanya berlaku kalau `NODE_ENV !== 'production'`** — untuk uji lokal di akhir pekan.

## 10. Alur Kritis: Cron Expiry (Codex — `jobs/expiryJob.js`, tiap hari 00:05 WIB)

```
1. subscriptionService.expireOverdue(now)  → 'active' yang expiresAt < now → 'expired'
2. subscriptionService.findUsersForReminder(now) →
   { trialEnded, trialEndsTomorrow, memberEndsTomorrow }
3. trialEnded        → kirim formatter.trialEnded(user, unmonitoredAlerts)
                        ("alert yang tidak dipantau lagi: ...")
   trialEndsTomorrow → kirim formatter.trialEndsTomorrow(...)
   memberEndsTomorrow→ kirim formatter.memberEndsTomorrow(...)
4. Tidak ada alert yang diubah atau dihapus. Turun tier hanya mempengaruhi
   alert mana yang dipantau (SPEC §4).
```

## 11. Alur: Sinkron Daftar Saham (Codex — `jobs/stockSyncJob.js`, Senin 07:00 WIB)

```
1. list = marketClient.listSymbols()
2. Upsert ke Stocks by symbol. Ticker yang hilang dari list → isActive = false.
   Tidak pernah menghapus baris Stocks.
3. Kalau list kosong / provider error → JANGAN menonaktifkan apa pun. Log & stop.
```

## 12. Kontrak Fungsi Antar Agent Server

Ini satu-satunya titik temu Claude Code dan Codex. Semua fungsi menerima `now`
sebagai parameter kalau butuh waktu. Semua error dilempar sebagai `{ name, message }`.

**Aturan nama export:** tabel di bawah memakai nama file. File service, market,
ai, dan jobs meng-export **satu class dengan static method** bernama PascalCase
(`quoteService.js` → `class QuoteService`, `marketCalendar.js` →
`class MarketCalendar`). Pengecualian yang meng-export fungsi biasa:
`marketClient.js`, `aiClient.js`, `telegram/client.js`, `telegram/formatter.js`,
`helpers/*.js`.

**Akses model database:**
| Model | Claude Code | Codex |
|---|---|---|
| `User`, `Subscription`, `Alert` | baca & tulis | **hanya lewat fungsi di tabel ini** |
| `Stock` | baca & tulis | baca & tulis (`stockSyncJob`) |
| `PriceDaily` | baca | baca & tulis (`historyService`) |
| `Analysis` | — | baca & tulis (`contextService`) |

Model dan migration-nya tetap ditulis Claude Code. Codex hanya memakai.

### Disediakan Codex, dipakai Claude Code
| Fungsi | Return |
|---|---|
| `quoteService.getQuote(symbol, now)` | quote (§4) — atau throw `{ name: 'MarketUnavailable' }` |
| `quoteService.getQuotes(symbols, now)` | `Map<symbol, quote>`, quote mencurigakan tidak ada di Map |
| `historyService.ensureHistory(StockId, now)` | `void` — backfill `HISTORY_BACKFILL_DAYS` kalau PriceDailies < `AVG_WINDOW_DAYS` baris |
| `historyService.getHistory(StockId, days, now)` | `[{ tradeDate, open, high, low, close, volume }]` urut naik |
| `metricsService.avgClose(StockId, days, beforeDate)` | integer (dibulatkan ke bawah) atau `null` kalau data kurang |
| `metricsService.forAlert(StockId, quote)` | `{ change5dBps, avg30Close }` |
| `marketCalendar.tradeDateOf(now)` | `'YYYY-MM-DD'` WIB |
| `marketCalendar.isInSession(now)` | boolean |

### Disediakan Claude Code, dipakai Codex
| Fungsi | Return |
|---|---|
| `EntitlementService.resolve(user, activeSubscription, now)` | `'trial' \| 'member' \| 'free'` |
| `EntitlementService.can(tier, capability)` | boolean |
| `EntitlementService.maxActiveAlerts(tier)` | integer |
| `alertService.findCheckable(now)` | `[{ alert, user, tier, stock }]` |
| `alertService.markTriggered(alertId, { price, asOf }, now)` | boolean — `true` hanya kalau UPDATE mengubah 1 baris |
| `alertService.activeSymbols()` | `string[]` unik, saham `isActive` |
| `alertService.findForDailySummary(now)` | `[{ user, items: [{ alert, stock }] }]` |
| `subscriptionService.expireOverdue(now)` | jumlah baris diubah |
| `subscriptionService.findUsersForReminder(now)` | lihat §10 |
| `formatter.alertTriggered({ alert, stock, quote, numbers, context, tier })` | `{ text, reply_markup }` |
| `formatter.dailySummary(items, tradeDate)` | `{ text }` |
| `formatter.trialEnded(user, unmonitoredAlerts)` / `trialEndsTomorrow(user)` / `memberEndsTomorrow(user)` | `{ text }` |
| `telegramClient.sendMessage(chatId, text, options)` | respons Telegram |

### Capability (dipakai kedua agent, didefinisikan di `entitlementService`)
```
quote.read, alert.price, alert.percent, alert.average, alert.context,
summary.daily, dashboard.read
```

Butuh fungsi yang tidak ada di tabel ini? **Jangan buat sendiri di folder agent
lain.** Buka issue ke owner untuk menambah baris di tabel ini dulu.
