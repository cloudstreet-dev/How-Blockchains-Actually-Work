# What Blockchains Are Actually Good For

We've now seen what blockchains are and how they work. The question this chapter is about is the one the whole book has been building toward:

> **When does using a blockchain actually make sense?**

The short answer: in a narrower set of cases than the marketing suggests, but in more cases than the reflexive skeptics allow. The job of this chapter is to draw that line cleanly, and give you tools to draw it yourself for cases we don't cover.

## The four-question test

Before anything else, here's the shortest checklist that cuts through 95% of bad blockchain pitches. When someone proposes using a blockchain for X, ask:

1. **Is there a single trusted party who could do this with a database?**
   If yes, use a database. A database is faster, cheaper, and easier to maintain than any blockchain. If no one can identify a trustworthy party willing to host the system and be accountable for it, you might need a blockchain.

2. **Who are the participants, and can they be identified in advance?**
   If you have a small set of known parties who just need to coordinate — a few banks, a supply-chain consortium, a government registry — you don't need open-membership Byzantine fault tolerance. A multi-primary database with Raft or classical BFT works fine. Skip the open blockchain.

3. **Is the problem actually about shared state, or about trust in a specific party?**
   "Notarize documents on a blockchain" is almost always an answer to a question that's really about trusting a notary. If you have a trustworthy notary (most jurisdictions do), use them. If you don't, a blockchain gives you an alternative *trust model*, not a substitute for the notary's local knowledge or legal standing.

4. **Is the cost of getting it wrong worth the cost of the system?**
   A blockchain imposes real costs: energy (PoW) or capital lockup (PoS), per-transaction fees, development complexity, worse performance than a centralized alternative, and an immutable history that can't be edited when you discover you want to. Is the value of *no-central-operator* trust worth paying for? For Bitcoin-as-digital-gold it clearly is. For most "blockchain for X" proposals, it isn't.

If your answer to question 1 is "yes, but we don't like them," that's often a legal or governance question masquerading as a technical question. Blockchain doesn't fix governance problems; it moves them.

## The honest yes-list

Cases where blockchain is genuinely the right tool:

### Censorship-resistant value transfer

The canonical case, and the one Bitcoin was invented for. You have a party that needs to send value to another party in a context where:

- Centralized payment processors might refuse the transaction (political dissidents, sanctioned-but-legal activity in different jurisdictions, etc.).
- Banking rails are unavailable (cross-border in low-infrastructure regions).
- The sender needs to be confident no third party can reverse or freeze the payment.

This is Bitcoin's original use case, and it's a real one. People in capital-controlled countries have used Bitcoin for legitimate cross-border savings. Journalists and activists have received donations when traditional channels blocked them. Refugees have moved wealth across borders when banks were inaccessible. The user experience is imperfect and the fees are non-trivial — but for the subset of transactions where this matters, blockchain is *the* answer.

"Digital gold" is the blunter version: a bearer asset with globally-verifiable ownership that doesn't require a jurisdictional home. Whether you think of Bitcoin as a currency or a commodity, the underlying property — *verifiable scarcity without a central issuer* — is the part that's genuinely new.

### Credibly neutral rule execution

If you need a set of rules enforced by code, transparently and immutably, with no single party having the authority to deviate, smart contracts are the right tool. Cases where this matters:

- **Automated market makers and lending**: once deployed, the rules don't change until a governance process (usually itself on-chain) amends them. No bank can unilaterally change the interest rate, no exchange can front-run your swap.
- **Escrow without a trusted escrow agent**: deposit funds; release conditional on verifiable events; refund if the conditions fail by a deadline.
- **Transparent prize pools, charitable giving, and bounties**: the rules are visible, the funds are visible, you don't have to trust the organizer not to pocket them.

The operative word is **credibly neutral**. If you need everyone to agree the rules will be followed without trusting any single rule-follower, that's when this matters. If you have a trustworthy operator, centralized infrastructure will always be cheaper and better.

### Auditable, tamper-evident logs

