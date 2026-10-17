# Provider and IDX readiness — 2026-10-17

Status: BLOCKED. No selected provider, credentials, licence evidence, or authentic response fixtures were supplied. Internal examples in ARCHITECTURE/STYLE are contracts, not provider evidence.

## Provider evaluation evidence required from owner

- [ ] Identify provider/product and official API documentation, quote/history/symbol endpoints and authentication scheme. STYLE's `/quotes` and bearer auth are illustrative only.
- [ ] Obtain written permission covering paid Telegram alerts, web display, redistribution, caching, historical storage, derived metrics and AI processing; record restrictions, expiry and attribution. No licensing claim is made here.
- [ ] Demonstrate IDX equities and IHSG coverage, ticker normalization, suspended/delisted flags, batch limits, pagination, quotas, costs and service availability.
- [ ] Supply sanitized actual success/error responses with capture date, endpoint, HTTP status and provenance. Remove credentials, personal data and licensed data not permitted in the repository. Codex creates fixtures in its later issue #2; none fabricated now.
- [ ] Map quote fields to `symbol`, integer `lastPrice`, integer `prevClose`, integer `changeBps`, timestamp `asOf`, boolean `isDelayed` (ARCHITECTURE §4). Confirm timestamp timezone and whether it represents trade time or publication time.
- [ ] Confirm historical daily OHLC integer rupiah, volume in shares, WIB tradeDate, ordering, gaps, adjustment policy, IHSG precision, corporate actions and previous-close semantics.
- [ ] Validate 60-day backfill suffices for 30 trading observations; maximum API history request is 90 days. Resolve calendar-day vs trading-day meaning before coding.
- [ ] Document 401/403 auth failures, 429/rate limits, 5xx, timeouts, empty/partial batches, unknown symbols, malformed data and stale/future timestamps; do not invent raw provider payloads.
- [ ] Confirm abort at MARKET_CALL_TIMEOUT_MS=10000, cache TTL=300 seconds, unique-symbol batching and retry next cron round; upstream failure maps to MarketUnavailable/HTTP 503 without leaking URL/key (WORKFLOW Phase 2; SPEC §5; STYLE §7).
- [ ] Resolve freshness: SPEC rejects non-current trading-date quotes and moves >3500 bps; cache age is not quote age. No intraday maximum age or future-clock tolerance is specified. ±15-minute delay is an assumption requiring provider evidence.

## Official IDX check

Attempted 2026-10-17 using official IDX sources only:
- [Trading mechanism and hours](https://www.idx.co.id/id/produk/mekanisme-dan-jam-perdagangan): indexed official result found; direct page retrieval returned HTTP 403.
- [Exchange holiday calendar](https://www.idx.co.id/id/berita/jadwal-libur-bursa): direct retrieval returned HTTP 403.

Official indexed results indicate a possible 15:49:59 regular Session II end, unlike the project's 16:00. This is preliminary discovery, NOT a verified schedule or an instruction to change WORKFLOW. Auction/post-trading phases and board applicability still require inspection of the current official rule and effective date.

Owner must retrieve the current official schedule, applicable price-fraction rule and 2026 holiday announcement (including amendments); retain source URLs, rule IDs, effective dates and date checked. Compare every session boundary and tick threshold to SPEC §8 / ARCHITECTURE §9 / WORKFLOW Phase 2. Approve an explicit reconciliation outside immutable WORKFLOW before implementation.

Asia/Jakarta, 15-minute session extensions and the 35% quote sanity threshold are project requirements; the latter two are not asserted to be IDX rules. Confirm how delayed data, breaks, closing auctions and holidays interact. No holiday dates or constants were generated. All official verification remains BLOCKED.
