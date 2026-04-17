# How Blockchains Actually Work (Without the Hype)

> Proof of Work vs Proof of Stake — real tradeoffs, real math, no hype.

A patient engineer's tour of consensus, finality, Sybil resistance, and why blockchains are a solution to a very specific problem. Skeptic-friendly. No token talk.

**Read the book:** https://cloudstreet-dev.github.io/How-Blockchains-Actually-Work/

## Why this book

Most blockchain writing is either marketing dressed as technology or academic papers written for people who already know. This book sits between: it assumes you know what a hash function is and what a distributed system looks like, and it walks you through the real engineering — including the parts that *don't* work, and the parts that are fine but oversold. Source is in [`src/`](src/). The book builds with mdBook and deploys to GitHub Pages via the workflow in [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml).

## Build locally

```
cargo install mdbook        # or: brew install mdbook
mdbook serve --open
```

## License

[CC0 1.0 Universal](LICENSE) — public domain dedication. Take it, remix it, teach from it.
