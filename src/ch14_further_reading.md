# Further Reading

This book is the ground floor. If you want to go deeper, here's the shelf.

We've tried to pick sources that are (a) primary or close to primary, (b) still relevant as of 2026, and (c) written for engineers, not marketers. Each entry includes a sentence on what it is, why it's worth reading, and — where applicable — what to read it *after*.

## Founding papers

**"Bitcoin: A Peer-to-Peer Electronic Cash System"** — Satoshi Nakamoto, 2008.
Available at <https://bitcoin.org/bitcoin.pdf>. Nine pages; should take you about 45 minutes. Read this after Chapter 5. Section 11 (the catchup probability math) is the one we leaned on in Chapter 7. The paper rewards a second reading once you know the material.

**"Ethereum: A Next-Generation Smart Contract and Decentralized Application Platform"** — Vitalik Buterin, 2014 (white paper). The yellow paper — Gavin Wood's *"Ethereum: A Secure Decentralised Generalised Transaction Ledger"* — is the formal specification and is hard going, but indispensable if you want to understand EVM semantics in detail. Start with the white paper.

**"Gasper: A Combination of GHOST and Casper"** — Buterin, Hernandez-Delgado, et al., 2020.
The formal description of Ethereum's current consensus. Dense, but this is the actual spec. Read Chapter 6 and 7 first.

**"Tendermint: Byzantine Fault Tolerance in the Age of Blockchains"** — Ethan Buchman, 2016 (M.Sc. thesis).
The best single treatment of BFT-style PoS. Readable, careful, and correct.

## Classical consensus theory

You can skip these if you've taken a distributed systems course. If you haven't:

**"The Byzantine Generals Problem"** — Lamport, Shostak, Pease, 1982.
The paper that gave us Byzantine fault tolerance. The 3f+1 bound is here. Short and worth reading in the original.

**"Practical Byzantine Fault Tolerance"** — Castro & Liskov, 1999.
The paper that made BFT practical. Most modern BFT-style protocols are descendants of this work.

**"Impossibility of Distributed Consensus with One Faulty Process"** — Fischer, Lynch, Paterson, 1985.
The FLP impossibility result. Establishes that asynchronous deterministic consensus is impossible with even one crashing process — which is why every real consensus protocol assumes partial synchrony.

**"The Part-Time Parliament"** — Leslie Lamport, 1998 (or his more readable 2001 retelling, "Paxos Made Simple").
The original Paxos paper. Not blockchain-specific, but anyone reasoning about distributed agreement should have read it.

## Textbooks

**"Bitcoin and Cryptocurrency Technologies"** — Narayanan, Bonneau, Felten, Miller, Goldfeder, 2016.
Princeton textbook, built around a well-known Coursera course. Technical, academic, honest. The single best textbook introduction to the field as of 2026, though it doesn't cover PoS or rollups (both of which post-date it for the most part).

**"Mastering Bitcoin"** — Andreas Antonopoulos, 3rd edition 2023.
More engineering-focused than Narayanan et al. Less formal, more practical. Open source at <https://github.com/bitcoinbook/bitcoinbook>.

**"Mastering Ethereum"** — Antonopoulos & Wood, 2018.
Same shape as *Mastering Bitcoin*, for Ethereum. Pre-merge, so the consensus chapter is out of date, but the EVM and smart contract material is still excellent. Open source at <https://github.com/ethereumbook/ethereumbook>.

**"Token Economy"** — Shermin Voshmgir, 2019.
Less technical, more sociotechnical. Good background if you want to understand how tokens are used in systems, governance, and organization design. Pairs well with *Mastering Ethereum* — read one for the engineering, the other for the context.

## Research and long-form

**Vitalik Buterin's blog** — <https://vitalik.eth.limo/>.
A lot of what the research frontier of Ethereum looks like, written accessibly by the best-known researcher in the space. "Proof of Stake: How I Learned to Love Weak Subjectivity" (2014) is still one of the clearest statements of that concept.

