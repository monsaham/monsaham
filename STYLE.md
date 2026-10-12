# STYLE.md — Gaya Coding Project Ini

> Tiru contoh ✅ persis. Hindari pola ❌.
> Tujuannya bukan "kode paling keren", tapi **kode yang pemiliknya bisa baca dan
> debug sendiri**. Berlaku sama untuk Claude Code dan Codex — kode dua agent
> harus terlihat seperti ditulis satu orang.

---

## Prinsip Umum

1. Kode ditulis untuk dibaca manusia, bukan untuk pamer.
2. Satu fungsi = satu tanggung jawab, maksimal ~30 baris.
3. Eksplisit lebih baik daripada pintar. Tidak ada "magic".
4. Nama variabel dalam bahasa Inggris, camelCase, deskriptif.
   Pesan ke user dalam bahasa Indonesia.
5. Komentar hanya untuk menjelaskan **kenapa**, bukan **apa**.
6. Dilarang abstraksi prematur: tidak ada factory, decorator, atau generic helper
   sampai pola yang sama muncul minimal 3 kali.

---

## 1. Controller

✅ **BEGINI** — class dengan static method, try/catch, lempar ke `next()`:
```js
class AlertController {
  static async findAll(req, res, next) {
    try {
      const { status, page = 1, limit = 10 } = req.query;

      const options = { where: { UserId: req.user.id }, limit: Number(limit) };
      options.offset = (Number(page) - 1) * Number(limit);
      options.order = [['createdAt', 'DESC']];
      options.include = [{ model: Stock, attributes: ['symbol', 'name'] }];

      if (status) {
        options.where.status = status;
      }

      const result = await Alert.findAndCountAll(options);

      res.status(200).json({
        data: result.rows,
        meta: {
          page: Number(page),
          limit: Number(limit),
          totalItems: result.count,
          totalPages: Math.ceil(result.count / Number(limit)),
        },
      });
    } catch (error) {
      next(error);
    }
  }
}
```

❌ **JANGAN BEGINI** — arrow function anonim, chaining padat, error ditelan:
```js
router.get('/alerts', (req, res) =>
  Alert.findAll({ where: req.query.status ? { status: req.query.status } : {} })
    .then(a => res.json(a))
    .catch(() => res.status(500).send('error')));
```

---

## 2. Error Handling

✅ **BEGINI** — lempar object bernama, ditangani terpusat:
```js
// di controller / service
if (!alert) {
  throw { name: 'NotFound', message: 'Alert tidak ditemukan' };
}
if (!isValidTick(targetPrice)) {
  throw { name: 'BadRequest', message: `Harga ${formatNumber(targetPrice)} tidak sesuai fraksi harga` };
}
if (!EntitlementService.can(tier, 'alert.percent')) {
  throw { name: 'Forbidden', message: 'Alert persen hanya untuk Member', requiredTier: 'member' };
}

// middlewares/errorHandler.js
function errorHandler(error, req, res, next) {
  console.error({ name: error.name, message: error.message });

  if (error.name === 'SequelizeValidationError') {
    return res.status(400).json({ message: error.errors[0].message });
  }
  if (error.name === 'BadRequest') {
    return res.status(400).json({ message: error.message });
  }
  if (error.name === 'Unauthorized') {
    return res.status(401).json({ message: error.message });
  }
  if (error.name === 'Forbidden') {
    return res.status(403).json({ message: error.message, requiredTier: error.requiredTier });
  }
  if (error.name === 'NotFound') {
    return res.status(404).json({ message: error.message });
  }
  if (error.name === 'TooManyRequests') {
    return res.status(429).json({ message: error.message });
  }
  if (error.name === 'MarketUnavailable') {
    return res.status(503).json({ message: 'Data harga sedang tidak tersedia, coba lagi sebentar' });
  }

  res.status(500).json({ message: 'Internal server error' });
}
```

❌ **JANGAN BEGINI** — `res.status()` berserakan di controller,
`catch (e) { console.log(e) }` tanpa response, atau meneruskan pesan error
provider mentah ke user (bisa berisi URL + API key).

---

## 3. Async/Await

✅ `await` dengan try/catch.
❌ `.then().catch()` berantai, callback bersarang, `Promise.all` tanpa penjelasan.

---

## 4. Pengecekan Tier

