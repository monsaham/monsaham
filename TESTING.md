# TESTING.md — TDD, Coverage & End-to-End

> Aturan pokok: **test ditulis sebelum kode.** PR yang test-nya ditulis belakangan
> dianggap belum selesai, walaupun hijau.

---

## 1. Siklus yang Diulang Terus

```
RED      → tulis test dari aturan bisnis di SPEC.md, jalankan, pastikan GAGAL
GREEN    → tulis kode seminimal mungkin sampai lulus
REFACTOR → rapikan, test tetap hijau
COMMIT   → commit test dulu, baru commit implementasi
```

Bukti di PR: dua commit terpisah, `test:` mendahului `feat:`. Kalau di riwayat
commit test dan implementasi masuk barengan, owner berhak menolak PR-nya.

**Kenapa seketat ini:** ini produk berbayar yang mengirim notifikasi harga
saham. Bug di `entitlementService` = user gratisan pakai fitur berbayar (rugi)
atau member tidak dapat fitur (komplain). Bug di `alertRules` atau
`markTriggered` = alert palsu atau dobel, dan user langsung berhenti percaya.
Bug di kalimat bot = risiko hukum.

## 2. Tiga Level Test

| Level | Folder | Database | AI | Data saham | Telegram |
|---|---|---|---|---|---|
| Unit | `__tests__/unit/` | mock | mock | mock | mock |
| Integration | `__tests__/integration/` | **Postgres test asli** | mock | mock | mock |
| E2E | `__tests__/e2e/` | **Postgres test asli** | mock | mock | mock, payload diperiksa |

Perbandingan jumlah kira-kira: unit 65%, integration 25%, e2e 10%.

**API AI, API data saham, dan Telegram SELALU di-mock.** Tidak ada panggilan
jaringan sungguhan di CI. Uji provider sungguhan dilakukan manual sebelum rilis.

## 3. Target Coverage (CI gagal kalau di bawah ini)

### Global
| Metrik | Minimum |
|---|---|
| Statements | **85%** |
| Branches | **80%** |
| Functions | **85%** |
| Lines | **85%** |

### Per file (lebih ketat untuk yang menyentuh uang, hak akses, dan kebenaran alert)

| File | Owner | Statements | Branches | Kenapa seketat ini |
|---|---|---|---|---|
| `services/entitlementService.js` | Claude Code | **100%** | **100%** | Menentukan siapa bayar dan dapat apa |
| `helpers/price.js` | Claude Code | **100%** | **100%** | Fungsi murni: tick, bps, format |
| `market/alertRules.js` | Codex | **100%** | **100%** | Menentukan alert kena atau tidak |
| `market/marketCalendar.js` | Codex | **100%** | **100%** | Salah jam = alert telat atau biaya data bocor |
| `ai/complianceFilter.js` | Codex | **100%** | **100%** | Pagar hukum |
| `services/alertService.js` | Claude Code | 95% | 90% | Batas tier, UPDATE bersyarat |
| `helpers/alertParser.js` | Claude Code | 95% | 90% | Pintu masuk semua tier |
| `market/quoteService.js` | Codex | 95% | 90% | Cache & sanity check |
| `market/metricsService.js` | Codex | 95% | 90% | Angka yang dilihat user |
| `ai/contextService.js` | Codex | 90% | 85% | Cache & pagar free |
| `jobs/` | Codex | 90% | 85% | |
| `middlewares/` | Claude Code | 90% | 85% | Auth & tier |
| `controllers/` | Claude Code | 85% | 80% | |
| `telegram/` | Claude Code | 85% | 80% | |
| `models/`, `migrations/`, `seeders/`, `config/`, `bin/` | — | — | — | **dikecualikan** |