**"The Economic Limits of Bitcoin and the Blockchain"** — Eric Budish, 2018.
A rigorous economic treatment of what it costs to attack a PoW chain and what that implies for long-run security. Cited in Chapter 5.

**"Selfish Mining: A 25% Attack Against the Bitcoin Network"** — Eyal & Sirer, 2013.
Shows that a miner with 25% of the hashrate can gain disproportionate block rewards by strategically withholding blocks. Important because it breaks the intuition that miners need 50% to misbehave profitably.

**ethresear.ch** — <https://ethresear.ch>.
Ethereum research forum. Long technical threads on the cutting edge: account abstraction, danksharding, single-slot finality, MEV. Not all of it is right, but it's where the live conversation happens.

**a16z crypto research** — <https://a16zcrypto.com/research/>.
Industry research with an investment lean; the writing quality varies but some pieces are excellent. Treat as supplementary.

**"Flash Boys 2.0"** — Daian et al., 2019.
The paper that coined MEV and made the field take transaction ordering seriously. Read after Chapter 9.

## Specific topics worth a separate deep dive

**Zero-knowledge proofs:**
- The Zero Knowledge Podcast (Anna Rose) covers the ZK research community in detail.
- Matthew Green's blog (<https://blog.cryptographyengineering.com>) for cryptographic context.
- The *Moonmath Manual* (ZK Hack) is an accessible introduction to the math underlying modern SNARKs.

**Rollups and scaling:**
- Polynya's blog (search "polynya rollups") for excellent, opinionated layperson treatments of the rollup landscape.
- Vitalik's "Endgame" post and the various data-availability posts on ethresear.ch.

**MEV:**
- Flashbots' documentation at <https://docs.flashbots.net>. Flashbots is the most significant practitioner organization in this space.
- "Ethereum is a Dark Forest" (Dan Robinson & Georgios Konstantopoulos, 2020) is the narrative account that made MEV widely understood.

**Security and smart contract auditing:**
- Secureum's (formerly Rarebits) study materials are the best structured introduction to Solidity security.
- The Rekt News site (<https://rekt.news>) catalogs major exploits with technical postmortems. Grim but educational.

## Things we didn't cover in this book and where to look

**Privacy-preserving chains (Zcash, Monero):** Zcash uses zk-SNARKs for shielded transactions; Monero uses ring signatures. Both have their own research communities and both are technically interesting. Zooko Wilcox's old writings on Zcash are a good start.

**Bridges and cross-chain messaging:** As of 2026 this remains the worst-audited corner of the ecosystem. Start with the LayerZero, Wormhole, and IBC documentation, then read rekt.news for a sense of the failure modes.

**On-chain governance:** The space of proposals (Compound Governor, Aragon, Snapshot) is broad and mostly pragmatic. Fred Ehrsam and Vlad Zamfir have written thoughtfully on the political-theory side.

**Stablecoins:** The most-used asset class on most chains by transaction volume. Design ranges from fiat-backed (USDC, USDT) to overcollateralized-crypto (DAI, LUSD) to algorithmic (most of which have failed — see the 2022 collapse of UST). Whitepapers are short; read them directly.

**Central bank digital currencies (CBDCs):** Mostly a different world — permissioned, regulated, state-issued. The BIS Innovation Hub publishes accessible technical briefs.

## How to keep learning

The ecosystem moves fast enough that any specific list goes out of date. What doesn't go out of date is the underlying material in chapters 1-7 of this book: the problem, Sybil resistance, the primitives, the consensus mechanisms, the finality questions. If you have those internalized, you can pick up any new paper or protocol by the twentieth page.

A reasonable cadence: every few months, pick one recent research paper (from ethresear.ch or a relevant academic venue) and read it end to end, working through anything unfamiliar until you understand it. Over two or three years of this, you'll have a working knowledge of the field at the level of the researchers, not the marketers.

And that, really, is the only way to stay honest about a field that generates marketing faster than it generates results. Read the primary sources. Derive the claims. Check the math. Keep your skepticism, keep your curiosity, and keep a good filter. That's the stance the book has tried to take, and it's the stance most likely to serve you.

Thanks for reading.
