# PRD — Mon Saham
**Product Requirements Document | Draft v2.0 | 14 Oktober 2026**

**Status:** Usulan revisi untuk review owner, **belum disahkan**  
**Dasar:** `PRD(2).md`, `SPEC.md`, dan pembahasan peran website/AI.  
**Batas perubahan:** `WORKFLOW.md` **tidak boleh diubah**. Dokumen ini tidak mengesahkan perubahan API, ERD, pembagian agent, urutan issue, maupun dependency implementasi.

> **Aturan sumber kebenaran:** PRD menjawab *mengapa* produk dibangun dan hasil apa yang diharapkan. `SPEC.md` menetapkan perilaku dan scope teknis yang sudah disetujui. Jika ada konflik, **jangan diam-diam mengimplementasikan usulan baru**: ajukan keputusan owner dan sinkronkan SPEC/ARCHITECTURE/ERD terlebih dahulu. `WORKFLOW.md` tetap utuh.

## 1. Ringkasan Eksekutif

**Mon Saham** adalah asisten pemantauan target harga saham Indonesia (IDX) berbasis Telegram. Pengguna menentukan sendiri saham dan kondisi harga, kemudian bot memeriksa harga selama sesi perdagangan dan mengirim notifikasi ketika kondisi terpenuhi. Dashboard web melengkapi Telegram untuk melihat kumpulan alert dan grafik harga dengan lebih nyaman.

**Nilai inti:** *set an alert as easily as sending a chat, then understand what happened when it triggers.*

Produk **bukan** broker, aplikasi transaksi, pemberi sinyal, atau penasihat investasi. Tidak menawarkan rekomendasi beli/jual. MVP memakai **data tertunda sekitar 15 menit**; karena itu bukan layanan eksekusi harga real-time.

**Hipotesis utama yang harus dibuktikan:** investor ritel non-day-trader merasa pembuatan alert via chat dan penjelasan faktual saat alert terpicu lebih praktis daripada alur yang sudah mereka gunakan di aplikasi sekuritas. Hipotesis ini **belum tervalidasi oleh riset pengguna**.

## 2. Masalah dan Peluang

### 2.1 Masalah yang hendak diselesaikan
1. Investor sudah punya angka target, tetapi tidak ingin terus memantau grafik saat bekerja, belajar, atau beraktivitas.
2. Pengguna ingin mengekspresikan kondisi seperti “turun 10%” atau “di bawah rata-rata 30 hari” tanpa mengisi banyak langkah.
3. Ketika alert terpicu, angka harga saja sering belum memberi gambaran singkat tentang perubahan historis.
4. Pengguna dengan beberapa alert ingin melihat keseluruhan targetnya dalam satu tampilan.

### 2.2 Hal yang **belum** terbukti
- Klaim bahwa seluruh aplikasi sekuritas memiliki alert yang kaku atau tanpa konteks **tidak boleh diperlakukan sebagai fakta pasar** tanpa audit kompetitor.
- Belum terbukti bahwa pengguna bersedia membuka dashboard tambahan, apalagi membayar Rp29.000.
- Belum terbukti bahwa keterlambatan data ±15 menit dapat diterima untuk semua jenis target.
- Belum terbukti bahwa insight AI meningkatkan retensi atau konversi.

### 2.3 Positioning dan diferensiasi
**Bukan:** “Stockbit/Ajaib versi lain”, “aplikasi portfolio baru”, atau “AI yang memilih saham.”  
**Adalah:** *Telegram-first stock alert companion* dengan pengingat yang diatur pengguna dan konteks faktual saat alert terpicu.

Diferensiasi yang akan diuji:
- Input kondisi harga melalui pesan natural sederhana.
- Notifikasi di kanal komunikasi yang sudah dipakai pengguna.
- Konteks angka dan narasi singkat yang tidak mengarahkan transaksi.
- Tampilan web untuk mengelola banyak alert, bukan menyalin seluruh fitur sekuritas.

## 3. Pengguna dan Jobs-to-be-Done

| Segmen | Situasi | Job-to-be-done | Ukuran manfaat |
|---|---|---|---|
| Karyawan investor jangka panjang | Memiliki target 4–6 emiten, tidak selalu memantau pasar | “Ingatkan saya jika target saya tercapai” | Alert berhasil dibuat dan terkirim benar |
| Investor dividen/jangka panjang | Ingin memantau kondisi harga historis | “Beri tahu ketika harga memenuhi kondisi yang saya pilih” | Pengguna memahami kondisi alert |
| Investor pemula | Memantau 1–2 saham | “Saya ingin mencoba tanpa membayar” | Berhasil membuat alert sederhana |