### Konfigurasi
```js
// server/jest.config.js
module.exports = {
  testEnvironment: 'node',
  collectCoverage: true,
  coverageReporters: ['text-summary', 'lcov', 'json-summary'],
  collectCoverageFrom: [
    '**/*.js',
    '!**/node_modules/**',
    '!**/coverage/**',
    '!**/__tests__/**',
    '!models/**', '!migrations/**', '!seeders/**', '!config/**', '!bin/**',
    '!jest.config.js',
  ],
  coverageThreshold: {
    global:                             { statements: 85, branches: 80, functions: 85, lines: 85 },
    './services/entitlementService.js': { statements: 100, branches: 100, functions: 100, lines: 100 },
    './helpers/price.js':               { statements: 100, branches: 100, functions: 100, lines: 100 },
    './market/alertRules.js':           { statements: 100, branches: 100, functions: 100, lines: 100 },
    './market/marketCalendar.js':       { statements: 100, branches: 100, functions: 100, lines: 100 },
    './ai/complianceFilter.js':         { statements: 100, branches: 100, functions: 100, lines: 100 },
    './services/alertService.js':       { statements: 95, branches: 90, functions: 95, lines: 95 },
    './helpers/alertParser.js':         { statements: 95, branches: 90, functions: 95, lines: 95 },
    './market/quoteService.js':         { statements: 95, branches: 90, functions: 95, lines: 95 },
    './market/metricsService.js':       { statements: 95, branches: 90, functions: 95, lines: 95 },
    './ai/contextService.js':           { statements: 90, branches: 85, functions: 90, lines: 90 },
    './jobs/':                          { statements: 90, branches: 85, functions: 90, lines: 90 },
    './middlewares/':                   { statements: 90, branches: 85, functions: 90, lines: 90 },
    './controllers/':                   { statements: 85, branches: 80, functions: 85, lines: 85 },
  },
  setupFilesAfterEnv: ['<rootDir>/__tests__/setup.js'],
  testTimeout: 15000,
  maxWorkers: 1,
};
```

**`jest.config.js` dipegang owner (manusia), bukan agent.** Jest gagal kalau
ambang per file menunjuk ke file yang belum ada, jadi mulailah dengan ambang
global + file yang sudah ada saja. Saat me-merge PR yang membuat file di tabel
di atas, owner menambahkan baris ambangnya di commit yang sama. Agent dilarang
mengubah file ini, terutama menurunkan angkanya.

### Cara membaca persentasenya
Output `npm run test:coverage`:
```
=============================== Coverage summary ===============================
Statements   : 88.24% ( 615/697 )
Branches     : 84.11% ( 281/334 )
Functions    : 90.00% ( 108/120 )
Lines        : 88.41% ( 602/681 )
================================================================================
```
Angka ini **wajib ditempel di deskripsi PR**. Coverage yang turun lebih dari 0,5
poin dibanding `main` ditolak, walaupun masih di atas ambang.

## 4. Struktur Folder Test & Kepemilikan

```
server/__tests__/
├── setup.js                      ← Claude Code (koneksi DB test, truncate antar test)
├── factories/                    ← Claude Code
│   ├── userFactory.js            createUser({ tier: 'member' })
│   ├── stockFactory.js           createStock({ symbol: 'BBCA' })
│   ├── alertFactory.js           createAlert({ type: 'price_below', targetPrice: 3000 })
│   └── market/                   ← Codex
│       ├── priceDailyFactory.js  createPriceDailies(stockId, [3100, 3080, ...])
│       └── quoteFactory.js       buildQuote({ symbol: 'BBCA', lastPrice: 2990 })
├── fixtures/
│   ├── telegram/                 ← Claude Code (update object: text, command, callback)
│   ├── parsing/                  ← Claude Code (pasangan input → expected, minimal 60)
│   └── market/                   ← Codex (respons provider mentah, kasus jadwal bursa)
├── mocks/
│   ├── telegramClient.js         ← Claude Code (menangkap payload sendMessage)
│   ├── marketClient.js           ← Codex (harga bisa diatur per test, punya counter per fungsi)
│   └── aiClient.js               ← Codex (output bisa diatur, punya counter, bisa disuruh timeout)
├── unit/                         owner test = owner file yang dites
├── integration/                  owner test = owner file yang dites
└── e2e/
    ├── webhook.*.test.js         ← Claude Code
    ├── api.*.test.js             ← Claude Code
    └── jobs.*.test.js            ← Codex
```

