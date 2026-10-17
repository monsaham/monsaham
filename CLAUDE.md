# AGENTS.md — Aturan Main untuk Semua AI Agent

> Berlaku untuk Claude Code, Codex, Cline, dan Antigravity.
> Codex dan Antigravity membaca file ini langsung (`AGENTS.md`).
> Salin file ini juga sebagai `CLAUDE.md` (Claude Code) dan
> `.clinerules/00-rules.md` (Cline). Isinya harus identik.

---

## 0. Baca dulu sebelum menulis kode apa pun

Urutan wajib di awal setiap sesi:
1. `SPEC.md` — apa yang dibangun & aturan bisnis
2. `ARCHITECTURE.md` — schema, endpoint, kepemilikan folder, **kontrak fungsi §12**
3. `ERD.md` — relasi & aturan integritas data (kalau task-nya menyentuh DB)
4. `STYLE.md` — gaya penulisan kode
5. `TESTING.md` — cara membuktikan task-nya selesai
6. File yang akan diubah — **baca isinya, jangan menebak**

`PRD.md` dibaca kalau kamu harus memutuskan sesuatu yang tidak tertulis di `SPEC.md`.

## 0b. Patokan Service (wajib dibaca sebelum menulis service baru)

Sebelum menulis service baru, baca `server/services/entitlementService.js` dan
`server/__tests__/unit/entitlementService.test.js`, lalu tiru strukturnya persis:

- Class dengan static method
- `now` diterima sebagai parameter, bukan `new Date()` di dalam service
- Throw object bernama (`{ name, message }`), ditangani `errorHandler`
- Test ditulis dulu, coverage sesuai ambang `TESTING.md §3`

Berlaku untuk **kedua** agent server. File di `market/`, `ai/`, dan `jobs/` milik
Codex juga mengikuti pola ini.

## 1. Anti-Halusinasi (aturan paling penting)

- **Dilarang menebak isi file.** Selalu baca file dulu sebelum mengedit.
- **Dilarang mengarang nama fungsi, kolom DB, endpoint, atau field response.**
  Kalau tidak ada di `ARCHITECTURE.md` → berhenti dan tanya.
- **Dilarang menambah dependency baru.** Termasuk library "kecil" seperti
  `lodash`, `moment`, `dayjs`, `uuid`, `axios` di server, SDK AI, atau library
  data saham / scraping. Stack sudah dikunci di `SPEC.md §9`. Kalau merasa butuh
  → tanya dulu.
- **Dilarang mengklaim "sudah jalan" tanpa bukti.** Sertakan output terminal
  (`npm test`, `curl`, log) di deskripsi PR.
- **Dilarang menulis kode untuk fitur yang ada di daftar "TIDAK MASUK v1.0"**
  di `SPEC.md`, meskipun kelihatan berguna.
- Kalau ada dua cara dan spesifikasi ambigu → **jangan pilih sendiri**, tanya.
- Kalau sebuah test gagal, **jangan mengubah test-nya supaya hijau.**
  Perbaiki kodenya, atau laporkan kalau memang spesifikasinya yang salah.

## 1b. Anti-Halusinasi Khusus Project Ini

Tujuh hal ini paling sering dilanggar agent. Hafalkan.

1. **Jangan menulis kalimat yang terdengar seperti saran beli/jual.** Di
   formatter, di prompt AI, di halaman dashboard, di pesan error. Tidak ada
   "murah", "saatnya beli", "rekomendasi", "cuan". Lihat `SPEC.md §5` no. 1.
2. **Jangan bikin kolom `plan` di tabel `Users`.** Tier dihitung, tidak disimpan.
3. **Jangan menyuruh AI menghitung.** Kalau prompt yang kamu tulis berisi deret
   harga mentah dan kata "hitung", "rata-rata", "berapa persen" — berhenti.
   Angka dihitung `metricsService`.
4. **Jangan memanggil API AI untuk tier `free`.** Bahkan kalau konteksnya sudah
   ada di cache.
5. **Jangan pakai float untuk harga atau persentase.** Harga = integer rupiah,
   persen = integer basis poin. `0.1` untuk 10% → salah. `1000` → benar.
