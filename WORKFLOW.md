# WORKFLOW.md — Panduan End-to-End

Cara menjalankan project Chatbot Alert Harga Saham dengan 4 AI agent
(Claude Code, Codex, Cline, Antigravity) tanpa mereka halu dan tabrakan.

| Agent | Wilayah | File aturan yang dibaca otomatis |
|---|---|---|
| Claude Code | `server/` — bot, API, DB, service, helper | `CLAUDE.md` |
| Codex | `server/` — `market/`, `ai/`, `jobs/`, `bin/cron.js` | `AGENTS.md` |
| Cline | `client/` | `.clinerules/00-rules.md` |
| Antigravity | `.github/`, `docs/`, `README.md` | `AGENTS.md` |

Ketiga file aturan berisi teks yang **sama persis** (`AGENTS.md`).

---

## FASE 0 — Persiapan (kamu sendiri, TANPA agent) — ±3 jam

Ini fase paling penting. Jangan lewati.

```bash
mkdir monsaham-project && cd monsaham-project
git init
mkdir -p server client docs .github/workflows .github/ISSUE_TEMPLATE .clinerules
```

1. Taruh `SPEC.md`, `PRD.md`, `ARCHITECTURE.md`, `ERD.md`, `STYLE.md`,
   `TESTING.md`, `AGENTS.md`, `WORKFLOW.md`. Baca ulang `ARCHITECTURE.md §12`
   (kontrak Claude Code ↔ Codex) — itu harus **final** sebelum agent mulai.
2. **Selesaikan dua keputusan yang memblokir coding** (`PRD.md §9`):
   - Provider data saham + lisensinya. Kamu butuh contoh respons API aslinya
     untuk `__tests__/fixtures/market/` — Codex tidak boleh mengarang formatnya.
   - Jam sesi & tabel fraksi harga — cocokkan dengan situs IDX.
3. `.gitignore` dan `.env.example` (sudah ada di paket ini).
4. **Bikin 3 bot Telegram lewat @BotFather:** dev, staging, production.
   Satu token tidak boleh dipakai dua environment — `setWebhook` akan saling
   menimpa.
5. Salin aturan agent:
   ```bash
   cp AGENTS.md CLAUDE.md
   cp AGENTS.md .clinerules/00-rules.md
   ```
   Codex dan Antigravity membaca `AGENTS.md` langsung. **Setiap kali `AGENTS.md`
   berubah, ulangi dua baris ini** — kalau tidak, agent membaca aturan berbeda.
6. Commit & push:
   ```bash
   git add -A
   git commit -m "docs: spesifikasi awal project"
   git remote add origin git@github.com:USERNAME/monsaham-project.git
   git push -u origin main
   ```

---

## FASE 1 — Setup GitHub — ±45 menit

1. **Branch protection** untuk `main`: require PR, require status checks,
   jangan izinkan force push.
2. **Labels**: `claude-code`, `codex`, `cline`, `antigravity`, `bug`, `blocked`,
   `kontrak` (issue yang mengubah `ARCHITECTURE.md §12`)
