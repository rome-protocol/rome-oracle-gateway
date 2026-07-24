# Rome Oracle Gateway

> **Built on [Rome Protocol](https://docs.rome.builders)** — EVM chains that run natively inside the Solana runtime, where Solidity apps call Solana programs atomically (CPI) and Solana users drive EVM apps: two VMs, one chain, one block.

The Oracle Gateway delivers Solana-native price feeds (**Pyth**, **Switchboard V3**) to EVM smart contracts on Rome through the standard Chainlink **`AggregatorV3Interface`**. Ethereum protocols porting to Rome keep their existing oracle integration code **unmodified** while drawing on Solana's high-frequency oracle data.

The adapter contracts, factory, and batch reader are public and canonical in **[`rome-solidity/contracts/oracle`](https://github.com/rome-protocol/rome-solidity/tree/master/contracts/oracle)**. This repository is the Oracle Gateway's home for builders — architecture, the keeper model, consumer examples, and integration guidance.

## Chainlink-compatible by design

```solidity
import {IAggregatorV3Interface} from "@rome-protocol/rome-solidity/contracts/oracle/IAggregatorV3Interface.sol";

// The exact interface Ethereum DeFi already uses — no code changes.
(, int256 price, , , ) = IAggregatorV3Interface(ADAPTER).latestRoundData();
// price = e.g. SOL/USD at 8 decimals
```

## Consume a feed in three steps

**1. Resolve the adapter address** — never hardcode it. Feeds live in the [Rome registry](https://github.com/rome-protocol/rome-registry) at `chains/<id>/oracle.json`:

```ts
import { getOracle } from "@rome-protocol/registry";
const solUsd = getOracle(200010).feeds["SOL/USD"].address;
// Hadrian today: 0x76b92646D63FB1AFEa687C7Dac48b437bF99C1B4
```

**2. Read it** — the exact interface Ethereum DeFi already uses:

```solidity
import {IAggregatorV3Interface} from "@rome-protocol/rome-solidity/contracts/oracle/IAggregatorV3Interface.sol";

(, int256 price, , uint256 updatedAt, ) = IAggregatorV3Interface(SOL_USD).latestRoundData();
// price at 8 decimals — a stale or uninitialized feed REVERTS rather than serving a frozen price
```

**3. Reading several feeds?** Use the `BatchReader` (Hadrian: `0x306d670dff7f51ae33f263f5122bd2b18d98adc7`) — `getLatestPrices(adapters[])` returns them in one call, and `getFeedHealth(adapters[])` tells you they're live before you depend on them.

That's the whole integration. **[cardo](https://github.com/rome-protocol/cardo)** consumes these feeds in production today (its chain config carries the full Hadrian feed map). Because the surface is exactly `AggregatorV3Interface`, Compound- and Aave-class protocols are **drop-in** — their existing oracle code needs zero changes; point their price-feed configuration at the adapter addresses above.

## How it works — two read paths

Every adapter exposes `latestRoundData()` (the Chainlink interface) and is deployed as an **EIP-1167 minimal-proxy clone** by the `OracleAdapterFactory`.

- **Direct** (`PythPullAdapter`, `SwitchboardV3Adapter`) — read and parse the Solana price account **at call time**, straight off the source.
- **Cached** (`CachedPythAdapter`, `CachedFeedAdapter`) — a keeper writes the price on-chain and consumers read a **cheap stored value (a plain `SLOAD`)**. The parse happens once, on the keeper's update — so **every consumer read is inexpensive**, which matters when one transaction reads several feeds at once (e.g. a multi-collateral borrow).

`CachedFeedAdapter` wraps **any** `AggregatorV3Interface` source (a Pyth adapter, a Switchboard adapter, a Chainlink feed); `CachedPythAdapter` re-parses the Pyth account directly.

## The keeper — one worker, effectively free

A cached price is only as good as its last update, so a single off-chain **keeper** keeps every feed fresh. Each update is **one transaction that does two things**: it reads the verified on-chain Pyth account and writes the price on-chain (an EVM `SSTORE`), submitted from the keeper's synthetic address on the Solana lane. The cache is always a snapshot of the *verified* on-chain Pyth account.

**It costs almost nothing.** A live keeper update on Hadrian — [`0xcae2…ba26e`](https://via-hadrian.testnet.romeprotocol.xyz/tx/0xcae2adefda86c69ffa3e9c3f793ce606b2cf1f2e6dff4e0ec30a083f566ba26e) — settles for the **Solana base fee (~5,000 lamports)** with **no EVM gas**. One keeper keeps an entire chain's feeds fresh.

**Refresh cadence — devnet / testnet vs mainnet.** On Solana devnet and testnet (where Rome's public chains settle today), the keeper refreshes on a fixed short interval — **~25 seconds** currently. On **Solana mainnet**, Pyth Labs runs production keepers and feeds update at Pyth's native high frequency; Rome's keeper exists to fill the gap on the best-effort devnet/testnet clusters. The cadence shown here is the current devnet/testnet default and will track production frequency on mainnet.

**`refresh()` is permissionless.** Any address can call `refresh()` on a cached adapter — it reads the **canonical on-chain Pyth account**, so a caller cannot forge or bias the price; the worst they can do is pay gas to make a feed *fresher*. The keeper guarantees a baseline cadence, but a consumer that wants an up-to-the-moment price can refresh it itself in the same transaction.

## Feed health

Each adapter carries an extended metadata surface (`IExtendedOracleAdapter` / `IAdapterMetadata`) beyond `AggregatorV3Interface`. Consumers can check health programmatically with **`BatchReader.getFeedHealth(adapters[])`** rather than inferring it from `latestRoundData()` alone — and read many feeds in one call via `getLatestPrices` / `getLatestRoundDataBatch`.

## Constraints

- **`latestRoundData()` only.** `getRoundData(roundId)` reverts by design — no historical rounds.
- **Uninitialized / stale reverts.** A fresh clone reverts `UninitializedPriceFeed` until its first `refresh()`, and `StalePriceFeed` once the cached price ages past `maxStaleness` — it fails loud rather than serving a frozen price. Defaults: **3600s** for cached adapters, **~60s** for direct Pyth reads (configurable per environment).
- **EMA is Pyth-only.** Exponential-moving-average data is exposed on Pyth adapters; Switchboard V3 does not provide it natively.

## Addresses & registration

Canonical `OracleAdapterFactory`, `BatchReader`, and per-feed adapter addresses for every public Rome chain (**Martius**, **Hadrian**) live in the [Rome registry](https://github.com/rome-protocol/rome-registry) (`chains/<id>/oracle.json`). To register new feeds or configure a consumer, use the Oracle Gateway Portal.

## Learn more

- **[Oracle Gateway guide](https://docs.rome.builders)** — integration walkthrough and the Rome oracle model.
- **[Contracts (`rome-solidity/contracts/oracle`)](https://github.com/rome-protocol/rome-solidity/tree/master/contracts/oracle)** — the adapters, factory, parsers, and batch reader.
- **[Rome Solidity SDK](https://github.com/rome-protocol/rome-solidity)** — precompile interfaces and the rest of the on-chain toolkit.

## Building on Rome with an agent
See [`AGENTS.md`](./AGENTS.md) — the Rome-specific rules a coding agent needs.
