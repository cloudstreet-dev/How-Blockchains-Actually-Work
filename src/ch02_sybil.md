# Sybil Attacks: Why Identity Is the Hard Problem

The previous chapter ended with an observation: classical Byzantine agreement assumes a fixed, known set of participants, and in an open system you don't have that. This chapter is about why that one assumption — "we know who the participants are" — is load-bearing, and what has to change when you take it away.

The term **Sybil attack** comes from a 2002 paper by John Douceur, named after a 1973 book about a woman with dissociative identity disorder. The idea is simple: if identities are cheap to create, a single attacker can manufacture as many as they need to dominate any system that weights participants equally.

## The setup

Suppose we have a voting-based consensus protocol. Participants vote on transactions, and we accept whichever outcome the majority prefers. This is the intuitive, democratic approach, and it's exactly what you'd reach for first.

For it to work, "majority" has to mean something. In a classical setting — a set of servers in a datacenter, say — it does. You configured 5 servers; 3 is a majority; done. Each server has one vote. Fine.

Now move to the open internet. "One person, one vote" requires that you can tell people apart. On the internet, you cannot. A laptop can create ten thousand Ethereum-style addresses in a second. Each one can sign messages. Each one looks, to the protocol, like a first-class participant. If you accept votes from anyone, the attacker votes ten thousand times and outvotes everyone else combined.

You might object: surely we can tell if addresses are controlled by the same person? We cannot. Not without external identity infrastructure. There's no observable signal that distinguishes "ten people each with one address" from "one person with ten addresses." They behave the same on the wire. They produce valid signatures. They pay fees. From the protocol's perspective, they are indistinguishable.

This is the Sybil problem, and it is not a clever trick you can engineer around with more code. It's a structural consequence of the fact that *identity on the internet is free*.

## Why "just require identity" doesn't work

A first instinct: make identities expensive. Require each participant to prove who they are — passport, phone number, KYC. Centralized systems do this. It works.

But requiring identity puts someone in charge of saying who's allowed to vote. That someone is a trusted third party — the one thing we declared, in Chapter 1, that we don't have. If we had a trusted identity registrar, we'd just have them run the database. We wouldn't need any of this.

You can imagine hybrid designs, and people have built them. You can require a real-world identity provider to sign off on each blockchain identity. The result is a permissioned system, which is a perfectly reasonable engineering choice for many use cases, and which we cover briefly in the ecosystem chapter. What it isn't is a solution to the original problem. It is a *different* problem, solved by moving the trust somewhere else.

So if we want the open-membership property — anyone can join, no registrar — then we cannot use identity as the thing that makes Sybil attacks expensive.

## The reframe: what's scarce?

Here is the move. If identity is not scarce on the internet, we need to find something else that is — something an attacker cannot costlessly replicate — and tie participation to *that*.

This is the Sybil resistance requirement, and it's the reframe that makes everything that follows possible:

> **To vote in an open-membership consensus, you must prove you've consumed a resource that is genuinely scarce.**

That's it. Every consensus mechanism in this book is, at bottom, a choice of what scarce resource to use.

- **Proof of Work** uses *computation*. To participate, you must show you've burned a measurable quantity of computational effort. Electricity and silicon are scarce in a physical sense; you cannot fake having computed a hash.

- **Proof of Stake** uses *capital*. To participate, you must show you've locked up a quantity of the system's native token. Acquiring that capital is necessarily expensive, because it's priced by a market that would refuse to sell it for zero.

- **Proof of Space**, **Proof of Elapsed Time**, **Proof of Authority**, and various more exotic schemes use other scarce resources — storage, trusted-hardware attestations, or a closed set of pre-approved validators. Each has its own tradeoffs. The first two dominate in practice.

In every case, the logic is the same: you cannot manufacture more votes than you can manufacture resource. And the resource has been chosen specifically because faking it is as hard as actually having it.

## What Sybil resistance buys you

Once you have Sybil resistance, you can run voting-based consensus again — but the votes are weighted by *resource consumed*, not by *address count*. An attacker with 10% of the hashrate has 10% of the votes, whether they split it across one node or a million. The thousand-identity attack no longer helps, because splitting the same hashrate across more identities doesn't give you more total hashrate.

This lets you port over the classical Byzantine agreement results, with one critical substitution. The `3f + 1` bound becomes, roughly:

> Honest participants must control more than `2/3` (for BFT-style PoS) or more than `1/2` (for Nakamoto-style PoW) of the **scarce resource**, not more than 2/3 or 1/2 of the *nodes*.

This is why, when you read news about a "51% attack" on Bitcoin, it's 51% of the hashrate — not 51% of the users, or nodes, or wallets. The voting unit is the hash.

## The cost we just accepted

Notice what we've done. We have made it expensive to participate. Previously, spinning up a node was free; now, it requires burning electricity or locking up capital. We have explicitly chosen to put a cost floor under participation, because that cost floor is also the security floor.

This is the part the skeptics are most often right about and the evangelists most often flinch from. The energy consumption of Proof of Work is not an implementation inefficiency you can engineer away. It's the security mechanism. The capital lockup in Proof of Stake is not a nuisance — it's what makes the vote count. You can argue about which cost is worth paying, and in Chapter 8 we do, at length. What you cannot do is remove the cost without removing the security it's buying.

If anyone ever tries to sell you a blockchain protocol that's both open-membership and "free to participate in," check whether they've really solved the Sybil problem or just hidden it. Usually they've hidden it — often behind a small set of pre-approved validators, which is a perfectly fine answer, but a different system than it's being marketed as.

## Where this leaves us

We now have a sharpened problem statement:

> Open-membership Byzantine agreement on an ordered log, using a scarce resource as the unit of voting weight.

That's the shape. The next chapter lays out the cryptographic primitives we'll need to build it. The one after that actually builds a toy version, in about 100 lines of Python, so the mechanics become physical before we try to reason about them abstractly.

From here on, whenever you read about a consensus mechanism, ask yourself: *what is the scarce resource, and how is participation tied to it?* That single question cuts through a startling amount of marketing noise.