**Tidak ditargetkan pada v1.0:** scalper/day trader, pencari rekomendasi saham, investor institusi, pengguna yang membutuhkan eksekusi transaksi atau data real-time.

Persona di atas adalah **asumsi produk dari PRD awal**, bukan hasil wawancara.

## 4. Tujuan Produk dan Non-Goals

### 4.1 Tujuan v1.0
- Memudahkan pembuatan alert saham IDX lewat Telegram.
- Memastikan alert hanya terpicu oleh quote yang sah dan tidak terkirim ganda.
- Mengirim notifikasi yang jelas, menyertakan waktu quote dan keterlambatan data.
- Memberikan konteks faktual untuk trial/member ketika alert terpicu.
- Menguji apakah dashboard web membantu monitoring multi-alert.
- Menguji konversi trial ke membership dengan biaya data dan AI yang terkendali.

### 4.2 Bukan tujuan v1.0
- Order beli/jual atau integrasi akun sekuritas.
- Portofolio kepemilikan, jumlah lot, average buy, profit/loss.
- Rekomendasi saham, target yang dibuat bot, ranking “saham terbaik”.
- Harga real-time, pasar selain saham IDX.
- Alert berita, volume, dividen, atau aksi korporasi.
- AI chat bebas, AI parsing, AI history analyst, atau ringkasan berita sebagai fitur yang wajib.
- Mobile native app.

## 5. Strategi Produk: Telegram dan Website

### 5.1 Telegram = primary interaction channel
Peran: registrasi `/start`, cek harga, membuat alert, melihat daftar, membatalkan, status paket, upgrade, notifikasi, reaktivasi/geser target sesuai kontrak yang disetujui.

**Kriteria pengalaman:** pengguna dapat membuat alert yang valid tanpa membuka website.

### 5.2 Website = complementary monitoring surface
**Baseline v1.0 yang bersumber dari SPEC awal:** login OTP via Telegram, overview, daftar alert, grafik harga, akses untuk trial/member.

**Hipotesis nilai tambah web:** lebih mudah melihat banyak alert dan pergerakan harga secara visual dibanding membaca chat satu per satu.

**Kandidat peningkatan, bukan otomatis masuk scope v1.0:**
- Garis target pada grafik harga.
- Filter/status dan riwayat alert terpicu.
- Ringkasan statistik monitoring.
- Pengelolaan alert dari web, **hanya jika** endpoint dan hak akses yang ada telah dikonfirmasi konsisten pada SPEC/ARCHITECTURE.
- Analisis riwayat oleh AI setelah data histori tersedia.

**Keputusan produk:** website **tidak diposisikan sebagai portfolio tracker** dalam MVP. Tidak ada klaim sinkronisasi kepemilikan dari Stockbit, Ajaib, Bibit, atau sekuritas lain.

### 5.3 Kriteria apakah website layak diperluas
Uji apakah pengguna trial/member:
- Membuka dashboard setelah membuat alert.
- Kembali ke dashboard untuk mengecek beberapa target.
- Lebih cepat memahami posisi target dari grafik dibanding daftar chat.
- Menganggap fitur tersebut cukup bermanfaat untuk digunakan lagi.

Jika tidak, prioritaskan Telegram dan pertahankan web minimal. **Jangan menambah fitur AI semata-mata agar website tampak berguna.**

## 6. Ruang Lingkup Fungsional MVP

| ID | Kemampuan | Kanal | Status |
|---|---|---|---|
| FR-01 | `/start` dan trial 7 hari sekali per Telegram ID | Telegram | Baseline |
| FR-02 | `/harga <KODE>` / input ticker | Telegram | Baseline |
| FR-03 | Parsing regex untuk 5 jenis alert | Telegram | Baseline |
| FR-04 | Validasi ticker, fraksi harga, target belum terlewati | Backend | Baseline |
| FR-05 | Cron memeriksa harga tiap 5 menit saat sesi bursa | Backend | Baseline |
| FR-06 | Alert sekali terpicu dan notifikasi tidak ganda | Backend/Telegram | Baseline |
| FR-07 | Konteks AI faktual ketika alert kena (trial/member) | Telegram | Baseline |
| FR-08 | Ringkasan harian tanpa AI (trial/member) | Telegram | Baseline |
| FR-09 | Trial/member/free dan pembayaran dengan webhook | Backend | Baseline |
| FR-10 | Login OTP Telegram, overview, daftar alert, grafik harga | Web | Baseline |
| FR-11 | Garis target, riwayat, analisis monitoring | Web | **Usulan, perlu persetujuan** |
| FR-12 | AI news, portfolio insight, AI parser | Multi-kanal | **Di luar MVP** |

