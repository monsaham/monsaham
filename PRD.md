# PRD.md — Kenapa Project Ini Dibangun

> `SPEC.md` menjawab **apa** yang dibangun. File ini menjawab **kenapa**.
> Agent tidak wajib baca file ini untuk koding, tapi wajib baca kalau harus
> memutuskan sesuatu yang tidak tertulis di `SPEC.md`.

---

## 1. Masalah

Investor ritel yang tidak trading harian punya pola yang sama: **sudah punya
angka di kepala** ("aku mau tambah BBRI kalau 4.200"), tapi terbentur dua tembok:

1. **Tidak sempat memantau.** Kerja, kuliah, urus anak. Harga menyentuh 4.200 jam
   10 pagi, naik lagi jam 2 siang, dan mereka baru buka aplikasi malamnya.
2. **Alert di aplikasi sekuritas kaku dan tersebar.** Tiap sekuritas punya menu
   sendiri, kondisinya hanya "harga ≥ / ≤", dan kalau kena cuma muncul angka
   tanpa konteks apa pun.

Yang kita jual: **pengingat yang bisa dipasang semudah chat**, dengan kondisi yang
masuk akal buat manusia ("turun 10%", "di bawah rata-rata bulan lalu"), dan
konteks singkat saat kena. **Keputusan beli tetap 100% di tangan user.**

## 2. Kenapa Telegram, bukan WhatsApp atau aplikasi sendiri

| Alasan | Penjelasan |
|---|---|
| Gratis | Bot API resmi, tanpa biaya per pesan (WA Cloud API charge per percakapan — mahal untuk notifikasi yang sering) |
| Notifikasi andal | Push notification Telegram cepat dan tidak dibatasi template seperti WA |
| Aman dari banned | Tidak pakai library tidak resmi |
| Tidak perlu install | Telegram sudah ada di HP user |
| Inline keyboard | [Aktifkan lagi] [Geser target] tanpa user mengetik |

## 3. Siapa Penggunanya

| Persona | Profil | Yang dia butuh |
|---|---|---|
| **Andi, 27, karyawan** | Nabung saham tiap gajian, 4–6 emiten bluechip | Dikabari saat harga incarannya tersentuh, tanpa pantengin aplikasi |
| **Maya, 34, ibu bekerja** | Portofolio dividen jangka panjang | Tahu kapan harga di bawah rata-rata biasanya, untuk menambah posisi |
| **Rizky, 22, mahasiswa** | Baru mulai, modal kecil, 1–2 saham | Gratis dulu, belajar membaca pergerakan harga |

**Bukan target v1.0:** trader harian / scalper (butuh real-time), pencari sinyal
("saham apa yang bagus?"), investor institusi.

## 4. Model Bisnis

| Tier | Durasi | Harga | Isi |
|---|---|---|---|
| Free Trial | 7 hari, sekali seumur akun | Rp 0 | Semua fitur v1.0 |
| Member | 30 hari | **Rp 29.000** *(asumsi — konfirmasi sebelum rilis)* | Semua fitur v1.0, maks. 50 alert |
| Free | selamanya | Rp 0 | Cek harga + 2 alert harga, tanpa AI, ringkasan, dashboard |

**Kenapa tier free tetap ada dan tetap berguna:** supaya user tidak menghapus
bot. 2 alert cukup untuk merasakan manfaatnya, dan cukup sedikit untuk terasa
kurang. User free adalah pipeline konversi bulan depan. Tapi mereka **tidak boleh**
menghabiskan biaya AI sepeser pun.

**Kenapa trial 7 hari, bukan 5 seperti Mon Mon:** nilai produk ini baru terasa
saat alert **kena**, dan itu tergantung pasar. 7 hari kalender = 5 hari bursa,
cukup untuk sebagian besar alert yang realistis (target 3–5% dari harga
sekarang) sempat tersentuh. 5 hari kalender bisa cuma 3 hari bursa kalau ada
akhir pekan.

**Beda struktur biaya dengan Mon Mon:** di Mon Mon biaya utama adalah AI per
user. Di sini ada **biaya data tetap per bulan** (langganan API harga saham) yang
harus dibayar walaupun user masih 0. Tapi biaya data naik mengikuti **jumlah
ticker unik yang dipantau**, bukan jumlah user — 1.000 user yang memasang alert
BBCA tetap cuma 1 request harga per putaran.

**Titik upsell yang direncanakan:**
1. User free mencoba membuat alert ke-3, atau alert persen / rata-rata
2. Alert user free kena: konteks AI ditampilkan terkunci
3. Hari ke-6 trial: "besok trial habis. Alert kamu yang tidak akan dipantau lagi: ..."