6. **Jangan UPDATE kolom target alert, dan jangan UPDATE status tanpa
   `WHERE status = 'active'`.** Alert dobel = bug paling memalukan di produk ini.
7. **Jangan menaruh `new Date()` di dalam service atau job.** Pakai
   `dateHelper.now()`, supaya test jam bursa, trial 7 hari, dan expiry 30 hari
   bisa ditulis.

Kalau kamu terlanjur menulis salah satu dari tujuh ini, **jangan diam-diam
diperbaiki** — sebutkan di deskripsi PR.

## 2. Batas Wilayah Kerja

| Agent | Boleh mengubah | Dilarang menyentuh |
|---|---|---|
| **Claude Code** | `server/app.js`, `server/bin/www.js`, `server/config/config.js`, `server/models/**`, `server/migrations/**`, `server/seeders/**`, `server/telegram/**`, `server/services/**`, `server/controllers/**`, `server/routes/**`, `server/middlewares/**`, `server/helpers/**`, `server/package.json`, test milik Claude Code (`TESTING.md §4`) | `server/market/**`, `server/ai/**`, `server/jobs/**`, `server/bin/cron.js`, `client/**`, `.github/**` |
| **Codex** | `server/market/**`, `server/ai/**`, `server/jobs/**`, `server/bin/cron.js`, test milik Codex (`TESTING.md §4`) | semua folder `server/` milik Claude Code, `client/**`, `.github/**` |
| **Cline** | `client/**` | `server/**`, `.github/**` |
| **Antigravity** | `.github/**`, `docs/**`, `README.md`, file konfigurasi root | `server/**`, `client/**` |

`server/config/constants.js`, `server/config/marketHolidays.js`,
`server/jest.config.js`, `SPEC.md`,
`PRD.md`, `ARCHITECTURE.md`, `ERD.md`, `STYLE.md`, `TESTING.md`, `AGENTS.md`,
`WORKFLOW.md` hanya boleh diubah oleh manusia.

### 2b. Aturan khusus dua agent di `server/`

- **Memanggil fungsi agent lain boleh, mengeditnya tidak.** Fungsi yang boleh
  dipanggil lintas agent hanya yang ada di `ARCHITECTURE.md §12`.
- **Fungsi lintas agent belum ada?** Jangan membuat versi sementara di folder
  sendiri, jangan membuat file di folder agent lain. Di test, mock fungsinya
  sesuai signature di §12. Di kode, tulis `ASUMSI:` di PR dan tunggu issue
  pasangannya di-merge.
- **Butuh konstanta baru?** Jangan menulis angka langsung. Minta owner
  menambahkannya ke `config/constants.js` lewat issue.
- **Butuh model / migration baru?** Hanya Claude Code yang menulis. Codex
  membuka issue.
- **Mengubah signature fungsi di §12 = breaking change.** Tidak boleh dilakukan
  agent. Owner mengubah `ARCHITECTURE.md` dulu, lalu dua issue (satu per agent).

## 3. Ukuran Task

- Satu task = satu GitHub Issue = satu branch = satu PR.
- **Maksimal ~200 baris perubahan per PR** (di luar test dan lockfile). Kalau
  lebih besar, pecah dulu dan laporkan rencana pemecahannya.
- Untuk task non-trivial: kirim **rencana singkat dulu** (file apa yang diubah,
  fungsi apa yang dibuat), tunggu approval owner, baru menulis kode.

## 4. Alur Git

```bash
git checkout main && git pull
git checkout -b claude/5-alert-parser
# ...kerja...
git add -A
git commit -m "feat(bot): parsing teks jadi perintah alert refs #5"
git push -u origin claude/5-alert-parser
```

- **Dilarang commit langsung ke `main`.**
- **Dilarang `git push --force`, `git rebase`, atau menghapus branch orang lain.**
- **Dilarang mengubah `package-lock.json`** kecuali memang task-nya soal dependency.
- Commit message: `type(scope): deskripsi refs #issue`
  (`type` = feat / fix / test / docs / chore / refactor)
- Scope yang dipakai: `bot`, `api`, `db`, `market`, `ai`, `jobs`, `client`, `ci`, `docs`
- **Nama branch wajib diawali nama agent**, karena CI memakai awalan ini untuk
  mengecek batas wilayah (§2):
  `claude/<issue>-<nama>`, `codex/<issue>-<nama>`, `cline/<issue>-<nama>`,
  `anti/<issue>-<nama>`. Branch perubahan dokumen oleh owner: `docs/<nama>`.