The hash-chain structure is useful as a standalone primitive, separate from open-membership consensus. If you're running an ordinary centralized system but want to publish a ledger of critical events that can be externally audited, you can do this on any blockchain — Bitcoin, Ethereum, or Cosmos-style chains all have this property. You post a hash of your event log periodically; later, you can prove what your log looked like at any given point by reference to the chain.

This is called **anchoring**, and it's used (quietly, without much marketing) by some enterprise audit systems, certificate transparency logs, and research data integrity schemes. It's a narrow, well-defined use case and it works.

### Coordination without a platform

The less obvious case, and the one worth thinking about carefully.

Suppose a large group of people want to coordinate around some shared state — a list of names, a registry of licenses, a roster, a shared document — and they specifically want to do so *without* appointing anyone to run the platform. This can be because:

- No one in the group is willing to trust anyone else to host.
- The group is too large for any single member to want the hosting responsibility.
- The coordination is politically or legally fraught and no one wants to be the operator of record.
- The group spans jurisdictions and no single legal framework cleanly covers all members.

A blockchain gives you a substrate for this kind of coordination that doesn't require a platform operator. You pay for it — every write is a transaction fee, every read that requires proof is some computation — but in exchange, the coordination mechanism has no one in charge.

This is the generalized version of the use cases above: DAOs, permissionless social networks like Farcaster's Hubs, some coordination games, certain kinds of open infrastructure. Whether these are "important" is a matter of taste; that they're possible and have been built is a matter of fact.

### Open infrastructure primitives

Ethereum Name Service (ENS), as a specific example, is a decentralized domain-name system for Ethereum addresses and identities. It could be implemented as a traditional database run by a foundation — and indeed the Handshake project tried the even-more-ambitious version of this for top-level domains — but the on-chain version has the property that registrations can't be revoked by an operator, transfers are cryptographically enforced, and the system works without requiring trust in any single party. About 3 million ENS names have been registered.

The Farcaster social protocol is another example: a decentralized identity layer with hubs that replicate state, where the "who owns this account" fact is enforced by an Ethereum contract.

These are infrastructure-layer uses: slow, expensive, valuable specifically because they can't be unilaterally shut off.

## The honest no-list

Cases where blockchain is almost always the wrong answer, despite the marketing:

### "Supply chain on the blockchain"

The pitch: track physical goods through the supply chain using a blockchain. The obvious flaw: a blockchain can tell you whether a *data entry* was tampered with, but it can't tell you whether the data entered in the first place was accurate. If the factory worker scans a box of fake goods as "authentic Rolex," the blockchain dutifully records that forever. You've gained immutable bad data.

The *legitimate* version of this is limited: using a blockchain as the audit-log substrate behind an existing trusted supply-chain system. That's anchoring, and it's useful, but it doesn't require a new blockchain — and it doesn't make the supply chain itself decentralized.

Vanishingly few supply-chain blockchain projects from the 2017-2022 wave are still in active use. Those that are, you'll find, are mostly replicated databases with a blockchain sticker.

### "Voting on the blockchain"

This one is worse than most people realize. The problems:

1. **Ballot secrecy is hard.** Blockchains are public by default. You can do zero-knowledge voting, but the cryptography is exotic and the trust assumptions shift in subtle ways.
2. **Voter authentication is fundamentally off-chain.** The blockchain cannot tell whether the person pressing the "submit vote" button is who they claim to be. You still need an identity registrar, and that registrar is now a central point of trust.
3. **Coercion and vote-selling become easier, not harder.** Traditional paper ballots make it hard to prove how you voted, which prevents vote buying. On-chain voting by default *can* be proved, which is exactly backward from what you want.
4. **Operational security is grim.** A voter's private key is effectively their vote forever. Losing it is disenfranchisement; having it stolen is fraud.

Real election security experts are, to a first approximation, unanimously against blockchain voting for public elections. There are cases where blockchain-like mechanisms make sense (corporate shareholder voting, DAO governance for organizations that literally exist on-chain) but public elections are not one of them.

### "Medical records on the blockchain"

HIPAA, GDPR, and their cousins exist because medical records need to be privacy-protected, access-controlled, and sometimes *deletable* (right to erasure). Blockchains are public, append-only, and immutable. These are the opposite of what you need.