**Mock wajib punya counter.** Banyak test berbunyi "pastikan AI tidak dipanggil
sama sekali" (`expect(aiClient.callCount).toBe(0)`) atau "pastikan provider
dipanggil sekali untuk banyak alert" (`expect(marketClient.getQuotes.callCount).toBe(1)`).

**Dua agent, satu pola mock.** Claude Code memakai mock milik Codex
(`marketClient`, `aiClient`) di test-nya, dan sebaliknya Codex memakai
`telegramClient` milik Claude Code. Dilarang membuat mock tandingan di folder
sendiri. Kalau mock kurang fitur → issue ke pemiliknya.

## 5. Skenario End-to-End Backend (wajib ada semua)

Setiap skenario memeriksa tiga hal: **(a)** status & body response (atau nilai
kembali job), **(b)** isi database setelahnya, **(c)** payload yang dikirim ke
Telegram.

### Onboarding & tier — Claude Code (E1–E3), Codex (E4–E5)
| # | Skenario | Yang diperiksa |
|---|---|---|
| E1 | `/start` user baru | User dibuat, `trialEndsAt` = +7 hari, balasan memuat disclaimer |
| E2 | `/start` dua kali | Tidak ada duplikat, trial tidak diperpanjang |
| E3 | `/status` tier trial | Sisa hari benar, "Alert aktif: 0 dari 50" |
| E4 | `expiryJob`: member lewat `expiresAt` | Status `expired`, request berikutnya diperlakukan free |
| E5 | `expiryJob`: trial habis, user punya 4 alert | Semua alert tetap `active`, pesan menyebut alert yang tidak dipantau lagi |

### Membuat alert — Claude Code
| # | Skenario | Yang diperiksa |
|---|---|---|
| E6 | "bbca 3000" (trial), harga 3.120 | 1 alert `price_below`, `basePrice` 3120, balasan menyebut jam data |
| E7 | Teks sama (free) | Alert dibuat, **aiClient.callCount === 0** |
| E8 | "tlkm turun 10%" (trial), harga 3.600 | `pct_drop`, `targetPctBps` 1000, `targetPrice` 3240 |
| E9 | "tlkm turun 10%" (free) | 0 alert, balasan upsell, **aiClient.callCount === 0**, **marketClient tidak dipanggil** |
| E10 | Alert ke-3 (free) | 0 alert baru, balasan "2 dari 2" + /upgrade |
| E11 | "bbca 3003" | 0 alert, tombol [3.000] [3.010] |
| E12 | "bbca <= 3200" saat harga 3.120 | 0 alert, balasan menyebut harga sekarang |
| E13 | "xxxx 3000" | 0 alert, "Kode XXXX tidak ditemukan" |
| E14 | "bagus gak bbca?" | 0 alert, balasan netral, **balasan lolos `complianceFilter`** |
| E15 | Provider data down saat membuat alert | 0 alert, balasan "data harga tidak tersedia", tidak 500 |
| E16 | "bbri bawah rata2", PriceDailies < 30 hari | 0 alert, balasan penjelasan |
| E17 | Callback `alert:cancel:<id>` untuk alert user lain | Tidak ada perubahan, balasan "tidak ditemukan" |
| E18 | [Geser target] → user ketik "2900" | Alert lama `cancelled`, alert baru `active` dengan `reactivatedFromId` |
| E19 | Ketik harga setelah TTL state lewat | 0 alert, balasan "sesi habis", tidak 500 |