**Khusus project ini:** commit `test:` ditulis **sebelum** commit `feat:` yang
membuatnya lulus. Riwayat commit adalah bukti bahwa TDD-nya dijalankan.

## 5. Format Laporan Selesai

Setiap PR wajib berisi:

```md
## Apa yang dikerjakan
- ...

## Agent
Claude Code / Codex / Cline / Antigravity

## File yang diubah
- server/helpers/alertParser.js (baru)
- server/__tests__/unit/alertParser.test.js (baru)

## Fungsi lintas agent yang dipakai / disediakan (ARCHITECTURE.md §12)
- memakai: — (parser adalah fungsi murni)
- menyediakan: —

## Bukti jalan
$ npm test
 PASS  __tests__/unit/alertParser.test.js
 Tests: 52 passed

$ npm run test:coverage
 All files | 88.2 | 84.1 | 90.0 | 88.4

## Yang TIDAK saya kerjakan / ragu
- Belum menangani "bbca 3000 atau 2900" (dua target sekaligus) karena tidak ada di SPEC.md
```

Bagian terakhir wajib diisi jujur. Menulis "tidak ada" padahal ada keraguan
dianggap pelanggaran berat.

## 6. Testing

Detailnya di `TESTING.md`. Yang wajib diingat:

- **Test ditulis duluan.** Jalankan, pastikan MERAH, baru tulis kodenya.
- Setiap endpoint minimal punya: 1 test sukses, 1 test validasi gagal (400),
  1 test tanpa token (401), 1 test tier salah (403).
- Setiap handler bot dan setiap job minimal punya: 1 test tier `trial`/`member`,
  1 test tier `free`. Test tier `free` wajib memverifikasi **AI tidak dipanggil
  sama sekali**.
- Setiap test yang menyentuh jam bursa wajib membekukan waktu (`TESTING.md §6`).
- Test pakai database test terpisah, dibersihkan di `afterEach`.
- API AI, API data saham, dan Telegram **selalu di-mock** di test. Tidak boleh
  ada panggilan jaringan sungguhan di CI.
- Dilarang memakai `.skip`, `.only`, atau `--forceExit` untuk melewati test.

## 7. Keamanan

- Password selalu di-hash bcrypt (salt rounds 10) lewat hook Sequelize.
- Password dan hash tidak pernah masuk response API atau log.
- Semua secret (`JWT_SECRET`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`,
  `AI_PROVIDER_API_KEY`, `MARKET_API_KEY`, kredensial DB) dari `.env`.
  **Dilarang hardcode.**
- `.env` tidak pernah di-commit. Update `.env.example` kalau ada variabel baru
  (lewat owner).
- Query selalu lewat Sequelize (parameterized). Dilarang string concat SQL.
- Webhook Telegram wajib cek `X-Telegram-Bot-Api-Secret-Token`; webhook payment
  wajib cek signature.
- Callback tombol dan endpoint alert wajib cek **alert milik user yang meminta**.
- **Isi pesan user (`rawMessage`) tidak boleh masuk log.** Boleh disimpan di DB
  untuk audit, tidak boleh di `console.log` / `console.error`.
- `MARKET_API_KEY` tidak boleh ikut ke response, log, atau pesan error ke user.

## 8. Kalau Kamu Ragu

Urutannya:
1. Cari jawabannya di `SPEC.md` → `ARCHITECTURE.md` → `ERD.md`.
2. Masih tidak ada? **Berhenti dan tanya owner.** Jangan pilih sendiri.
3. Kalau owner tidak ada dan kamu terpaksa memilih: pilih yang **paling tidak
   terdengar seperti saran investasi**, **paling murah biaya data & AI-nya**, dan
   **paling mudah dites**, lalu tulis di deskripsi PR dengan awalan `ASUMSI:`.

Jangan pernah: menghapus test yang gagal, menurunkan ambang coverage, mengubah
bentuk response supaya test lulus, mengedit file agent lain "karena cuma
sedikit", atau membuat "solusi sementara" yang tidak dicatat di PR.