3. **Milestone**: `v1.0 MVP`
4. **Project board**: `Todo / In Progress / Review / Done`
5. **Issue template** `.github/ISSUE_TEMPLATE/task.md` (sudah ada di paket ini).
6. **Cek wilayah otomatis di CI** (dikerjakan Antigravity di issue #4). Workflow
   membaca awalan branch dan menolak PR yang mengubah file di luar wilayahnya:

   | Awalan branch | Boleh mengubah |
   |---|---|
   | `claude/` | lihat `AGENTS.md §2` baris Claude Code |
   | `codex/` | `server/market/**`, `server/ai/**`, `server/jobs/**`, `server/bin/cron.js`, test Codex |
   | `cline/` | `client/**` |
   | `anti/` | `.github/**`, `docs/**`, `README.md`, konfigurasi root |
   | `docs/` | hanya kamu — file `.md` di root, `constants.js`, `marketHolidays.js`, `jest.config.js` |

   Ini penting karena dua agent bekerja di `server/`. Tanpa cek otomatis, cepat
   atau lambat salah satunya "cuma membetulkan sedikit" file milik yang lain.

---

## FASE 2 — Fondasi (KERJAKAN SENDIRI, jangan diserahkan ke agent) — ±4 jam

Agent bekerja jauh lebih baik kalau sudah ada pola untuk ditiru. Banyak file bisa
**disalin dari Mon Mon** lalu disesuaikan:

| File | Dari Mon Mon? | Yang diubah |
|---|---|---|
| `server/app.js`, `bin/www.js`, `config/config.js` | salin | nama project |
| `middlewares/errorHandler.js` | salin | tambah `TooManyRequests` & `MarketUnavailable` (STYLE §2) |
| `middlewares/telegramAuth.js` | salin | — |
| `helpers/dateHelper.js` | salin | — |
| `telegram/client.js` | salin | — |
| `routes/webhook.js` | salin | — |
| `services/entitlementService.js` + test | salin | capability baru + `maxActiveAlerts` (STYLE §4) |
| Model `User` + migration | salin | hapus Wallet/Category, tambah `dailySummaryEnabled` |
| `__tests__/e2e/webhook.start.test.js` (E1) | salin | trial 7 hari, tanpa wallet/kategori |
| `config/constants.js` | **tulis baru** | isi di bawah |
| `config/marketHolidays.js` | **tulis baru** | isi dari kalender libur bursa IDX tahun berjalan |
| `jest.config.js` | salin | ambang global dulu (`TESTING.md §3`) |

**`server/config/constants.js`** — tulis lengkap sekarang, supaya dua agent
server tidak perlu menyentuhnya:
```js
// config/constants.js
//
// Semua angka "ajaib" wajib didefinisikan di sini (STYLE.md §14).
// Hanya owner yang boleh mengubah file ini.

const TRIAL_DAYS = 7;
const MEMBER_DAYS = 30;
const FREE_ALERT_LIMIT = 2;
const MEMBER_ALERT_LIMIT = 50;

const CONFIRMATION_TTL_SECONDS = 300;
const OTP_MAX_ATTEMPTS = 3;

const AI_CALL_TIMEOUT_MS = 20000;
const MARKET_CALL_TIMEOUT_MS = 10000;
const QUOTE_CACHE_TTL_SECONDS = 300;
const QUOTE_SANITY_MAX_BPS = 3500;       // 35% dari penutupan kemarin
const DATA_DELAY_MINUTES = 15;

const AVG_WINDOW_DAYS = 30;
const HISTORY_BACKFILL_DAYS = 60;
const MAX_HISTORY_DAYS = 90;

const MARKET_TIMEZONE = 'Asia/Jakarta';
const ALERT_CHECK_CRON = '*/5 8-16 * * 1-5';   // marketCalendar yang menyaring jam persisnya
const EOD_CRON = '30 16 * * 1-5';
const EXPIRY_CRON = '5 0 * * *';
const STOCK_SYNC_CRON = '0 7 * * 1';

// WAJIB dicek ulang ke jadwal resmi IDX sebelum rilis.
const MARKET_SESSIONS = {
  mondayToThursday: [{ start: '09:00', end: '12:00' }, { start: '13:30', end: '16:00' }],
  friday:           [{ start: '09:00', end: '11:30' }, { start: '14:00', end: '16:00' }],
};

// WAJIB dicek ulang ke peraturan fraksi harga IDX terbaru sebelum rilis.
const TICK_SIZE_TABLE = [
  { below: 200, tick: 1 },
  { below: 500, tick: 2 },
  { below: 2000, tick: 5 },
  { below: 5000, tick: 10 },
  { below: Infinity, tick: 25 },
];

// Dicocokkan per kata/frasa utuh, tidak peka huruf besar-kecil (complianceFilter).
const BANNED_PHRASES = [
  'murah', 'mahal', 'diskon', 'undervalued', 'overvalued', 'cuan',
  'saatnya beli', 'saatnya jual', 'layak beli', 'layak dibeli', 'layak jual',
  'sebaiknya beli', 'sebaiknya jual', 'segera beli', 'segera jual',
  'rekomendasi', 'rekomendasikan', 'target harga', 'sinyal beli', 'sinyal jual',
  'pasti naik', 'pasti turun', 'akan naik', 'akan turun', 'berpotensi naik',
  'momen bagus', 'kesempatan emas', 'buy', 'sell',
];

const DISCLAIMER_LINE = 'ℹ️ Pengingat harga, bukan saran investasi.';

module.exports = {
  TRIAL_DAYS, MEMBER_DAYS, FREE_ALERT_LIMIT, MEMBER_ALERT_LIMIT,
  CONFIRMATION_TTL_SECONDS, OTP_MAX_ATTEMPTS,
  AI_CALL_TIMEOUT_MS, MARKET_CALL_TIMEOUT_MS, QUOTE_CACHE_TTL_SECONDS,
  QUOTE_SANITY_MAX_BPS, DATA_DELAY_MINUTES,
  AVG_WINDOW_DAYS, HISTORY_BACKFILL_DAYS, MAX_HISTORY_DAYS,
  MARKET_TIMEZONE, ALERT_CHECK_CRON, EOD_CRON, EXPIRY_CRON, STOCK_SYNC_CRON,
  MARKET_SESSIONS, TICK_SIZE_TABLE, BANNED_PHRASES, DISCLAIMER_LINE,
};
```

**`server/config/marketHolidays.js`**:
```js
// Tanggal libur bursa (bukan akhir pekan), format YYYY-MM-DD WIB.
// Diisi owner tiap awal tahun dari kalender resmi IDX.
module.exports = [
  // '2026-12-25',
];
```

**Kenapa `entitlementService` kamu yang menulis:** file paling penting di repo,
dipakai **kedua** agent server, coverage dipatok 100%. Kalau polanya salah dari
awal, semua fitur berbayar ikut salah.

---

## FASE 3 — Bagi Task ke Issue — ±1,5 jam

Urutkan berdasarkan ketergantungan. **Satu agent satu task per gelombang** —
setiap agent hanya punya satu worktree.

**Gelombang 1 — fondasi paralel**
| # | Issue | Agent |
|---|---|---|
| 1 | Model + migration + seeder: Subscriptions, Stocks, Alerts, PriceDailies, Analyses | Claude Code |
| 2 | `market/marketClient.js` + `mocks/marketClient.js` + `mocks/aiClient.js` + fixtures provider | Codex |
| 3 | Setup Vite + routing + halaman Login (OTP) | Cline |
| 4 | GitHub Actions CI (install, lint, test, coverage gate, **cek wilayah per awalan branch**) | Antigravity |

**Gelombang 2 — fungsi murni (tanpa DB, tanpa jaringan)**
| # | Issue | Agent |
|---|---|---|
| 5 | `helpers/price.js` + `helpers/alertParser.js` (≥60 kasus fixture) | Claude Code |
| 6 | `market/marketCalendar.js` + `market/alertRules.js` + `ai/complianceFilter.js` | Codex |
| 7 | Auth context + simpan token + protected route | Cline |
| 8 | ESLint + Prettier sesuai STYLE.md | Antigravity |

**Gelombang 3**
| # | Issue | Agent |
|---|---|---|
| 9 | `/start`, `/status`, `/bantuan` + `stockService` | Claude Code |
| 10 | `market/quoteService.js` (cache + sanity check) | Codex |
| 11 | Halaman Alerts (daftar + batalkan) | Cline |

**Gelombang 4**
| # | Issue | Agent |
|---|---|---|
| 12 | `telegram/formatter.js` — semua fungsi di `ARCHITECTURE.md §12` | Claude Code |
| 13 | `market/historyService.js` + `market/metricsService.js` | Codex |
| 14 | Halaman Overview | Cline |

**Gelombang 5**
| # | Issue | Agent |
|---|---|---|
| 15 | `alertService.create` + `/harga` + teks → alert + `/alert` | Claude Code |
| 16 | `ai/aiClient.js` + `ai/contextService.js` | Codex |
| 17 | Halaman StockDetail + grafik harga + garis target | Cline |

**Gelombang 6**
| # | Issue | Agent |
|---|---|---|
| 18 | `alertService`: `findCheckable`, `markTriggered`, `activeSymbols`, `findForDailySummary` | Claude Code |
| 19 | `jobs/stockSyncJob.js` + `bin/cron.js` | Codex |
| 20 | Halaman Upgrade + komponen `UpgradePrompt` untuk 403 | Cline |

**Gelombang 7**
| # | Issue | Agent |
|---|---|---|
| 21 | `/daftar`, `/hapus`, callback tombol, `stateStore`, alur [Geser target] | Claude Code |
| 22 | `jobs/alertCheckJob.js` (E23–E34) | Codex |
| 23 | README + `docs/DEPLOYMENT.md` | Antigravity |

**Gelombang 8**
| # | Issue | Agent |
|---|---|---|
| 24 | REST: auth OTP + `/api/me` + `/api/alerts` (CRUD + reactivate) | Claude Code |
| 25 | `jobs/eodJob.js` (E35–E36) | Codex |

**Gelombang 9**
| # | Issue | Agent |
|---|---|---|
| 26 | `subscriptionService` + checkout + webhook payment + `/upgrade` | Claude Code |

**Gelombang 10**
| # | Issue | Agent |
|---|---|---|
| 27 | REST: `/api/stocks`, quote, history, `/api/dashboard/summary` | Claude Code |
| 28 | `jobs/expiryJob.js` (E4–E5) | Codex |

**Aturan:**
- Jangan jalankan dua issue yang menyentuh file sama secara bersamaan.
- Issue yang **memakai** fungsi lintas agent dijadwalkan **setelah** issue yang
  **menyediakan** fungsi itu di-merge. Tabel di atas sudah mengikuti ini.

**Urutan ini disengaja:**
- Jalur tanpa AI (parser regex, alert harga, cek alert) dibangun **sebelum**
  `contextService`. Kalau AI dibangun duluan, agent cenderung menjadikannya
  jalur default dan tier free ikut menyentuh AI.
- `complianceFilter` dibangun **sangat awal** (gelombang 2), supaya `formatter`
  (gelombang 4) sudah bisa dites netral sejak hari pertama ditulis.
- `marketClient` + mock-nya dibangun **paling awal**, karena test Claude Code
  pun butuh mock harga.

---

## FASE 4 — Siapkan Worktree (biar 4 agent tidak timpa-menimpa)

```bash
git worktree add ../monsaham-claude -b claude/5-price-parser
git worktree add ../monsaham-codex  -b codex/6-calendar-rules
git worktree add ../monsaham-cline  -b cline/7-auth-context
git worktree add ../monsaham-anti   -b anti/8-eslint
```

Sekarang ada 4 folder terpisah, satu repo yang sama:
- `../monsaham-claude` → Claude Code
- `../monsaham-codex` → Codex (CLI atau IDE extension, jalankan dari folder ini)
- `../monsaham-cline` → VS Code window lain → Cline
- `../monsaham-anti` → Antigravity

**Paling penting untuk dua agent server:** Claude Code dan Codex **tidak boleh**
berbagi folder kerja, walaupun wilayah file mereka berbeda. `npm install`,
`node_modules`, DB test, dan `git status` akan saling mengganggu.

**DB test terpisah per worktree.** Buat dua database: `monsaham_test_claude` dan
`monsaham_test_codex`, lalu isi `DATABASE_URL_TEST` berbeda di `.env` masing-masing
worktree. Kalau dua agent menjalankan test e2e bersamaan di satu DB, `truncate`
milik yang satu menghapus data test yang lain → test gagal acak.

---

## FASE 5 — Menjalankan Satu Task (siklus yang diulang terus)

### Prompt pembuka ke agent
```
Baca SPEC.md, ARCHITECTURE.md, ERD.md, STYLE.md, TESTING.md, AGENTS.md dulu.
Lalu kerjakan issue #15.

Batasan:
- Kamu adalah <Claude Code / Codex>. Wilayahmu ada di AGENTS.md §2. Jangan
  mengubah file di luar wilayah itu, walaupun "cuma sedikit".
- Hanya boleh mengubah file yang disebut di issue #15
- Fungsi milik agent lain hanya boleh dipanggil kalau ada di ARCHITECTURE.md §12
- Tiru struktur server/services/entitlementService.js
- Jangan menambah dependency apa pun (stack dikunci di SPEC.md §9)
- TULIS TEST DULU, jalankan, tunjukkan output MERAH-nya, baru tulis kodenya
- Kalau ada yang tidak jelas di ARCHITECTURE.md, BERHENTI dan tanya saya

Sebelum menulis kode, tunjukkan dulu rencanamu:
file apa yang dibuat/diubah, fungsi apa saja di dalamnya, fungsi lintas agent
apa yang dipakai, dan daftar test yang akan kamu tulis.
```

### Alur
1. Agent kirim rencana + daftar test → **kamu baca dan approve/koreksi**
2. Agent menulis test, jalankan, tunjukkan output **MERAH**
3. Agent menulis kode sampai **HIJAU**
4. Agent jalankan `npm run test:coverage`, tunjukkan angkanya
5. `git commit` (test dulu, baru implementasi) + `git push`
6. Buka PR di GitHub
7. **Kamu review** (checklist di bawah)
8. Merge. Kalau PR membuat file yang punya ambang coverage di `TESTING.md §3`,
   tambahkan barisnya di `jest.config.js` (lihat `TESTING.md §3`)

### Checklist review (5–10 menit per PR)
- [ ] Diff-nya < 200 baris? Kalau tidak, tolak dan minta dipecah
- [ ] **Ada file di luar wilayah agent?** (CI harusnya sudah menolak, cek lagi)
- [ ] `package.json` berubah tanpa izin?
- [ ] Test ditulis duluan? (lihat urutan commit)
- [ ] Angka coverage ditempel, dan tidak turun?
- [ ] Ada `console.log` sisa debugging?
- [ ] Ada secret ter-hardcode?
- [ ] Gaya kodenya sesuai `STYLE.md`?
- [ ] **Ada pengecekan tier di luar `entitlementService`?** → tolak
- [ ] **Ada `new Date()` di dalam service atau job?** → tolak
- [ ] **Ada jalur di mana tier free bisa memanggil AI atau membaca `Analyses`?** → tolak
- [ ] **Ada float untuk harga atau persen?** → tolak
- [ ] **Ada UPDATE status alert tanpa `WHERE status = 'active'`?** → tolak
- [ ] **Ada panggilan provider di dalam loop per alert?** → tolak
- [ ] **Baca semua kalimat baru yang dilihat user. Ada yang terdengar seperti
      saran beli/jual?** → tolak
- [ ] Test-nya benar-benar menguji sesuatu, atau cuma `expect(200)`?
- [ ] **Bisakah kamu jelaskan alur kode ini ke orang lain?** Kalau tidak, jangan merge.

---

## FASE 6 — Setelah Merge

```bash
git checkout main && git pull
git worktree remove ../monsaham-codex
git worktree add ../monsaham-codex -b codex/10-quote-service
```

Selalu `git pull` sebelum task berikutnya, dan **beri tahu agent** bahwa `main`
sudah berubah. Ini dua kali lebih penting di project ini: fungsi yang baru
di-merge Claude Code bisa jadi yang ditunggu Codex, dan sebaliknya.

---

## FASE 7 — Menjalankan Bot Secara Lokal

Butuh **tiga proses**: web server, cron, dan tunnel.

```bash
# terminal 1 — web server (webhook + REST)
cd server && npm run dev

# terminal 2 — cron (cek alert, EOD, expiry)
cd server && npm run cron

# terminal 3 — tunnel
npx cloudflared tunnel --url http://localhost:3000

# terminal 4 — daftarkan webhook (pakai bot DEV)
curl -F "url=https://xxx.trycloudflare.com/webhook/telegram" \
     -F "secret_token=$TELEGRAM_WEBHOOK_SECRET" \
     "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/setWebhook"
```

**Uji di akhir pekan / malam hari:** set `MARKET_FORCE_OPEN=true` di `.env` lokal
supaya `alertCheckJob` tetap jalan di luar jam bursa (harga yang didapat adalah
penutupan terakhir). Diabaikan kalau `NODE_ENV=production`.

Kalau bot diam saja: `getWebhookInfo` (ada error?) → log server (request masuk?)
→ header secret token (cocok?) → tier user.

Kalau alert tidak pernah kena: log cron (`job.failed`?) → `isInSession` (di luar
jam?) → `market.quote.suspicious` di log? → status alert di DB masih `active`?
→ tier user free dan alert-nya di luar 2 tertua?

---

## FASE 8 — Deploy

- Backend → Railway/Render, DB → PostgreSQL managed
- Frontend → Vercel, `VITE_API_URL` dari env
- Jalankan migration lewat script, bukan `sequelize.sync({ force: true })`
- **Cron jalan sebagai proses terpisah, tepat SATU instance.** Kalau cron
  ikut di-scale jadi 2, setiap putaran jalan dua kali. UPDATE bersyarat mencegah
  alert dobel, tapi biaya provider dan AI tetap dobel.
- **Zona waktu server tidak boleh dipercaya.** Semua jadwal pakai
  `timezone: 'Asia/Jakarta'` di `node-cron`, semua jam bursa lewat `marketCalendar`.
- `MARKET_FORCE_OPEN` **tidak boleh ada** di env production.
- Setelah deploy: `setWebhook` ke URL production, lalu smoke test dari akun uji:
  `/status` harus dibalas < 5 detik, `/harga BBCA` di jam bursa harus
  menampilkan jam data hari ini.
- Antigravity yang menulis dokumentasi deploy di `docs/DEPLOYMENT.md`

---

## Tanda Bahaya (agent mulai halu)

Hentikan dan reset sesi kalau melihat:

- Menyebut nama file atau fungsi yang tidak ada di repo
- Menambah library yang tidak diminta
- Bilang "sudah saya test" tanpa menunjukkan output
- Mengubah isi test supaya lulus
- PR tiba-tiba 800 baris padahal task-nya kecil
- Mengedit file di luar wilayahnya
- Menjawab "sudah sesuai spesifikasi" tanpa menyebut bagian mana

**Tanda bahaya khusus project ini:**

- Claude Code membuat file di `market/`, `ai/`, atau `jobs/` — atau Codex
  membuat model, migration, atau mengubah `formatter.js`
- Membuat "versi sementara" fungsi milik agent lain karena belum di-merge
- Membuat kolom `plan` di tabel `Users`
- Prompt AI yang berisi deret harga mentah, kata "hitung", atau pertanyaan
  "apakah layak dibeli"
- Kalimat bot atau UI dengan kata "murah", "saatnya", "rekomendasi", "potensi"
- `parseFloat` atau `* 0.9` di kode harga
- `alert.save()` setelah mengubah status, alih-alih UPDATE bersyarat
- `fetch` ke URL data saham di luar `marketClient.js`
- `new Date()` di dalam service atau job
- Menurunkan angka di `coverageThreshold` supaya CI hijau
- Test yang isinya cuma `expect(response.status).toBe(200)`

Cara reset: mulai sesi baru, suruh baca ulang `SPEC.md` + `ARCHITECTURE.md` +
file yang relevan, dan kerjakan ulang dari `main` yang bersih.

---

## Prioritas Waktu Reviewmu

Tidak semua kode setara. Baca baris-per-baris untuk:

1. `services/entitlementService.js` dan `middlewares/entitlement.js` — menjaga pendapatan
2. `telegram/formatter.js`, `ai/complianceFilter.js`, dan prompt di
   `ai/contextService.js` — menjaga dari risiko hukum
3. `market/alertRules.js` dan `alertService.markTriggered` — menjaga alert
   tidak palsu dan tidak dobel
4. `market/marketClient.js`, `ai/aiClient.js`, dan semua pemanggilnya — ini
   yang menghabiskan uang
5. `middlewares/telegramAuth.js` dan webhook payment — pintu masuk dari luar
6. Apa pun yang menyentuh `.env` atau query database

Sisanya (form React, styling, halaman statis) cukup dibaca cepat.