✅ **BEGINI** — satu sumber, dipanggil dari middleware, handler bot, dan job:
```js
// services/entitlementService.js
const CAPABILITIES = {
  trial:  ['quote.read', 'alert.price', 'alert.percent', 'alert.average', 'alert.context', 'summary.daily', 'dashboard.read'],
  member: ['quote.read', 'alert.price', 'alert.percent', 'alert.average', 'alert.context', 'summary.daily', 'dashboard.read'],
  free:   ['quote.read', 'alert.price'],
};

class EntitlementService {
  static resolve(user, activeSubscription, now) {
    if (activeSubscription && activeSubscription.status === 'active' && activeSubscription.expiresAt > now) {
      return 'member';
    }
    if (user.trialEndsAt && now < user.trialEndsAt) {
      return 'trial';
    }
    return 'free';
  }

  static can(tier, capability) {
    return CAPABILITIES[tier].includes(capability);
  }

  static maxActiveAlerts(tier) {
    if (tier === 'free') {
      return FREE_ALERT_LIMIT;
    }
    return MEMBER_ALERT_LIMIT;
  }
}
```

```js
// middlewares/entitlement.js
function entitlement(capability) {
  return function (req, res, next) {
    try {
      if (!EntitlementService.can(req.user.tier, capability)) {
        throw { name: 'Forbidden', message: 'Fitur ini hanya untuk Member', requiredTier: 'member' };
      }
      next();
    } catch (error) {
      next(error);
    }
  };
}

// dipakai di route:
router.get('/alerts', authentication, entitlement('dashboard.read'), AlertController.findAll);
```

```js
// di job (Codex) — tier sudah dihitung alertService.findCheckable, cukup cek capability
if (EntitlementService.can(row.tier, 'alert.context')) {
  context = await ContextService.getContext(row.stock.id, tradeDate, row.tier);
}
```

❌ **JANGAN BEGINI** — pengecekan tier tersebar, tiap tempat beda logika:
```js
if (user.plan === 'member' || user.trialEndsAt > new Date()) { ... }   // ❌
if (req.user.isPremium) { ... }                                         // ❌
if (row.tier !== 'free') { ... }                                        // ❌ pakai can()
if (activeAlerts >= 2) { ... }                                          // ❌ pakai maxActiveAlerts()
```

---

## 5. Harga, Persen, dan Waktu

✅ **BEGINI**:
```js
// helpers/price.js — fungsi murni, tanpa I/O, integer semua
function parsePrice(text) {
  // "3000" → 3000, "3.000" → 3000, "3rb" → 3000, "3,5rb" → 3500
}

function tickSizeFor(price) {
  const row = TICK_SIZE_TABLE.find((tier) => price < tier.below);
  return row.tick;
}

function isValidTick(price) {
  return price > 0 && price % tickSizeFor(price) === 0;
}

function toBps(delta, reference) {
  return Math.round((delta * 10000) / reference);   // -110 dari 3100 → -355
}

function formatRupiah(price) {
  return `Rp ${price.toLocaleString('id-ID')}`;      // Rp 3.000
}

function formatBps(bps) {
  // -830 → "−8,3%". Pembulatan ke 1 desimal: Math.round(bps / 10)
}
```

```js
// helpers/dateHelper.js — supaya bisa di-freeze saat test
function now() {
  return new Date();
}
```

❌ **JANGAN BEGINI**:
```js
const price = parseFloat(input);                  // ❌ harga jangan float
const dropPct = 0.1;                              // ❌ persen = 1000 bps
const target = basePrice * 0.9;                   // ❌ float + belum dibulatkan ke tick
if (new Date().getHours() >= 9) { ... }           // ❌ jam server bisa UTC, dan tidak bisa di-test
const rupiah = 'Rp ' + price;                     // ❌ Rp 3000, tanpa pemisah ribuan
```

Waktu di service dan job selalu diterima sebagai parameter atau lewat
`dateHelper.now()`. Jam bursa selalu dihitung dalam **Asia/Jakarta** lewat
`marketCalendar`, tidak pernah dari `getHours()` server.

---

## 6. Memanggil AI (Codex)

✅ **BEGINI** — satu pintu, timeout eksplisit, gagal tidak menghancurkan alur:
```js
// ai/aiClient.js — SATU-SATUNYA file yang memanggil API AI
async function callAI(prompt, { timeoutMs = AI_CALL_TIMEOUT_MS } = {}) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(process.env.AI_PROVIDER_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'x-api-key': process.env.AI_PROVIDER_API_KEY },
      body: JSON.stringify({ model: process.env.AI_MODEL_CONTEXT, messages: [{ role: 'user', content: prompt }] }),
      signal: controller.signal,
    });

    if (!response.ok) {
      throw { name: 'AIError', message: 'AI provider error' };
    }

    return await response.json();
  } finally {
    clearTimeout(timer);
  }
}
```

