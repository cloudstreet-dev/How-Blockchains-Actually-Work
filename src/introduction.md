# Introduction: What This Book Is (And Isn't)

This book is a patient, honest tour of how blockchains work — the ideas, the math, and the engineering — written for someone who has watched the hype cycle for more than a decade and remains, quite reasonably, unconvinced.

It assumes you know what a hash function is, roughly what a distributed system looks like, and that "asynchronous" isn't a scary word. It does not assume you know Byzantine fault tolerance, the Nakamoto consensus rule, how Proof of Stake slashing works, or what a rollup is. By the end, you will.

## What this book is

It's a derivation. We start with a specific problem — how do you get a group of mutually suspicious strangers to agree on the order of events without anyone in charge — and build up from there. Every construction exists because the previous construction was missing something. When we introduce Proof of Work, it's because we showed, a chapter earlier, that voting-based consensus breaks under cheap identities. When we introduce Merkle trees, it's because we need efficient membership proofs over a set we don't fully trust. This is how the ideas actually arrived historically, and it's how they're easiest to understand.

It's also opinionated. When something in the ecosystem is genuinely clever — Merkle trees, Proof of Work as Sybil resistance, BFT consensus, the rollup construction — we'll say so directly. When something is marketing fluff dressed up as technology, we'll say that too. There are more of both than you might expect.

## What this book isn't

It isn't an investment guide. There is no "which chain should I bet on" chapter. There is no token analysis. There is exactly one chapter that talks about money at all, and its job is to explain why we don't talk about it more.

It isn't a history of scams, a political argument about monetary policy, or a defense of any particular community. The engineering is interesting on its own terms. That's the book.

It isn't exhaustive. Whole subfields — zero-knowledge proofs in depth, cryptoeconomic mechanism design, cross-chain bridges, DeFi primitives — get at most a pointer to further reading. The goal is that by the end, you can walk into a whitepaper and tell whether it's saying something real.

## How to read this

The chapters build. Chapter 2 explains why the problem in Chapter 1 is hard. Chapter 5 is Proof of Work, but Chapters 3 and 4 are what make it feel inevitable once you get there. If you jump straight to "Proof of Stake" because that's what you came for, you'll get less from it than if you read the five chapters before it.

The Python code in [Chapter 4](ch04_build_blockchain.md) is runnable. Type it in. Break it. The mechanics become physical in a way they never do from diagrams alone.

## On citations and dates

Numbers in the ecosystem move. Where we cite specific figures — hashrate, staking ratios, attack costs — we mark them "as of early 2026" and link to the source we took them from. If you're reading this in 2030, the numbers have shifted; the reasoning behind them should not have. Where we give a formula, the formula is the point. Plug in current numbers and the argument still works.

## A note on tone

You have a BS detector, and we respect it. This book will not tell you that blockchains are going to remake civilization. It will also not pretend that nothing interesting happened after 2008. The real story is narrower and more interesting than either extreme: a small pile of cryptographic tricks, a clever incentive structure, and a lot of engineering — most of it hard-won, some of it still unfinished.

Let's get to it.
