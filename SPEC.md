# SPEC.md — Chatbot Alert Harga Saham

> Sumber kebenaran tunggal untuk SEMUA agent (Claude Code, Codex, Cline, Antigravity).
> Kalau sesuatu tidak tertulis di sini → **berhenti dan tanya owner**, jangan mengarang.

---

## 1. Tujuan Project

Chatbot Telegram yang **mengabari user saat harga saham Indonesia (IDX)
menyentuh target yang user tentukan sendiri**. User cukup kirim pesan biasa
("kabarin BBCA kalau 3000"), bot memantau harganya selama jam bursa, lalu
mengirim notifikasi begitu target tersentuh. Ada dashboard web untuk melihat
daftar alert dan grafik harga.

Bedanya dengan alert di aplikasi sekuritas: tidak perlu buka aplikasi, cukup
chat. Bisa pakai kondisi yang lebih manusiawi ("turun 10%", "di bawah rata-rata
bulan lalu"), dan saat alert kena, member dapat konteks singkat pergerakan
harganya.

**Ini BUKAN bot sinyal atau rekomendasi.** Target selalu ditentukan user. Bot
hanya menyampaikan fakta harga. Lihat aturan bahasa di §5 no. 1–2.

**Alur inti:** pesan Telegram → parsing → alert tersimpan → cron cek harga →
alert kena → kirim notifikasi. Semua fitur lain adalah turunan dari data alert
dan harga ini.

## 2. Scope MVP (v1.0)

Yang **MASUK** v1.0:

- Registrasi otomatis lewat `/start` + free trial 7 hari
- Cek harga saham (`/harga BBCA` atau cukup kirim `BBCA`), data tertunda ±15 menit
- Buat alert dari teks bahasa natural ("bbca 3000", "tlkm turun 10%")
- 5 jenis alert: `price_below`, `price_above`, `pct_drop`, `pct_rise`, `below_avg_30d`
- Validasi fraksi harga (tick size) IDX saat membuat alert
- Pengecekan alert otomatis tiap 5 menit selama sesi perdagangan
- Notifikasi alert + tombol inline [Aktifkan lagi] [Geser target]
- Konteks AI singkat saat alert kena (trial, member), di-cache per saham per hari
- Ringkasan harian setelah bursa tutup: posisi harga vs target (trial, member, tanpa AI)
- 3 tier: `trial` / `member` / `free` dengan pembatasan fitur
- Langganan member 30 hari + webhook pembayaran
- Dashboard web: login OTP via Telegram, overview, daftar alert, grafik harga

Yang **TIDAK MASUK** v1.0 (jangan dikerjakan sampai diminta):

- Rekomendasi, sinyal, skor, "saham pilihan", target harga dari bot — **tidak akan pernah**
- Data real-time (v1.0 hanya data tertunda)
- Alert volume, alert 52-minggu, alert berita/aksi korporasi
- Jurnal / pencatatan portofolio, hitung untung-rugi
- Kalender dividen
- Integrasi akun sekuritas / order beli-jual
- Saham luar negeri, kripto, reksa dana, waran, right issue
- Grup Telegram (v1.0 hanya chat privat)
- Mobile app native
- Dark mode

## 3. Definisi Istilah

| Istilah | Arti di project ini |
|---|---|
| Ticker / Kode saham | 4 huruf kapital kode emiten IDX, mis. `BBCA`. Regex: `^[A-Z]{4}$` |
| Alert | Satu permintaan user: "kabari saya kalau kondisi X terpenuhi untuk saham Y". **Sekali kena lalu mati** (status `triggered`). |
| Target | Harga yang dibandingkan dengan harga pasar. Untuk `pct_*` dihitung sekali saat alert dibuat. Untuk `below_avg_30d` dihitung ulang tiap hari. |
| Base price | Harga pasar saat alert dibuat. Disimpan, tidak pernah diubah. |
| Quote | Harga terakhir satu saham dari provider data, beserta `asOf` (jam data itu). |
| Harga | **Integer rupiah per lembar.** `3000`, bukan `3000.00`. IDX tidak punya harga desimal. |
| Basis poin (bps) | Cara menyimpan persentase tanpa float. `1% = 100 bps`, `10% = 1000 bps`. |
| Fraksi harga (tick) | Kelipatan harga yang sah di IDX, tergantung rentang harga. Lihat §8. |
| Sesi perdagangan | Jam bursa aktif. Di luar sesi, harga tidak dicek. Lihat `ARCHITECTURE.md §9`. |
| Tier | Hak akses user saat ini: `trial`, `member`, atau `free`. **Dihitung**, bukan disimpan. |
| Metrics | Angka yang dihitung JS (bukan AI) dari data harga, dipakai sebagai input konteks AI. |
| Konteks | Narasi AI 1–2 kalimat tentang pergerakan harga. Fitur berbayar. Bukan saran. |

## 4. Tiga Tier (ini inti bisnisnya, jangan sampai salah)

| | `trial` | `member` | `free` |
|---|:---:|:---:|:---:|
| Durasi | 7 hari sejak `/start` | 30 hari per pembelian | selamanya |
| Cek harga (`/harga`) | ✅ | ✅ | ✅ |
| Alert `price_below` / `price_above` | ✅ | ✅ | ✅ |
| Alert `pct_drop` / `pct_rise` | ✅ | ✅ | ❌ |
| Alert `below_avg_30d` | ✅ | ✅ | ❌ |
| Maks. alert aktif yang dipantau | 50 | 50 | **2** |
| Konteks saat alert kena (📊 angka JS + 💡 narasi AI) | ✅ | ✅ | ❌ |
| Ringkasan harian | ✅ | ✅ | ❌ |
| Dashboard web | ✅ | ✅ | ❌ |

Angka batas ada di `config/constants.js`: `FREE_ALERT_LIMIT = 2`,
`MEMBER_ALERT_LIMIT = 50`.

**Cara menentukan tier (deterministik, jangan dikarang):**

```
kalau ada Subscription status 'active' dan expiresAt > now  → 'member'
kalau tidak, tapi now < user.trialEndsAt                    → 'trial'
selain itu                                                  → 'free'
```

**Saat tier turun ke `free`:** alert **tidak diubah dan tidak dihapus**. Cron hanya
memantau `FREE_ALERT_LIMIT` alert `price_below`/`price_above` yang paling lama
dibuat. Alert lainnya tetap `active` di DB tapi dilewati, dan di `/daftar` ditandai
"⏸ tidak dipantau". Begitu user upgrade, semuanya otomatis dipantau lagi, tanpa
migrasi data apa pun.

## 5. Aturan Bisnis (WAJIB — ini yang paling sering di-halu-kan agent)

1. **Bot tidak pernah menyarankan beli atau jual.** Tidak ada kata "murah",
   "mahal", "saatnya beli", "layak dibeli", "rekomendasi", "target harga" (sebagai
   saran dari bot), "undervalued", "cuan", "pasti naik". Bot menyampaikan fakta:
   harga, perubahan %, posisi terhadap target user. Pelanggaran aturan ini
   berisiko hukum (izin penasihat investasi OJK), bukan sekadar soal gaya.
2. **Output AI wajib lewat `ai/complianceFilter.js`.** Kalau mengandung frasa
   terlarang (daftar di `config/constants.js` → `BANNED_PHRASES`), **seluruh
   konteks dibuang** dan notifikasi dikirim tanpa konteks. Jangan coba menyunting
   kalimat AI.
3. **Tier tidak disimpan sebagai kolom.** Tidak ada kolom `plan` di tabel `Users`.
   Tier selalu dihitung ulang lewat `services/entitlementService.js`.
4. **Semua pengecekan hak akses lewat `entitlementService`.** Dilarang menulis
   `if (user.plan === 'member')` di controller, route, handler, atau job mana pun.
5. **User `free` TIDAK PERNAH menyentuh API AI.** Tidak ada konteks, tidak ada
   ringkasan, bahkan kalau konteks untuk saham itu sudah ada di cache (cache hit
   pun tetap tidak ditampilkan ke free).
6. **AI tidak pernah menghitung angka.** Semua harga, selisih, persentase, dan
   rata-rata dihitung JavaScript. AI cuma dikasih `metrics` yang sudah jadi.
7. **Harga selalu integer rupiah. Persentase selalu integer basis poin.**
   Dilarang float untuk harga maupun persentase yang disimpan atau dibandingkan.
   Float hanya boleh muncul di `formatter` saat menampilkan "8,3%".
8. **Parsing pesan user pakai regex (`helpers/alertParser.js`) untuk SEMUA tier.**
   v1.0 tidak memakai AI untuk parsing. AI hanya untuk konteks.
9. **Target harus sah saat alert dibuat:**
   - Kelipatan fraksi harga (tick) IDX. Tidak sah → tolak, tawarkan 2 harga sah
     terdekat.
   - Belum terlewati. `price_below` dengan target ≥ harga sekarang → tolak dengan
     info harga sekarang. Sama untuk `price_above` (target ≤ harga sekarang) dan
     `below_avg_30d` (harga sekarang sudah di bawah rata-rata).
   - Ticker ada di tabel `Stocks` dan `isActive = true` dan `isIndex = false`.
10. **Alert sekali kena lalu mati.** Begitu kena → status `triggered`, tidak dicek
    lagi. [Aktifkan lagi] membuat **baris alert baru** dengan `basePrice` baru,
    bukan mengubah baris lama.
11. **Alert tidak boleh terkirim dua kali.** Perubahan `active → triggered` wajib
    lewat UPDATE bersyarat `WHERE status = 'active'`. Notifikasi hanya dikirim
    kalau UPDATE itu mengubah tepat 1 baris.
12. **Kolom target alert tidak pernah di-UPDATE.** `type`, `targetPrice`,
    `targetPctBps`, `basePrice`, `StockId` tetap seumur hidup. [Geser target] =
    batalkan alert lama (`cancelled`) + buat alert baru dengan
    `reactivatedFromId`.
13. **Harga yang mencurigakan tidak dipakai.** Quote yang berubah lebih dari
    `QUOTE_SANITY_MAX_BPS` (35%) dari penutupan kemarin, atau `asOf`-nya bukan
    hari bursa ini → dilewati dan dicatat di log. Alert palsu lebih buruk daripada
    alert terlambat.
14. **Setiap pesan yang menampilkan harga wajib menyebut jam data dan bahwa data
    tertunda**, mis. "Per 10:15 WIB · data tertunda ±15 menit".
15. **Kalau AI gagal/timeout → notifikasi alert tetap dikirim** tanpa konteks.
    Kalau provider data gagal → cron mencoba lagi di putaran berikutnya, tidak
    ada alert yang ditandai `triggered` tanpa harga yang sah.
16. **Webhook Telegram wajib divalidasi** pakai header
    `X-Telegram-Bot-Api-Secret-Token`. Tidak cocok → 401, jangan proses apa pun.
17. **Webhook idempoten.** `update_id` yang sama tidak boleh menghasilkan dua alert.
18. **Balas 200 ke Telegram secepatnya**, proses setelah itu.
19. **Trial sekali seumur hidup per `telegramId`.** `/start` kedua kali tidak
    memperpanjang trial dan tidak membuat user baru.
20. **Password (kalau user pakai login email) tidak pernah masuk response API
    atau log.**

## 6. Perintah Telegram v1.0

| Command | Tier | Fungsi |
|---|---|---|
| `/start` | semua | Registrasi + mulai trial 7 hari + penjelasan singkat "bukan rekomendasi" |
| `/bantuan` | semua | Contoh format alert + disclaimer |
| `/status` | semua | Tier aktif, sisa hari, jumlah alert aktif vs batas |
| `/harga <KODE>` | semua | Harga terakhir, perubahan hari ini, jam data |
| `/alert <KODE> <kondisi>` | semua* | Buat alert secara eksplisit (format sama dengan §8) |
| `/daftar` | semua | Daftar alert aktif + tombol [Hapus] per alert |
| `/hapus <id>` | semua | Batalkan alert (status `cancelled`) |
| `/ringkasan on\|off` | trial, member | Nyalakan/matikan ringkasan harian |
| `/upgrade` | semua | Paket member + link pembayaran |
| `/batal` | semua | Batalkan input yang menggantung (mis. setelah [Geser target]) |

\* Jenis alert yang boleh dibuat mengikuti tabel tier §4.

Pesan tanpa command diproses oleh `alertParser`: kode saham saja → sama dengan
`/harga`. Kode saham + kondisi → buat alert.

## 7. Contoh Balasan Bot (tiru gaya ini)

**Alert dibuat:**
```
✓ Alert dibuat: BBCA ≤ Rp 3.000
Harga sekarang Rp 3.120 (kurang 3,9% lagi)
Per 10:15 WIB · data tertunda ±15 menit

Alert aktif: 3 dari 50
```

**Alert kena, tier `trial` / `member`:**
```
🔔 BBCA menyentuh target kamu
Harga Rp 2.990 · target ≤ Rp 3.000
Per 10:15 WIB · data tertunda ±15 menit

📊 5 hari bursa: −8,3% · rata-rata 30 hari Rp 3.210
💡 BBCA ditutup melemah di 4 dari 5 hari terakhir, sementara IHSG
di periode yang sama naik tipis 0,4%.

ℹ️ Pengingat harga, bukan saran investasi.
[Aktifkan lagi] [Geser target]
```
Baris 📊 dihitung JS dan selalu segar. Baris 💡 adalah konteks AI (cache per
saham per hari).

**Alert kena, tier `free`:**
```
🔔 BBCA menyentuh target kamu
Harga Rp 2.990 · target ≤ Rp 3.000
Per 10:15 WIB · data tertunda ±15 menit

🔒 Konteks pergerakan harga tersedia untuk Member.
Ketik /upgrade untuk buka akses 30 hari.
[Aktifkan lagi] [Geser target]
```

**Free mencoba alert ke-3:**
```
Alert aktif kamu sudah 2 dari 2 (batas tier Free).
Hapus salah satu lewat /daftar, atau /upgrade untuk sampai 50 alert.
```

**Target tidak sesuai fraksi harga:**
```
Harga 3.003 tidak sah untuk BBCA (fraksi Rp 10 di rentang ini).
Pilih salah satu: [3.000] [3.010]
```

**Ringkasan harian (trial/member, tanpa AI):**
```
📋 Penutupan 23 Sep · data tertunda ±15 menit
BBCA Rp 3.120 · target ≤ 3.000 · kurang 3,9%
TLKM Rp 3.450 · target turun 10% (≤ 3.240) · kurang 6,1%
BBRI Rp 4.350 · target di bawah rata-rata 30 hari (Rp 4.260) · kurang 2,1%
```

Aturan gaya pesan: netral, faktual, maksimal 3 emoji, maksimal ~12 baris.
❌ "BBCA lagi murah nih, momen bagus!" → ✅ "BBCA Rp 2.990, turun 8,3% dalam 5 hari bursa."

## 8. Format Input yang Wajib Dikenali Parser

| Input user | Hasil |
|---|---|
| `bbca` | tampilkan harga BBCA (sama dengan `/harga BBCA`) |
| `bbca 3000` | alert harga; arah otomatis: target < harga sekarang → `price_below`, target > harga sekarang → `price_above`, sama → tolak |
| `kabarin bbca kalau 3.000` | sama dengan baris di atas (kata pengisi diabaikan) |
| `bbca <= 3000` / `bbca di bawah 3rb` | `price_below`, 3000 |
| `tlkm >= 4000` / `tlkm di atas 4rb` | `price_above`, 4000 |
| `tlkm turun 10%` | `pct_drop`, 1000 bps |
| `antm naik 15%` | `pct_rise`, 1500 bps |
| `bbri bawah rata2` / `bbri di bawah rata-rata bulan lalu` | `below_avg_30d` |
| `bbca 3003` | tolak — tidak sesuai fraksi harga, tawarkan 3.000 / 3.010 |
| `xxxx 3000` (ticker tidak dikenal) | tolak — "Kode XXXX tidak ditemukan" |
| `beli bbca` / `bagus gak bbca?` | balasan netral: bot hanya memberi pengingat harga, bukan saran |

Normalisasi: ticker → huruf kapital. `rb`/`ribu`/`k` → ×1.000. Titik/koma
pemisah ribuan dibuang ("3.000" → 3000). Persen dibulatkan ke basis poin
integer ("10%" → 1000, "2,5%" → 250). Hasil akhir selalu integer.

**Fraksi harga IDX** (dipakai untuk validasi target; tabel di
`config/constants.js` → `TICK_SIZE_TABLE`, **owner wajib cek ulang ke
peraturan IDX terbaru sebelum rilis**):

| Rentang harga | Fraksi |
|---|---|
| < Rp 200 | Rp 1 |
| Rp 200 – < Rp 500 | Rp 2 |
| Rp 500 – < Rp 2.000 | Rp 5 |
| Rp 2.000 – < Rp 5.000 | Rp 10 |
| ≥ Rp 5.000 | Rp 25 |

**Target untuk alert persen** dihitung sekali saat dibuat, lalu dibulatkan ke
fraksi yang sah:
- `pct_drop`: `basePrice × (10000 − bps) / 10000`, dibulatkan **ke bawah**
- `pct_rise`: `basePrice × (10000 + bps) / 10000`, dibulatkan **ke atas**

## 9. Tech Stack (DIKUNCI — dilarang menambah/mengganti tanpa izin)

- Backend: Node.js 20, Express 4, Sequelize 6, PostgreSQL
- Telegram: `node-telegram-bot-api` (mode webhook, bukan polling)
- Auth: `jsonwebtoken`, `bcryptjs`
- Data saham: HTTP API provider pakai **`fetch` bawaan Node 20**, hanya lewat
  `market/marketClient.js`. Dilarang menambah SDK / library scraping.
- AI: HTTP API provider pakai **`fetch` bawaan Node 20**, hanya lewat
  `ai/aiClient.js`. Dilarang menambah SDK.
- Cron: `node-cron` (dengan opsi `timezone: 'Asia/Jakarta'`)
- Cache harga: **`Map` in-memory** dengan TTL di `market/quoteService.js`.
  **Bukan** Redis di v1.0 — satu instance dulu.
- State input menggantung: **`Map` in-memory** dengan TTL di `helpers/stateStore.js`.
- Test: Jest + Supertest
- Frontend: React 18 + Vite, React Router, Axios
- Grafik: `recharts` — hanya ini, dan hanya untuk grafik
- Styling: CSS biasa / CSS Modules — **bukan** Tailwind, **bukan** UI library
- State frontend: React state + Context. **Bukan** Redux/Zustand/React Query di v1

## 10. Definition of Done (satu task dianggap selesai kalau...)

- [ ] Kode jalan lokal tanpa error
- [ ] Test ditulis **sebelum** kode (lihat `TESTING.md`), dan `npm test` hijau
- [ ] Coverage memenuhi ambang di `TESTING.md §3`
- [ ] Tidak melanggar `STYLE.md`
- [ ] Tidak menambah dependency baru
- [ ] File yang disentuh hanya yang disebut di issue **dan** ada di wilayah agent
      tersebut (`AGENTS.md §2`)
- [ ] Ada bukti output terminal di deskripsi PR