```js
// ai/contextService.js — cache dulu, AI belakangan, filter selalu
static async getContext(stockId, tradeDate, tier) {
  if (!EntitlementService.can(tier, 'alert.context')) {
    return null;                                       // free: tidak baca cache pun
  }

  const cached = await Analysis.findOne({ where: { StockId: stockId, scope: 'alert_context', tradeDate } });
  if (cached) {
    return cached.insight;                             // bisa null kalau dulu ditolak filter
  }

  const metrics = await MetricsService.forContext(stockId, tradeDate);
  const result = await callAI(buildContextPrompt(metrics));
  const insight = ComplianceFilter.isClean(result.text) ? result.text : null;

  await Analysis.create({ StockId: stockId, scope: 'alert_context', tradeDate, metrics, insight, ... });
  return insight;
}
```

```js
// pemakaian di job: notifikasi tetap terkirim walaupun AI mati
let context = null;
try {
  context = await ContextService.getContext(stock.id, tradeDate, tier);
} catch (error) {
  console.error({ event: 'ai.context.failed', stockId: stock.id, message: error.message });
}
await telegramClient.sendMessage(user.telegramId, ...formatter.alertTriggered({ ..., context }));
```

❌ **JANGAN BEGINI**:
```js
await callAI(`hitung rata-rata dari harga ini: ${JSON.stringify(closes)}`);  // ❌ AI tidak menghitung
await callAI(`apakah BBCA layak dibeli sekarang?`);                         // ❌ meminta saran
return result.text.replace('murah', 'rendah');                              // ❌ jangan menyunting, buang
if (tier === 'free') { return cached?.insight; }                            // ❌ free tidak dapat cache
```

---

## 7. Memanggil API Data Saham (Codex)

✅ **BEGINI** — satu pintu, batch, timeout, API key tidak pernah bocor:
```js
// market/marketClient.js — SATU-SATUNYA file yang memanggil API data saham
async function getQuotes(symbols) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), MARKET_CALL_TIMEOUT_MS);

  try {
    const url = `${process.env.MARKET_API_URL}/quotes?symbols=${symbols.join(',')}`;
    const response = await fetch(url, {
      headers: { Authorization: `Bearer ${process.env.MARKET_API_KEY}` },
      signal: controller.signal,
    });

    if (!response.ok) {
      throw { name: 'MarketUnavailable', message: `Market provider status ${response.status}` };
    }

    const body = await response.json();
    return body.data.map(toQuote);        // ubah format provider → format ARCHITECTURE.md §4
  } finally {
    clearTimeout(timer);
  }
}
```

```js
// market/quoteService.js — cache dulu, sanity check selalu
static async getQuotes(symbols, now) {
  const missing = symbols.filter((symbol) => !isFresh(cache.get(symbol), now));
  if (missing.length > 0) {
    const fetched = await marketClient.getQuotes(missing);
    fetched.forEach((quote) => cache.set(quote.symbol, { quote, cachedAt: now }));
  }
  return buildSaneQuoteMap(symbols, now);   // quote mencurigakan tidak dimasukkan
}
```

❌ **JANGAN BEGINI**:
```js
for (const alert of alerts) {
  const quote = await marketClient.getQuote(alert.symbol);   // ❌ 1 request per alert, bukan per ticker
}
await fetch('https://finance.yahoo.com/...');                // ❌ scraping, di luar marketClient
throw { name: 'MarketUnavailable', message: url };           // ❌ url bisa memuat API key
```

**Format data provider tidak boleh keluar dari `marketClient.js`.** Semua file
lain hanya mengenal bentuk quote di `ARCHITECTURE.md §4`. Ganti provider = ganti
satu file.

---

## 8. Mengubah Status Alert (Claude Code)

✅ **BEGINI** — UPDATE bersyarat, hasilnya menentukan boleh kirim atau tidak:
```js
// services/alertService.js
static async markTriggered(alertId, { price, asOf }, now) {
  const [affectedCount] = await Alert.update(
    { status: 'triggered', triggeredAt: now, triggeredPrice: price, triggeredPriceAsOf: asOf },
    { where: { id: alertId, status: 'active' } },
  );
  return affectedCount === 1;
}
```