### Keamanan webhook — Claude Code
| # | Skenario | Yang diperiksa |
|---|---|---|
| E20 | Webhook tanpa secret token | 401, tidak ada efek samping apa pun |
| E21 | Webhook secret token salah | 401 |
| E22 | `update_id` dikirim dua kali | Hanya 1 alert tersimpan |

### Cek alert — Codex (`jobs.alertCheck.test.js`)
| # | Skenario | Yang diperiksa |
|---|---|---|
| E23 | Rabu 10:15 WIB, harga 2.990, alert ≤ 3.000 (trial) | `triggered`, 1 pesan dengan 📊 + 💡 + disclaimer, 1 baris `Analyses` |
| E24 | Sama, tapi user free | `triggered`, pesan tanpa 📊/💡 + CTA, **aiClient.callCount === 0**, tidak ada query ke `Analyses` |
| E25 | Job dijalankan 2× dengan harga yang sama | **Tepat 1 pesan** terkirim |
| E26 | Sabtu / 12:30 WIB Senin / hari libur | `marketClient` **tidak dipanggil** |
| E27 | 100 alert di 3 ticker | `marketClient.getQuotes` dipanggil **1×** dengan 3 simbol |
| E28 | Quote +50% dari penutupan kemarin | Tidak `triggered`, log `market.quote.suspicious` |
| E29 | User free dengan 4 alert (sisa trial) | Hanya 2 alert `price_*` tertua yang dicek |
| E30 | AI timeout | Alert `triggered`, pesan terkirim dengan 📊 tanpa 💡 |
| E31 | Output AI mengandung "murah" | 💡 dibuang, `Analyses.insight` null; alert kedua saham sama hari itu → **AI tidak dipanggil lagi** |
| E32 | 2 user trial, alert saham sama kena di hari sama | **aiClient dipanggil 1×** (cache) |
| E33 | `below_avg_30d`, harga turun di bawah rata-rata | `triggered` |
| E34 | Telegram gagal untuk 1 dari 3 alert | 2 lainnya tetap terkirim, ketiganya `triggered` |

### Setelah bursa tutup — Codex (`jobs.eod.test.js`)
| # | Skenario | Yang diperiksa |
|---|---|---|
| E35 | Hari bursa, 16:30 WIB | PriceDailies ter-upsert; ringkasan ke trial/member yang `dailySummaryEnabled`; **tidak** ke free; **tidak** ke yang mematikan; aiClient 0 |
| E36 | Hari libur bursa | Tidak ada panggilan provider, tidak ada pesan |

### REST API — Claude Code
| # | Skenario | Yang diperiksa |
|---|---|---|
| E37 | request-otp → verify-otp → `GET /api/me` | Token valid, `tier` & `alertLimit` benar |
| E38 | OTP salah 3× | 429 |
| E39 | `GET /api/alerts` tanpa token | 401 |
| E40 | `GET /api/alerts` tier free | 403 + `requiredTier` |
| E41 | `POST /api/alerts` target tidak sesuai tick | 400 |
| E42 | `DELETE /api/alerts/:id` milik user lain | 404, alert tidak berubah |
| E43 | `POST /api/alerts/:id/reactivate` | Baris baru, baris lama utuh |
| E44 | Checkout → webhook bayar | Subscription `active`, `expiresAt` +30 hari |
| E45 | Webhook bayar dikirim dua kali | Tetap 1 subscription (idempoten) |
| E46 | Webhook bayar signature salah | 401, tidak ada perubahan data |

**E7, E9, E24, E25, E31, E40 adalah test paling penting di repo ini.** Tiga
menjaga pagar antara user gratis dan biaya AI, satu menjaga alert tidak dobel,
satu menjaga kalimat bot tetap netral, satu menjaga dashboard berbayar.
Dilarang di-`skip` dengan alasan apa pun.

## 6. Aturan Menulis Test