## 5. Ukuran Keberhasilan

| Yang diukur | Target 90 hari pertama |
|---|---|
| User yang membuat ≥1 alert di 24 jam pertama | ≥ 60% |
| User trial yang mengalami ≥1 alert kena selama trial | ≥ 35% |
| Konversi trial → member | ≥ 10% |
| Member yang perpanjang di bulan ke-2 | ≥ 55% |
| Alert palsu (kena padahal harga sah tidak menyentuh target) | **0** |
| Alert terkirim dua kali | **0** |
| p95 jeda dari harga tersedia di provider → notifikasi terkirim | ≤ 6 menit |
| Biaya data + AI per member aktif per bulan | ≤ 25% dari harga jual |

Kalau konversi < 7% setelah 200 user trial, yang dievaluasi duluan adalah
**berapa persen user trial yang alert-nya pernah kena**. Kalau rendah, masalahnya
di onboarding (user memasang target terlalu jauh), bukan di harga.

## 6. Prinsip Produk (dipakai untuk memutuskan hal yang tidak tertulis)

1. **Jangan pernah terdengar seperti rekomendasi.** Kalau ragu sebuah kalimat
   terdengar seperti saran beli/jual, hapus kalimatnya. Ini risiko hukum.
2. **Alert terlambat lebih baik daripada alert palsu.** Satu alert palsu merusak
   kepercayaan lebih parah daripada sepuluh alert yang telat 5 menit.
3. **Angka harus benar, narasi boleh luwes.** Makanya AI tidak menghitung.
4. **Jujur soal keterbatasan data.** Data tertunda selalu disebut jelas.
5. **Kalau ragu, pilih yang lebih murah biayanya** — baik biaya data maupun AI.
6. **Sederhana dulu.** Fitur yang tidak dipakai tetap harus dirawat.

## 7. Roadmap

| Versi | Isi | Selesai kalau |
|---|---|---|
| **v1.0** | Scope di `SPEC.md §2` | Konversi trial→member bisa diukur |
| v1.1 | Alert volume melonjak + alert harga terendah 52 minggu | Member aktif memakai ≥ 2 jenis alert |
| v1.2 | Jurnal portofolio via chat ("beli BBRI 5 lot 4.200") + untung-rugi | Retensi bulan ke-2 naik |
| v1.3 | Kalender dividen & aksi korporasi untuk saham di alert | — |
| v2.0 | Tier Pro dengan data real-time | Ada permintaan nyata & biaya data tertutup |

## 8. Risiko

| Risiko | Mitigasi |
|---|---|
| **Dianggap penasihat investasi tanpa izin (OJK)** | Target selalu dari user, bot hanya fakta, `complianceFilter` untuk output AI, disclaimer di `/start`, `/bantuan`, dan notifikasi. **Konsultasi hukum sebelum menerima pembayaran.** |
| **Lisensi data untuk penggunaan komersial** | Pakai provider berlisensi, bukan scraping. Semua akses lewat `market/marketClient.js` supaya ganti provider cukup di satu file. |
| Provider data down / data salah | Sanity check ±35% dari penutupan kemarin, alert tidak ditandai kena tanpa harga sah, cron coba lagi di putaran berikutnya |
| Biaya data tetap sebelum ada pemasukan | Mulai dari paket provider terkecil, pantau jumlah ticker unik |
| User kecewa karena data tertunda | Disebut jelas di setiap pesan harga, di `/start`, dan di halaman upgrade |
| Alert dobel karena cron jalan 2× | Cron proses terpisah satu instance + UPDATE bersyarat `WHERE status = 'active'` |
| Alert menumpuk selamanya, ticker dipantau makin banyak | Pantau di v1.0; kalau perlu, v1.1 tambah masa berlaku alert |
| Biaya AI membengkak | Konteks di-cache 1× per saham per hari, free tier tanpa AI |
| Telegram mengubah kebijakan Bot API | Semua panggilan Bot API lewat `telegram/client.js` saja |

## 9. Yang Masih Perlu Diputuskan Owner

- [ ] **Provider data saham** + pastikan lisensinya boleh untuk produk berbayar
- [ ] **Konsultasi hukum** soal batas "alat pengingat" vs "penasihat investasi"
- [ ] Payment gateway: Midtrans / Xendit / transfer manual + verifikasi admin?
- [ ] Provider AI untuk konteks
- [ ] Harga final member (asumsi sekarang Rp 29.000 / 30 hari)
- [ ] Nama produk final (nama kerja: Mon Saham)
- [ ] Verifikasi jam sesi & tabel fraksi harga ke peraturan IDX terbaru
- [ ] Berapa lama data user `free` yang tidak aktif disimpan?
