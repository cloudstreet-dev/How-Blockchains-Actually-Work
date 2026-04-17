# The Ecosystem Survey

You can understand the whole blockchain landscape from the consensus and scaling primitives in the previous chapters. This one is a field guide: what do the production chains actually do, and where do they sit on the tradeoff curves we've drawn?

This isn't exhaustive. There are hundreds of chains. Most of them are minor variations on one of the below, or outright copies. The ones here are the ones you will actually encounter.

## Bitcoin: the minimalist

- **Consensus**: Nakamoto-style Proof of Work.
- **Block time**: ~10 minutes target, exponentially distributed.
- **Smart contracts**: intentionally not Turing-complete. Bitcoin Script supports a small set of operations — signature checks, time locks, multi-sig — that are enough for programmable payments but not for arbitrary state. This was a deliberate design choice.
- **Typical throughput**: 3–7 transactions per second on the base chain.
- **Design philosophy**: do one thing — sound, censorship-resistant digital money — and resist every feature that could broaden the attack surface.

Bitcoin is the conservative reference point for the whole ecosystem. It is the only chain with an unbroken track record from 2009, the only one with no effective leadership (Satoshi is gone; no single person or foundation can unilaterally ship changes), and the only one where "don't change anything that isn't absolutely necessary" is a serious, coherent engineering position rather than conservatism by default.

The main Bitcoin L2 is the **Lightning Network**, a payment-channel network that lets users open bilateral channels backed by Bitcoin deposits and route payments through them. Lightning is fast and cheap for routine payments but has real UX and liquidity-routing issues for larger or less-common transfers. It's grown steadily and real merchants accept it, but it remains a minority use case.

## Ethereum: the generalist

- **Consensus**: Proof of Stake (post-Merge, September 2022). Hybrid Gasper design — LMD-GHOST for head + Casper FFG for finality.
- **Block time**: 12 seconds per slot, 32 slots per epoch; finality via FFG in ~2 epochs (~6.4 minutes).
- **Smart contracts**: EVM, Turing-complete, with gas as the DOS defense.
- **Typical base-chain throughput**: 15–25 transactions per second; much higher effective throughput via L2 rollups.
- **Design philosophy**: general-purpose programmable chain, with ongoing upgrades to improve scalability, data availability, and validator decentralization.

Ethereum is the center of gravity for everything that isn't Bitcoin. The EVM is, by a wide margin, the most-implemented virtual machine across other chains (most of the "L1 competitors" below implement EVM compatibility specifically to capture Ethereum's developer mindshare). The main research frontiers — rollups, data availability, MEV, account abstraction — mostly originate in or around Ethereum's research community.

The pace of change is the tradeoff: Ethereum has had more upgrades in a decade than Bitcoin has had in nearly two. Some of those upgrades shipped bugs that were caught and patched in the wild. This is a real difference in operational posture, and reasonable people disagree about which posture is correct for what.

## Solana: high throughput, different tradeoffs

- **Consensus**: Proof of Stake with a specialized leader-based scheme called "Tower BFT," plus "Proof of History" — a verifiable-delay-function-based timestamping scheme that orders transactions before they're voted on.
- **Block time**: ~400 ms target.
- **Smart contracts**: custom runtime (Sealevel), mostly programmed in Rust. Not EVM-compatible.
- **Typical throughput**: high — often cited at thousands of tps peak, more realistically a few hundred to low thousands sustained. The headline "65,000 tps" you sometimes see is a benchmark number, not a real-world observation.
- **Design philosophy**: trade hardware-cost decentralization for throughput. Solana validators run on substantially beefier hardware than Ethereum validators, and there are far fewer of them (hundreds, not thousands).

Solana has had several network outages — periods of hours where no new blocks finalized — that are rare on Ethereum or Bitcoin. The outages have been patched and the protocol is more stable than it was in 2021-22, but the tradeoff is visible: optimizing for throughput has ongoing liveness costs.

The Solana ecosystem is the most significant non-EVM chain with real user activity, particularly in areas like consumer payments, meme trading (for better or worse), and NFT activity.

## Cosmos: the multi-chain application-specific model

- **Consensus**: Tendermint / CometBFT, giving deterministic BFT finality in 1–7 seconds.
- **Smart contracts**: CosmWasm (Rust-based WebAssembly) on chains that support it; many Cosmos chains don't support generic smart contracts and instead implement their specific application directly as chain logic.
- **Interoperability**: IBC (Inter-Blockchain Communication) — a standardized protocol for chains to send packets to each other, secured by each chain's own validators.
- **Design philosophy**: each application gets its own chain, optimized for that application, with IBC as the connective tissue. "Application-specific chains," or "appchains."

The Cosmos view is that general-purpose smart-contract chains are a category error — you're running every application on the same slow, expensive, congested computer. Better to give each application its own chain. Cosmos Hub, Osmosis (a DEX), Celestia (DA layer), dYdX (a perpetual-futures exchange), and many others exist as independent Tendermint chains that communicate via IBC.

Whether this works long-term depends on whether the interop (IBC) is rich enough and fast enough to substitute for having everything live in a single shared state. As of early 2026, IBC is reliable for asset transfers but awkward for more complex composition. The jury is still out on whether appchains dominate or smart-contract-chains dominate, and it's possible the answer is "both, for different things."

## Private / permissioned: Hyperledger, Fabric, Quorum

- **Consensus**: permissioned (known validator set), typically BFT-flavored (Raft, PBFT, IBFT variants).
- **Smart contracts**: varies. Hyperledger Fabric has chaincode in Go/JavaScript; Quorum is EVM; others have their own environments.
- **Design philosophy**: blockchain-shaped data structures for enterprise use cases — shared ledgers between known consortium members, not open-membership.