**Jenis alert baseline:** `price_below`, `price_above`, `pct_drop`, `pct_rise`, `below_avg_30d`. Persentase dihitung dari base price saat alert dibuat; rata-rata 30 hari dihitung ulang sesuai SPEC.

## 7. User Journey dan Acceptance Criteria Produk

### 7.1 Onboarding
1. User membuka bot dan mengirim `/start`.
2. Sistem membuat akun jika belum ada, menetapkan trial 7 hari sekali seumur akun.
3. Bot menjelaskan contoh input dan disclaimer.
4. User mencoba membuat alert.

**Sukses jika:** user yang baru mendaftar dapat membuat alert valid, melihat konfirmasi target dan timestamp data. `/start` ulang tidak memperpanjang trial.

### 7.2 Membuat alert
1. User mengetik `bbca 3000` atau `tlkm turun 10%`.
2. Parser deterministik mengekstrak ticker/kondisi.
3. Backend memvalidasi ticker, fraksi IDX, quote, kondisi dan entitlement.
4. Alert tersimpan; user menerima konfirmasi.

**Sukses jika:** target tidak sah ditolak dengan informasi koreksi; tidak ada alert yang langsung dianggap terpenuhi ketika dibuat.

### 7.3 Trigger dan notifikasi
1. Cron memperoleh quote valid dari provider.
2. Kondisi alert dipenuhi.
3. Status `active` berubah menjadi `triggered` secara atomik.
4. Bot mengirim harga, target, waktu quote, label data tertunda; konteks AI hanya untuk yang berhak.

**Sukses jika:** tidak ada trigger palsu/duplikat. Gagal AI tidak membatalkan notifikasi dasar; gagal data tidak menandai alert terpicu.

### 7.4 Dashboard
1. Trial/member login dengan OTP Telegram.
2. User melihat overview, daftar alert, grafik harga.
3. User dapat kembali ke Telegram untuk interaksi cepat.

**Sukses jika:** data alert konsisten dengan backend; pengguna Free tidak dapat mengakses dashboard premium.

### 7.5 Trial berakhir
User turun ke Free tanpa kehilangan data alert. Hanya dua alert harga sederhana tertua yang tetap dipantau sesuai aturan SPEC; yang lain ditandai tidak dipantau.

## 8. Strategi AI dan Guardrails

### 8.1 AI di MVP
**AI Stock Movement Explainer** menghasilkan narasi singkat 1–2 kalimat berdasarkan metrik harga yang dihitung JavaScript, untuk trial/member pada notifikasi trigger.

- AI **tidak** melakukan perhitungan harga, persentase, atau rata-rata.
- AI **tidak** menentukan target harga atau keputusan beli/jual.
- Output harus melewati `ai/complianceFilter.js`; jika gagal, **seluruh narasi dibuang**.
- Jika AI timeout/gagal, notifikasi faktual tetap dikirim.
- Konteks di-cache per ticker per hari sesuai SPEC.
- Free tier tidak mengakses maupun menampilkan konteks AI, termasuk dari cache.

### 8.2 Kandidat sesudah MVP
| Kandidat | Nilai yang diuji | Prasyarat |
|---|---|---|
| AI Alert History Analyst | Menjelaskan pola aktivitas alert user | Riwayat alert yang andal, persetujuan akses data |
| News Summarizer | Merangkum berita relevan dengan sumber dan tanggal | Provider berita berlisensi, verifikasi sumber |
| Portfolio Insight | Menjelaskan komposisi portofolio tercatat | Portfolio tracker yang disetujui, data kepemilikan |
| Smart Alert Parser | Memahami instruksi kompleks | Validasi deterministik, konfirmasi user, biaya terukur |

Semua kandidat memerlukan PRD/SPEC change request, penilaian risiko, dan evaluasi biaya sebelum diimplementasikan.

## 9. Model Bisnis dan Entitlement

| Tier | Durasi | Harga | Kemampuan |
|---|---|---|---|
| Trial | 7 hari, sekali per Telegram ID | Gratis | 50 alert, semua jenis, konteks AI, ringkasan, dashboard |
| Member | 30 hari/pembelian | **Rp29.000 (hipotesis, belum final)** | Setara trial |
| Free | Tanpa batas | Gratis | Cek harga, maksimal 2 alert harga sederhana yang dipantau; tanpa AI, ringkasan, dashboard |