❌ **JANGAN BEGINI**:
```js
const alert = await Alert.findByPk(id);
if (alert.status === 'active') {          // ❌ race: dua putaran cron bisa lolos bersamaan
  alert.status = 'triggered';
  await alert.save();
}
await alert.update({ targetPrice: newTarget });   // ❌ target tidak pernah di-UPDATE, buat alert baru
```

---

## 9. Middleware

✅ Satu file satu middleware, nama file = nama fungsi:
`middlewares/authentication.js`, `middlewares/entitlement.js`,
`middlewares/telegramAuth.js`, `middlewares/errorHandler.js`

```js
async function telegramAuth(req, res, next) {
  try {
    const secret = req.headers['x-telegram-bot-api-secret-token'];
    if (secret !== process.env.TELEGRAM_WEBHOOK_SECRET) {
      throw { name: 'Unauthorized', message: 'Invalid webhook secret' };
    }
    next();
  } catch (error) {
    next(error);
  }
}
```

---

## 10. Balasan Bot & Bahasa Netral

✅ **BEGINI** — penyusunan teks di `telegram/formatter.js`, bukan di service/job:
```js
function alertTriggered({ alert, stock, quote, numbers, context, tier }) {
  const lines = [
    `🔔 ${stock.symbol} menyentuh target kamu`,
    `Harga ${formatRupiah(quote.lastPrice)} · target ${describeTarget(alert)}`,
    dataTimeLine(quote.asOf),
  ];

  if (numbers) {
    lines.push('', `📊 5 hari bursa: ${formatBps(numbers.change5dBps)} · rata-rata 30 hari ${formatRupiah(numbers.avg30Close)}`);
  }
  if (context) {
    lines.push(`💡 ${context}`);
  }
  if (numbers || context) {
    lines.push('', DISCLAIMER_LINE);
  } else if (!EntitlementService.can(tier, 'alert.context')) {
    lines.push('', '🔒 Konteks pergerakan harga tersedia untuk Member.', 'Ketik /upgrade untuk buka akses 30 hari.');
  }

  return { text: lines.join('\n'), reply_markup: alertButtons(alert.id) };
}
```

**Kosakata** — dipakai di formatter, prompt AI, dan dashboard:

| ❌ Jangan | ✅ Pakai |
|---|---|
| "BBCA lagi murah" | "BBCA Rp 2.990, turun 8,3% dalam 5 hari bursa" |
| "Saatnya beli!" / "Momen bagus" | (tidak ada padanannya — jangan ditulis) |
| "Target harga kami" | "Target kamu" |
| "Diskon 10%" | "Turun 10% dari Rp 3.320" |
| "Berpotensi naik" | (prediksi — jangan ditulis) |
| "Sinyal" | "Alert" / "pengingat" |

❌ **JANGAN BEGINI** — template string raksasa dengan ternary bersarang di dalam
service/job, atau `sendMessage` dipanggil dengan teks yang disusun di luar
`formatter.js`.

---

## 11. Job Terjadwal (Codex)

✅ **BEGINI** — `bin/cron.js` hanya menjadwalkan, logika ada di `jobs/`, satu
kegagalan tidak menghentikan yang lain:
```js
// bin/cron.js
cron.schedule(ALERT_CHECK_CRON, () => runSafely('alertCheck', AlertCheckJob.run), { timezone: MARKET_TIMEZONE });
cron.schedule(EOD_CRON, () => runSafely('eod', EodJob.run), { timezone: MARKET_TIMEZONE });

async function runSafely(jobName, jobFunction) {
  try {
    await jobFunction(dateHelper.now());
  } catch (error) {
    console.error({ event: 'job.failed', jobName, message: error.message });
  }
}
```

```js
// jobs/alertCheckJob.js — setiap job menerima `now`, sehingga bisa dites di jam berapa pun
class AlertCheckJob {
  static async run(now) {
    if (!MarketCalendar.isInSession(now)) {
      return { checked: 0, triggered: 0 };
    }
    // ... ARCHITECTURE.md §7
  }
}
```

❌ **JANGAN BEGINI** — logika bisnis langsung di callback `cron.schedule`,
`setInterval` sebagai pengganti cron, job yang tidak mengembalikan ringkasan
(`{ checked, triggered }`) sehingga tidak bisa diperiksa di test.