It's worth being blunt about these: **if the validator set is permissioned and pre-approved, you don't need a blockchain.** You have a replicated database, which the industry has known how to build since the 1990s. The database will be faster, cheaper, easier to debug, and no less secure — possibly *more* secure, because its attack surface is much smaller.

So why do these exist? Two reasons, honestly:

1. **Multi-party coordination without designating a single operator.** If three banks want a shared ledger and none of them wants to host it on behalf of the others, a permissioned blockchain gives you a rotating-leader BFT protocol across their three datacenters. This is a genuine use case, and a permissioned blockchain is one reasonable answer to it. But a well-configured multi-primary distributed database is another reasonable answer, and the blockchain label doesn't add much.
2. **Marketing / procurement.** In the 2016-2020 period, "blockchain for enterprise" was a hot procurement line, and Hyperledger Fabric was the default non-crypto answer. Many of these projects quietly became replicated databases with a blockchain sticker on them.

You'll still see Hyperledger projects in logistics, supply chain, and cross-bank settlement. Some of them do real work; some are long-running pilots that never graduated. Evaluate on the merits of the specific use case, not on the label.

## Rollup ecosystems on top of Ethereum

The rollup chapter introduced the leading rollups. A quick recap of positioning:

- **Arbitrum One** (Offchain Labs): optimistic rollup, EVM-equivalent, largest TVL among Ethereum L2s as of early 2026. Sequencer is centralized. Challenge window is 7 days.
- **Optimism / OP Mainnet** (Optimism Labs): optimistic rollup, EVM-equivalent. Spawned the **OP Stack**, a framework other projects use to launch rollups (Base by Coinbase, Worldcoin's chain, and others). "Superchain" is the branded vision of many OP-stack rollups interoperating.
- **Base** (Coinbase): OP-stack optimistic rollup, operated by Coinbase. Grew rapidly in 2024-25 thanks to Coinbase's distribution. Sequencer centralized (Coinbase runs it).
- **zkSync Era** (Matter Labs): ZK rollup, largely EVM-compatible but with some quirks (it doesn't use the standard Ethereum account model in the same way).
- **Starknet** (StarkWare): ZK rollup based on Cairo, *not* EVM. Uses its own language for efficient provability. Smaller ecosystem than EVM rollups but technically interesting.
- **Scroll, Linea, Polygon zkEVM**: other EVM ZK rollups, each with slightly different implementation choices and slightly different trust assumptions at present.

All of these have *some* centralization at the sequencer level today. Decentralizing sequencers is an active roadmap item for all of them. When someone tells you a specific rollup is "fully decentralized," check when they rotated off a single sequencer. As of early 2026, few have.

## Layer-1 competitors you'll hear about

A partial list of chains that market themselves as "Ethereum alternatives" or "next-generation L1s." We're not going to go deep on each; the structural story is similar.

- **Avalanche**: three-chain architecture (X, P, C chains); the C-chain is an EVM using Snowball/Avalanche consensus (novel metastable protocol). Fast finality, EVM-compatible.
- **Near**: sharded state across "shards" (currently 4, designed for many more). WASM runtime. Different developer experience than EVM.
- **Polkadot**: multi-chain "parachain" architecture with shared security via relay chain. Substrate framework for building chains.
- **Cardano**: PoS (Ouroboros protocol). UTXO model extended with scripts (Plutus/Aiken). Smaller developer ecosystem than EVM chains.
- **Aptos / Sui**: HotStuff-derived PoS chains using the Move language, originally developed for Meta's (aborted) Diem project.
- **Tron, BNB Smart Chain, others**: EVM-compatible chains with various degrees of centralization, typically optimized for low-cost DeFi activity. BSC, in particular, has been subject to criticism for the small size of its validator set (21 validators, most affiliated with Binance).

There is a persistent pattern: new L1s launch, grow quickly on a combination of technical claims and token incentives, and either fade or settle into a narrow niche. The survivors tend to be the ones with clear technical differentiation (Solana's throughput, Sui's object model) or strong distribution partners (BSC via Binance). The ones without either have mostly faded.

## What's actually different vs. what's marketing

When evaluating a new chain, the actually-meaningful questions are:

1. **What's the consensus mechanism, and what's the cost of attack?** If they can't tell you this in one sentence, they don't know.
2. **What's the validator set size and concentration?** 21 validators owned by one company is not the same thing as 800,000 Ethereum validators.
3. **What are the finality properties? Probabilistic, deterministic, hybrid?**
4. **Is state fully re-executable by anyone with commodity hardware?** If not — if they're claiming high throughput by requiring specialized hardware or a small validator set — what's the tradeoff they're hiding?
5. **What's the data availability model? Where are transaction data stored, for how long, and under what trust assumptions?**
6. **What's the upgrade process? Who can change the protocol rules, and how?**

If those six questions have clear answers and the answers are reasonable, the chain is at least coherent. Whether it's *good* depends on what you're trying to do with it.

Marketing differentiators that don't matter as much as the above:
- TPS benchmark numbers (usually best-case, rarely sustained).
- "Modular vs monolithic" labels (architectural language that can hide weak security assumptions).
- "Quantum-resistant" claims (the cryptography used in all major chains will need an upgrade path when large quantum computers arrive; the specific primitive is much less important than whether the team has an upgrade plan).
- Celebrity endorsements, partnership announcements, "enterprise integrations" — these tell you about the business development team, not the technology.

The field is full of chains that are technically capable and socially moribund, and chains that are technically weak but loud. Neither category is the one you want. Look for the boring, functioning, economically durable systems.

## What's next

Now we have enough context to tackle the harder question: *what is any of this actually for?* That's the next chapter, and it's the part where we stop describing what exists and start saying which applications make sense and which don't.