Tier **dihitung dari subscription dan masa trial**, bukan kolom `plan` pada user.

**Asumsi ekonomi yang harus diuji:**
- Biaya lisensi data berpotensi menjadi fixed cost bahkan sebelum ada pelanggan.
- Request quote bisa dibagi per ticker unik, namun biaya aktual mengikuti kebijakan provider yang dipilih.
- AI per ticker per hari mengurangi biaya tetapi tidak otomatis menjamin unit economics positif.
- Rp29.000 tidak boleh dipublikasikan sebagai harga final sebelum disetujui.

**Upsell baseline:** saat free mencapai batas alert, meminta alert premium, atau menjelang trial habis. Komunikasi upgrade tidak boleh menyesatkan soal ketepatan waktu data.

## 10. KPI dan Rencana Validasi

### 10.1 KPI dari PRD awal (target, bukan capaian)
| Metrik | Target 90 hari |
|---|---|
| User membuat ≥1 alert dalam 24 jam | ≥60% |
| Trial user mengalami ≥1 trigger | ≥35% |
| Trial → member | ≥10% |
| Renewal bulan ke-2 | ≥55% |
| False trigger | 0 |
| Notifikasi ganda | 0 |
| p95 quote tersedia → notifikasi terkirim | ≤6 menit |
| Biaya data+AI per member aktif | ≤25% dari harga jual |

**Definisi pengukuran perlu dipastikan:** periode cohort, denominasi konversi, definisi trigger valid, dan cara mengukur latency provider.

### 10.2 Metrik baru (baseline dahulu, target ditetapkan setelah riset)
- Aktivasi: waktu dari `/start` sampai alert pertama.
- Kualitas input: persentase pesan yang berhasil diparse dan validasi gagal.
- Web adoption: persentase user berhak yang membuka dashboard.
- Web retention: pengguna kembali ke dashboard dalam 7 hari.
- Web usefulness: pengguna berhasil menemukan posisi target melalui grafik.
- AI usefulness: persentase user yang merasa penjelasan membantu, tanpa mendorong keputusan transaksi.
- Cost per monitored ticker, cost per active member, AI fallback rate.
- Alert latency end-to-end dan persentase quote ditolak karena stale/sanity check.

### 10.3 Validasi sebelum perluasan scope
1. Audit fitur alert di beberapa aplikasi sekuritas (berdasarkan bukti dan tanggal).
2. Wawancara investor non-day-trader mengenai masalah monitoring dan toleransi delayed data.
3. Uji prototipe Telegram untuk kemudahan membuat alert.
4. Uji prototipe dashboard versus hanya Telegram.
5. Uji willingness-to-pay dan biaya provider.
6. Gunakan hasilnya untuk memutuskan roadmap, bukan asumsi internal.

## 11. Risiko, Kepatuhan, dan Privasi

| Risiko | Dampak | Mitigasi / keputusan |
|---|---|---|
| Disalahartikan sebagai penasihat investasi | Hukum dan kepercayaan | Target dari user; bahasa faktual; filter AI; disclaimer; **konsultasi hukum sebelum monetisasi** |
| Lisensi data komersial tidak memadai | Produk tak layak rilis | Konfirmasi hak penggunaan, redistribusi, caching dan tampilan grafik |
| Delayed quote menyesatkan | Pengguna salah mengartikan ketepatan waktu | Label timestamp + ±15 menit di semua tampilan harga |
| Quote salah/stale | False alert | Sanity check, skip invalid, retry |
| Trigger ganda | Hilang kepercayaan | Atomic conditional update, idempotency |
| AI halusinasi atau bahasa rekomendatif | Risiko keputusan investasi | Metrik dihitung JS, compliance filter, fail closed pada narasi |
| Penggunaan dashboard rendah | Biaya pengembangan tanpa manfaat | Ukur adopsi, jangan perluas tanpa bukti |
| AI/data terlalu mahal | Margin negatif | Cache, batas tier, audit penggunaan |
| Kebocoran Telegram ID/OTP/riwayat | Privasi/keamanan | OTP berumur pendek, kontrol akses, redaksi log, retensi data |
| Ketergantungan Telegram/provider | Gangguan layanan | Isolasi client, observability, retry terkontrol |
| Scope creep mengganggu workflow | Delay dan kontrak rusak | Change control, jangan edit WORKFLOW.md |

**Penting:** Disclaimer saja tidak otomatis menghapus kewajiban hukum. Owner harus memverifikasi aturan OJK dan lisensi IDX/provider sebelum layanan berbayar dirilis.