- Nama test menyebut aturan bisnis yang diuji:
  ```js
  describe('entitlementService', () => {
    it('mengembalikan free kalau trial sudah lewat dan tidak ada subscription aktif', ...);
    it('maxActiveAlerts mengembalikan FREE_ALERT_LIMIT untuk tier free', ...);
  });
  describe('alertRules.isHit', () => {
    it('price_below kena saat harga sama persis dengan target', ...);
  });
  ```
- **Waktu wajib di-freeze, dan pakai jam WIB yang jelas.** Test jam bursa tidak
  stabil kalau bergantung jam nyata atau zona waktu mesin CI:
  ```js
  // Rabu 23 Sep 2026, 10:15 WIB = 03:15 UTC
  jest.spyOn(dateHelper, 'now').mockReturnValue(new Date('2026-09-23T03:15:00Z'));
  ```
  Setiap test `marketCalendar` wajib punya kasus: tepat jam buka, tepat jam
  tutup, istirahat siang, Jumat, Sabtu, hari libur, dan perpanjangan
  `DATA_DELAY_MINUTES`.
- **Tabel kasus untuk fungsi murni.** `price.js`, `alertRules.js`,
  `alertParser.js` dites pakai `it.each` dari fixtures, bukan satu per satu.
  Wajib ada batas rentang fraksi (199/200, 499/500, 1995/2000, 4990/5000).
- Pakai factory, bukan fixture panjang: `await createAlert({ type: 'pct_drop' })`.
- Test harus lulus kalau file-nya dijalankan sendirian, bukan cuma sebagai satu
  suite besar. Cache in-memory (`quoteService`, `stateStore`, update_id)
  wajib di-reset di `beforeEach`.
- Test yang cuma `expect(response.status).toBe(200)` tanpa memeriksa isi
  dianggap tidak menguji apa-apa.
- **Test kalimat netral:** setiap fungsi di `formatter.js` punya satu test yang
  menjalankan outputnya melalui `complianceFilter.isClean` dan mengharapkan `true`.

## 7. Yang Memblokir Merge

1. `npm run lint` — 0 error
2. `npm run test:unit` — hijau
3. `npm run test:integration` — hijau
4. `npm run test:e2e` — hijau, tidak ada yang di-skip
5. Ambang coverage global + per file terpenuhi
6. Coverage tidak turun > 0,5 poin dari `main`
7. Migration bisa `up` lalu `down` bersih
8. Build frontend sukses
9. **PR tidak menyentuh file di luar wilayah agent pembuatnya** (dicek CI, lihat
   `WORKFLOW.md` Fase 1)

## 8. Script npm

```json
{
  "scripts": {
    "test":             "NODE_ENV=test jest",
    "test:watch":       "NODE_ENV=test jest --watch",
    "test:unit":        "NODE_ENV=test jest __tests__/unit",
    "test:integration": "NODE_ENV=test jest __tests__/integration --runInBand",
    "test:e2e":         "NODE_ENV=test jest __tests__/e2e --runInBand",
    "test:coverage":    "NODE_ENV=test jest --coverage",
    "db:migrate":       "sequelize db:migrate",
    "db:migrate:test":  "NODE_ENV=test sequelize db:migrate",
    "db:migrate:undo":  "sequelize db:migrate:undo",
    "cron":             "node bin/cron.js"
  }
}
```

## 9. Test Frontend (OWNER: Cline)

- Vitest + React Testing Library.
- Coverage minimum: **75%** untuk `src/components` dan `src/pages`, **90%** untuk
  `src/services/api.js`.
- Setiap halaman wajib punya test untuk 4 keadaan: loading, kosong, error, sukses.
- Wajib ada test untuk **403 + `requiredTier` → tampil halaman upsell**, bukan
  pesan error merah.
- Wajib ada test bahwa setiap tampilan harga memuat teks "data tertunda".
- Wajib ada test **503 → pesan "data harga sedang tidak tersedia"**, bukan crash.
- Mock API pakai contoh response dari `ARCHITECTURE.md §3–4`. Kalau kontraknya
  berubah, mock ikut berubah di PR yang sama.