What you can do with blockchains in healthcare: anchor attestations of record integrity, use off-chain systems for the actual records. This is a narrow, useful application — but nothing like "medical records on-chain," which is a bad idea for technical reasons that don't go away no matter how much ZK you stack on top.

### "Real estate titles on the blockchain"

The appeal is obvious — a tamper-proof registry of who owns what land. The problems:

1. Land registries are jurisdictional and legal. A blockchain can record that Alice transferred a token to Bob; it cannot make the court recognize that Bob now owns the land. Unless the legal system defers to the chain, the chain is decorative.
2. Land ownership is adversarial. Bugs, stolen keys, or transfers made under duress need to be reversible. Immutability is the wrong property here.
3. The existing registry systems, though often slow and paper-heavy, work. The bottleneck is political/institutional, not technical.

Pilots have run in several jurisdictions. None have replaced their official registry. This is probably the right outcome.

### "NFTs for X"

The technical mechanism — a non-fungible token representing an object or right — is fine. The question is always: *what does the token represent, and is that thing meaningfully better represented on-chain than in a database?*

For art: the token represents ownership of a particular instance of a digital work. This is interesting as an experiment in how digital property works. Whether it's a durable market, and whether the prices are rational, is a separate (and mostly-not-this-book's) question.

For tickets, credentials, and identity: if the issuer is trusted, a centralized system is simpler and provides better privacy. If the issuer is untrusted, the NFT gives you transferability but not much else — and for tickets specifically, transferability is often what the issuer doesn't want.

NFTs have a real narrow niche (digital ownership of pure-digital objects where you specifically want open secondary-market tradability) and a very wide not-niche (everywhere else). The proportion of all NFT projects in the wide-not-niche is large.

### "Put your loyalty program on the blockchain"

Loyalty programs are operated by a single company. The blockchain adds complexity without changing the trust model — you still trust the issuer to honor points, and the issuer can still change the terms. What it adds is transferability, which the issuer might or might not want, and a public record of all point balances, which is usually considered user-hostile.

If you find yourself in a meeting where this is being proposed, the right response is usually to ask what specific property of the blockchain is providing value, and to check whether a simpler system has that property.

## How to evaluate a blockchain pitch

If someone is pitching you on using a blockchain, work through these:

1. **Describe the system without the word "blockchain."** What's the actual flow, the actual participants, the actual data? If the description is still clear and useful, the blockchain might be adding something. If you can't describe it without the word, you're probably buying the label, not the technology.

2. **Ask what would break if you used a database run by the most trustworthy party in the system.** If the answer is "nothing meaningful," there's your answer.

3. **Ask where the critical off-chain inputs come from.** Every blockchain application that touches the real world has an **oracle problem**: someone has to tell the chain what's true about the world. If that someone is a single trusted party, you have a central point of failure regardless of what the chain does.

4. **Ask who can upgrade the code.** If the answer is "the team, via a proxy contract," you have a centralized system with a tamper-evident log. That's fine; just know what you have.

5. **Look at what happens when things go wrong.** Stolen funds, buggy contract, compromised key. What's the remediation? If the answer is "nothing, sorry," the system's immutability is a feature *and* a cost. Is the cost acceptable for the application?

6. **Look at the economics, not the technology.** Every blockchain application has a transaction-cost structure. If the users are paying fees for every interaction, who benefits from those fees, and does the value delivered exceed the costs paid? "Gas fees" have killed many a blockchain product that was technically interesting but economically unviable.

If a pitch survives all six of these, it might be a real application of the technology. Many pitches don't. Some do — and those are worth taking seriously.

## A takeaway

Blockchains are a narrow, expensive solution to a narrow, real problem: *reaching agreement on a shared log without designating anyone to run it*. Most problems are not that problem. For the problems that are, blockchains are remarkable — the existence of Bitcoin, a decentralized payment system with over a decade of continuous operation, was genuinely impossible before 2008 and remains impressive engineering now. For problems that aren't, they're an expensive costume.

The skill is telling which is which, and this chapter's checklist is the starting point.

One last category, the one the book has studiously avoided talking about, gets a very brief chapter of its own. That's next.