## 12. Roadmap dan Change Control

### 12.1 Roadmap baseline dari PRD awal (belum diubah)
| Versi | Isi |
|---|---|
| v1.0 | Scope MVP pada SPEC |
| v1.1 | Alert volume dan harga terendah 52 minggu |
| v1.2 | Jurnal portofolio via chat dan P/L |
| v1.3 | Kalender dividen dan aksi korporasi |
| v2.0 | Tier Pro data real-time |

### 12.2 Usulan evaluasi roadmap, **bukan pengganti otomatis**
- Setelah MVP, uji garis target dan riwayat monitoring sebagai kandidat penguatan web.
- Uji AI Alert History Analyst **hanya** setelah histori dan guardrails memadai.
- Evaluasi berita/portofolio berdasarkan permintaan nyata, biaya dan lisensi.
- Real-time hanya jika kebutuhan, lisensi, dan economics terbukti.

**Prosedur perubahan:** keputusan owner → revisi PRD → revisi SPEC → cek ARCHITECTURE/ERD/TESTING/AGENTS → buat issue sesuai **WORKFLOW.md yang tidak diubah**. Jika urutan atau ownership workflow menghalangi usulan, tunda usulan atau minta owner memilih tanpa mengedit workflow.

## 13. Kriteria Kesiapan Rilis

- Fitur MVP dalam SPEC telah diterima dengan test sesuai TESTING.md.
- Quote valid, timestamp dan label delayed selalu tampil.
- Alert tidak salah trigger dan tidak terkirim dua kali pada skenario uji.
- Entitlement trial/member/free dan expiry konsisten.
- AI gagal atau tidak lolos filter tidak menghambat notifikasi inti.
- OTP, webhook Telegram dan pembayaran divalidasi.
- Lisensi data komersial, privasi, dan review hukum diselesaikan.
- Metrik aktivasi, trigger, conversion, cost, dan web usage dapat diukur.
- Owner menyetujui harga, provider, dan scope final.

## 14. Keputusan yang Masih Dibutuhkan Owner

### Blocking sebelum rilis berbayar
- [ ] Provider data IDX dan izin penggunaan komersial/redistribusi/caching.
- [ ] Review hukum terkait batas informasi faktual vs nasihat investasi.
- [ ] Payment gateway dan kontrak webhook.
- [ ] Provider AI dan batas biaya.
- [ ] Harga member final.
- [ ] Verifikasi sesi perdagangan dan tabel tick IDX terbaru.
- [ ] Kebijakan privasi, retensi data, dan OTP.

### Keputusan produk
- [ ] Nama produk final (nama kerja: Mon Saham).
- [ ] Setujui/tolak positioning Telegram-first.
- [ ] Pastikan dashboard v1.0 tetap baseline atau ajukan perluasan dengan kontrak API yang sesuai.
- [ ] Tentukan apakah garis target/riwayat masuk versi setelah MVP.
- [ ] Tentukan apakah AI Alert History Analyst layak diprioritaskan setelah riset.
- [ ] Setujui rencana audit kompetitor dan pengujian pengguna.

## 15. Matriks Perubahan dari PRD Lama

| Area | Perlakuan | Alasan |
|---|---|---|
| Problem dan persona | Dipertahankan, diberi label hipotesis | Hindari klaim riset yang belum ada |
| Telegram-first | Dipertahankan | Kanal inti produk |
| Website | Diperjelas sebagai monitoring, bukan portfolio tracker | Menjawab alasan website ada |
| AI trigger context | Dipertahankan dan diberi guardrails | Sesuai SPEC |
| AI history/news/portfolio | Didaftarkan sebagai kandidat, bukan MVP | Cegah scope creep |
| Tier dan harga asumsi | Dipertahankan | Belum ada persetujuan harga baru |
| KPI awal | Dipertahankan, definisi perlu diperjelas | Konsistensi dengan baseline |
| KPI web/AI | Ditambahkan tanpa target karangan | Menguji manfaat aktual |
| Roadmap awal | Dipertahankan | Tidak mengubah keputusan owner secara sepihak |
| Risiko | Diperluas | Lisensi, privasi, AI, adopsi |
| WORKFLOW.md | **Tidak diubah** | Constraint eksplisit |

---
**Status akhir:** Dokumen ini adalah **PRD revisi untuk persetujuan**, bukan perintah otomatis mengimplementasikan seluruh fitur usulan. Untuk coding, ikuti SPEC dan kontrak yang sudah disahkan, serta WORKFLOW.md tanpa perubahan.