---

## 12. React Component (Cline)

✅ **BEGINI** — function component, satu komponen satu file, state eksplisit:
```jsx
function AlertList() {
  const [alerts, setAlerts] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [errorMessage, setErrorMessage] = useState('');

  async function fetchAlerts() {
    setIsLoading(true);
    setErrorMessage('');
    try {
      const response = await api.get('/alerts', { params: { status: 'active' } });
      setAlerts(response.data.data);
    } catch (error) {
      setErrorMessage(error.response?.data?.message || 'Gagal memuat alert');
    } finally {
      setIsLoading(false);
    }
  }

  useEffect(() => {
    fetchAlerts();
  }, []);

  if (isLoading) return <p>Loading...</p>;
  if (errorMessage) return <p className="error">{errorMessage}</p>;
  if (alerts.length === 0) return <EmptyState message="Belum ada alert aktif" />;

  return (
    <div className="alert-list">
      {alerts.map((alert) => (
        <AlertRow key={alert.id} alert={alert} />
      ))}
    </div>
  );
}
```

**Khusus project ini:**
- Setiap halaman wajib menangani **403 dengan `requiredTier`** sebagai halaman
  upsell, bukan pesan error merah:
  ```jsx
  if (error.response?.status === 403 && error.response.data.requiredTier) {
    return <UpgradePrompt requiredTier={error.response.data.requiredTier} />;
  }
  ```
- Setiap angka harga di UI wajib didampingi jam data + "data tertunda ±15 menit".
- Grafik harga menampilkan garis target user. **Tidak ada** garis "support",
  "resistance", atau anotasi apa pun yang terdengar seperti analisis.
- Kosakata §10 berlaku juga di UI.

❌ **JANGAN BEGINI** — custom hook untuk hal sepele, komponen 300 baris, ternary
bersarang di JSX, satu file berisi 5 komponen.

---

## 13. Penamaan

| Jenis | Aturan | Contoh |
|---|---|---|
| Variabel & fungsi | camelCase | `targetPrice`, `fetchAlerts` |
| Boolean | awali `is`/`has`/`can` | `isInSession`, `hasActiveSubscription` |
| Class & Component | PascalCase | `AlertController`, `AlertRow` |
| File model/controller | PascalCase | `AlertController.js` |
| File lain | camelCase | `alertCheckJob.js`, `quoteService.js` |
| Konstanta | UPPER_SNAKE | `FREE_ALERT_LIMIT`, `QUOTE_CACHE_TTL_SECONDS` |
| Konstanta env | UPPER_SNAKE | `MARKET_API_KEY` |
| Kolom FK | PascalCase + Id | `UserId`, `StockId` |
| Harga (integer rupiah) | akhiran `Price` / `Close` | `lastPrice`, `avg30Close` |
| Persen (integer bps) | akhiran `Bps` | `changeBps`, `targetPctBps` |

Dilarang singkatan tidak jelas: `p`, `d`, `tmp`, `data2`, `handleThing`, `px`, `chg`.

---

## 14. Yang Dilarang Keras

- Chaining lebih dari 2 level dalam satu baris
- Ternary bersarang
- `var`
- `==` (pakai `===`)
- **Float untuk harga atau persen**
- **Angka ajaib.** `7` → `TRIAL_DAYS`, `2` → `FREE_ALERT_LIMIT`, `300` →
  `QUOTE_CACHE_TTL_SECONDS`, `3500` → `QUOTE_SANITY_MAX_BPS`. Semua di
  `config/constants.js`
- **Pengecekan tier di luar `entitlementService`**
- **Memanggil API AI di luar `ai/aiClient.js`**
- **Memanggil API data saham di luar `market/marketClient.js`**
- **Memanggil Telegram Bot API di luar `telegram/client.js`**
- **Menyuruh AI menghitung, memprediksi, atau memberi saran**
- **UPDATE status alert tanpa `WHERE status = 'active'`**
- **Kalimat yang terdengar seperti saran beli/jual** (§10)
- **Mengedit file di folder agent lain**
- Logic bisnis di dalam file route atau di callback `cron.schedule`
- Query database langsung di dalam React component
- `console.log` sisa debugging (pakai `console.error` untuk error sungguhan)
- Menyimpan `rawMessage` atau isi pesan user ke log
- Menghapus atau mengomentari kode orang lain tanpa penjelasan di PR
