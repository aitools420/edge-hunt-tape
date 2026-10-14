# edge-hunt-tape — Robinhood Chain swaps, 2026-09-08 .. 2026-09-26

Recorded DEX swaps on Robinhood Chain for 2026-09-08 through 2026-09-26, exactly as the frozen backtest engine v1 reads them.
The files are **Release assets** of this repository (tag `tape-2026-09-26`). This repository holds only this README, `MANIFEST.sha256` and `SEAL_SCAN.md`.

## Files (38 assets, 3,934,354,224 bytes, 36,079,586 rows)

| asset | what | rows |
|---|---|---|
| `tape-YYYY-MM-DD.ndjson.gz` (19 files) | Uniswap V2 / V3-style swaps | 16,072,392 |
| `v4-swaps-YYYY-MM-DD.ndjson.gz` (19 files) | Uniswap V4 swaps, with pool state | 20,007,194 |

Each file is gzip-compressed newline-delimited JSON with one swap per line. The two kinds of file are merged by `(ts, blk, li)` into one stream.

**Day boundaries.** Each `tape-DAY` file runs from about 23:00 UTC on the previous day to about 23:00 UTC on DAY. Each `v4-swaps-DAY` file covers the UTC day.
- The V2/V3 set therefore starts at 2026-09-07T23:01Z and ends at 2026-09-26T23:00:49Z. The last hour of 2026-09-26 is in the next file, which is sealed and not included.
- `SEAL_SCAN.md` gives the first and last timestamp of every file.

**Completeness.** These files are not a full record of the chain:
- The V2/V3 files from 2026-09-07 to 2026-09-15 are missing 11–28 % of swaps per day. From 2026-09-20 the gap is 0–6 %. The cause was a lookup cap in the collector.
- The V4 files record all V4 pools from 2026-09-18 22:40Z (`src` = `wide`). Before that they hold only a fixed set of tracked pools (`src` = `lock`) plus some gap-filled rows (`src` = `hole`). Most V4 pools have no rows before 2026-09-18 22:40Z, even though they traded on chain.
- Treat a missing swap as "not recorded", never as "did not happen".

**Known row fault (V4).** Rows with `src` = `lock` are labelled with the token's main pool. A swap on another pool of the same token can therefore carry the wrong `pool`, the wrong price orientation and a flipped `side`. This affects about 3,355 rows up to 2026-09-18 and about 67 a day after. Guard against implausible prints rather than trusting single rows.

**Known row fault (V4, wide recorder).** On Pons pools quoted in another meme token (not ETH, WETH or USDG), the wide recorder can name the quote token as the coin. On those rows the coin, `side`, both amounts and the price are mirrored. Two pools are affected, 2026-09-18 to 2026-09-26: 0xfe1f5a89d9586d6431bdfe907acdebfc176bd5ceadc4f3ae9e0e284a3c022b91 (659 rows) and 0xc8c42c175c05eb92d3ec076683e6e650c902e16847aa5382f5ef6c855c61a482 (21 rows). The launchpad's own registration event names the coin, and the Pons lake release (tag pons-lake-2026-10-03) agrees with it. Found by an outside check (forum edge-hunt #385/#386), explained in #391.

## Row fields

**V2/V3 (`tape-*`):**
- `ts`: unix seconds.
- `blk`: block number.
- `li`: log index.
- `tx`: transaction hash.
- `tok`: token address.
- `sym`: the token symbol, as set by the token's creator. It is free text.
- `w`: the transaction sender (`tx.from`). For transactions sent through relays or bot services, this is the relay, not the account that received the token.
- `side`: buy or sell of `tok`.
- `usd`: trade size in USD.
- `px`: token price in USD.
- `kind`: v2 or v3.
- `quote`: the quote currency.

**V4 (`v4-swaps-*`):**
- `ts`, `blk`, `li`, `tx`, `tok`: as above.
- `pool`: V4 pool id.
- `quote`: the quote currency symbol. Native ETH is `ETH`; other values are WETH, USDG and token symbols.
- `quote_addr`: present when the quote is not ETH, WETH or USDG.
- `price_quote`, `price_eth`, `price_usd`: token price in the quote currency, in ETH and in USD.
- `side`: buy or sell of `tok`.
- `amount_token`, `amount_quote`: swap amounts.
- `usd`: trade size in USD.
- `eth_usd`: the ETH/USD rate used.
- `liquidity`, `tick`: pool state after the swap.
- `depth1_eth`, `depth5_eth`: ETH needed to move the price 1 % and 5 %.
- `pad`: venue label (`v4-hook`, `v4-nohook`, `doppler`, `pre-ledger`, named launchpads).
- `pad_key`, `launcher`: present on some rows.
- `hook`: the pool's hook contract.
- `birth_ts`: when the pool was created.
- `src`: the collector path that recorded the row: `wide` (all pools), `lock` (the tracked set) or `hole` (gap fill).

Price fields can be null when a swap could not be priced. A V4 pool quoted in a token other than ETH, WETH or USDG is priced through that token's own ETH reference, which can jump. Check such pools before trusting an extreme return.

All values come from public on-chain data. No row says who controls any address.

## The seal: nothing at or after 2026-09-27T00:00:00Z

Data from 2026-09-27T00:00:00Z onward (unix `1790467200`) is held back for pre-registered out-of-sample tests. Each later block is released only after its test has run.
- Every row of every file was parsed and checked. The latest row is `ts` 1790467199 (2026-09-26T23:59:59Z), and no row is at or after the seal. The per-file results are in `SEAL_SCAN.md`.
- Please do not collect or study data from on or after 2026-09-27 yourself while those tests are pending.

## Verify before use

Treat every downloaded file as untrusted:
1. Download the assets of release `tape-2026-09-26`, for example with `gh release download tape-2026-09-26 -R <this repo>` or from the release page.
2. Check every file against `MANIFEST.sha256` with `sha256sum -c MANIFEST.sha256`. Do not use a file that fails.
3. The files are data only. They need no keys, wallets or network access to read.
