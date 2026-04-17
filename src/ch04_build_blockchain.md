# Build the Smallest Possible Blockchain (in Python)

Theory is easier to reason about once you've actually typed the thing in. This chapter builds a working blockchain in about 120 lines of Python, using only the standard library. It's small enough to fit in your head and real enough to exhibit every feature we've discussed: a genesis block, hash-linked blocks, signed transactions, a longest-chain fork resolution rule, and a naive Proof of Work puzzle. You can run it on your laptop in about a second.

The goal is that by the time you're done reading this chapter, nothing in a blockchain feels mysterious. After this, we zoom out again and talk about what the production systems do differently — and why.

## Ground rules for the toy

We are going to cheat in a handful of places, deliberately and clearly, so the code stays small:

- We use SHA-256 for hashing and the standard library's `ecdsa`-free `hashlib` — but we'll swap in a real signature scheme from `cryptography` (a well-maintained library) for transactions, because hand-rolling elliptic curves in a tutorial is a bad plan.
- We ignore networking. The "chain" is a single Python object. Peer gossip, mempool propagation, and fork detection across a network are important, but they are engineering on top of the ideas we're showing, not the ideas themselves.
- Our Proof of Work is real (SHA-256 with a difficulty target), but the difficulty is low so it mines in milliseconds. Adjusting it to mainnet-scale is a one-line change.
- Fees, UTXOs, and smart contracts don't exist here. We model account balances as a dict, which is closer to Ethereum's state model than to Bitcoin's UTXO model, but simpler than either.

None of the cheats change the fundamental mechanics. They just let us ship 120 lines instead of 2,000.

## The code

Save this as `tinychain.py`. You'll need the `cryptography` library: `pip install cryptography`.

```python
# tinychain.py — a minimal pedagogical blockchain.
# Not for production. Not audited. Runs in a second.

from __future__ import annotations
import hashlib, json, time
from dataclasses import dataclass, field, asdict
from typing import List, Optional
from cryptography.hazmat.primitives.asymmetric import ed25519
from cryptography.hazmat.primitives import serialization
from cryptography.exceptions import InvalidSignature

# ---------- crypto helpers ----------

def sha256(b: bytes) -> bytes:
    return hashlib.sha256(b).digest()

def pk_bytes(pk: ed25519.Ed25519PublicKey) -> bytes:
    return pk.public_bytes(serialization.Encoding.Raw,
                           serialization.PublicFormat.Raw)

def address(pk: ed25519.Ed25519PublicKey) -> str:
    # 20-byte address = first 20 bytes of SHA-256(public key).
    return sha256(pk_bytes(pk))[:20].hex()

# ---------- transactions ----------

@dataclass
class Tx:
    sender: str         # address
    recipient: str      # address
    amount: int
    nonce: int          # prevents replay: strictly increasing per sender
    pubkey: str         # hex of sender's public key, so verifiers can check
    signature: str = "" # hex; filled in by sign()

    def _signing_bytes(self) -> bytes:
        # Signature covers everything except the signature itself.
        d = {k: v for k, v in asdict(self).items() if k != "signature"}
        return json.dumps(d, sort_keys=True).encode()

    def sign(self, sk: ed25519.Ed25519PrivateKey) -> None:
        self.signature = sk.sign(self._signing_bytes()).hex()

    def verify(self) -> bool:
        try:
            pk = ed25519.Ed25519PublicKey.from_public_bytes(bytes.fromhex(self.pubkey))
        except Exception:
            return False
        if address(pk) != self.sender:
            return False  # pubkey doesn't match claimed sender address
        try:
            pk.verify(bytes.fromhex(self.signature), self._signing_bytes())
            return True
        except InvalidSignature:
            return False

# ---------- blocks ----------

@dataclass
class Block:
    height: int
    prev_hash: str           # hex of parent block's hash, or 64 zeros for genesis
    timestamp: float
    txs: List[Tx]
    difficulty: int          # number of leading zero bits required in block hash
    nonce: int = 0

    def header_bytes(self) -> bytes:
        # Merkle root would go here in a real chain; we hash the tx list directly
        # because it's shorter and the tradeoffs aren't the point of this chapter.
        tx_hash = sha256(json.dumps([asdict(t) for t in self.txs],
                                    sort_keys=True).encode()).hex()
        d = {"height": self.height, "prev_hash": self.prev_hash,
             "timestamp": self.timestamp, "tx_hash": tx_hash,
             "difficulty": self.difficulty, "nonce": self.nonce}
        return json.dumps(d, sort_keys=True).encode()

    def hash(self) -> str:
        return sha256(self.header_bytes()).hex()

    def meets_difficulty(self) -> bool:
        # Interpret the hash as a big integer; require it to be below 2^(256-d).
        return int(self.hash(), 16) < (1 << (256 - self.difficulty))

    def mine(self) -> None:
        while not self.meets_difficulty():
            self.nonce += 1

# ---------- chain ----------

@dataclass
class Chain:
    blocks: List[Block] = field(default_factory=list)
    difficulty: int = 16  # ~65k hashes per block on average; fast on a laptop

    def genesis(self) -> None:
        g = Block(height=0, prev_hash="0" * 64, timestamp=0.0,
                  txs=[], difficulty=self.difficulty)
        g.mine()
        self.blocks = [g]

    def head(self) -> Block:
        return self.blocks[-1]

    def balances(self) -> dict:
        # Derived state: replay all transactions from genesis.
        bal: dict = {}
        for b in self.blocks:
            for t in b.txs:
                bal[t.sender] = bal.get(t.sender, 0) - t.amount
                bal[t.recipient] = bal.get(t.recipient, 0) + t.amount
        return bal

    def add_block(self, txs: List[Tx]) -> Block:
        b = Block(height=self.head().height + 1,
                  prev_hash=self.head().hash(),
                  timestamp=time.time(),
                  txs=txs,
                  difficulty=self.difficulty)
        b.mine()
        assert self.is_valid_next(b)
        self.blocks.append(b)
        return b

    def is_valid_next(self, b: Block) -> bool:
        p = self.head()
        if b.height != p.height + 1: return False
        if b.prev_hash != p.hash():   return False
        if not b.meets_difficulty():  return False
        for t in b.txs:
            if not t.verify():        return False
        # (A real chain also checks: no double-spend, sufficient balance,
        #  monotone nonces, sane timestamps, size limits, etc.
        #  We skip those here for brevity; adding them is mechanical.)
        return True

    def is_valid(self) -> bool:
        if not self.blocks: return False
        if self.blocks[0].prev_hash != "0" * 64: return False
        for i in range(1, len(self.blocks)):
            p, b = self.blocks[i-1], self.blocks[i]
            if b.prev_hash != p.hash(): return False
            if not b.meets_difficulty(): return False
        return True

    @staticmethod
    def choose_longest(a: "Chain", b: "Chain") -> "Chain":
        # Fork resolution: the chain with more total work (here: more blocks,
        # since difficulty is constant) wins. This is the Nakamoto rule.
        if not a.is_valid(): return b
        if not b.is_valid(): return a
        return a if len(a.blocks) >= len(b.blocks) else b

# ---------- demo ----------

if __name__ == "__main__":
    alice_sk = ed25519.Ed25519PrivateKey.generate()
    bob_sk   = ed25519.Ed25519PrivateKey.generate()
    alice_pk, bob_pk = alice_sk.public_key(), bob_sk.public_key()
    alice, bob = address(alice_pk), address(bob_pk)

    chain = Chain()
    chain.genesis()

    # Block 1: mint 100 to Alice (sender = "coinbase"; we skip signing for the mint).
    mint = Tx(sender="coinbase", recipient=alice, amount=100,
              nonce=0, pubkey="", signature="")
    # Our verify() rejects "coinbase" because its signature is empty,
    # so for the toy we smuggle it in and accept it by convention:
    b1 = Block(height=1, prev_hash=chain.head().hash(),
               timestamp=time.time(), txs=[mint], difficulty=chain.difficulty)
    b1.mine()
    chain.blocks.append(b1)

    # Block 2: Alice sends 30 to Bob, properly signed.
    t = Tx(sender=alice, recipient=bob, amount=30, nonce=1,
           pubkey=pk_bytes(alice_pk).hex())
    t.sign(alice_sk)
    chain.add_block([t])

    print("valid:", chain.is_valid())
    print("balances:", chain.balances())
    for b in chain.blocks:
        print(f"  block {b.height}: hash={b.hash()[:16]}... nonce={b.nonce}")
```

Run it:

```
$ python3 tinychain.py
valid: True
balances: {'coinbase': -100, 'e3b0...': 70, 'a9f1...': 30}
  block 0: hash=0000f3c7... nonce=42137
  block 1: hash=0000aa21... nonce=18890
  block 2: hash=0000bd53... nonce=95104
```

(The addresses and nonces will differ on your run, but the block hashes will all start with several zero hex digits — that's the Proof of Work.)

## What's actually happening here

Read the code once through. Then read it again with these annotations in mind:

- **Hash chain.** `Block.prev_hash` makes each block cryptographically dependent on its parent. Alter any earlier block and every later block's `prev_hash` field becomes wrong. The `is_valid` loop is what enforces that.

- **Proof of Work.** `mine()` is a `while` loop that increments `nonce` until `block.hash()` happens to fall below a threshold. For difficulty 16, that threshold is `2^(256-16) = 2^240`, which means roughly 1 in `2^16` (`65,536`) nonces will work. On a laptop this finishes in milliseconds. Bitcoin's current mainnet difficulty would require roughly `2^80` attempts per block — 20 orders of magnitude harder. The mechanism is identical; only the constant changes.

- **Transactions are self-authenticating.** A transaction carries its own public key and signature. Any node can verify it offline, without consulting a registry. `address(pk) != sender` is the check that prevents someone from pasting a valid signature from the wrong key.

- **Nonce in the transaction.** Not to be confused with the *mining* nonce. The transaction nonce prevents replay: once Alice uses nonce `1`, a second transaction with nonce `1` is rejected. Without this, an attacker could rebroadcast Alice's old transaction and spend her money twice. Real chains enforce strictly monotonic per-sender nonces; our toy leaves that as a hole you'd fix in ten lines.

- **State is derived.** `balances()` doesn't live in a field — it's computed by replaying every transaction. This is how every blockchain works at a fundamental level: the *state* is a pure function of the *history*. Nodes cache the state for speed, but it's always reproducible from scratch.

- **Fork resolution is a single rule.** `choose_longest` says: when given two valid histories, take the one with more accumulated work. That's it. There is no committee, no election, no tiebreaker beyond "which is longer." This is the Nakamoto consensus rule in one function.

## What a fork looks like in practice

Forks happen when two miners produce a valid block at the same height, nearly simultaneously, and their networks propagate them in opposite directions. Half the network hears block A first; half hears block B. Both chains are valid. The rule says: keep mining on whichever you heard first, and if a longer chain appears later, switch to it.

We can simulate it in four lines:

```python
c1 = Chain(); c1.genesis(); c1.add_block([])
c2 = Chain(); c2.blocks = list(c1.blocks)  # shared history so far
c1.add_block([]); c1.add_block([])         # c1 is now ahead by 2
c2.add_block([])                            # c2 only extends by 1
winner = Chain.choose_longest(c1, c2)
print("winning length:", len(winner.blocks))  # 4
```

In a real network, nodes would continuously broadcast their longest known chain; the losing fork's transactions would rejoin the mempool, to be included in some future block. The losing block is called an **orphan** (or, more precisely, a "stale" block). Orphans happen naturally — on Bitcoin, the historical orphan rate has been roughly 1–2% of blocks depending on network conditions, and the difficulty is tuned so that orphans stay rare but nonzero.

## What our toy is missing, and what it matters

To make this production-grade, you would add:

- **Networking.** Peer-to-peer gossip, block/transaction propagation, peer discovery. This is the bulk of a real client.
- **A proper transaction model.** Either a UTXO set (Bitcoin-style) with explicit inputs and outputs, or an account/state tree (Ethereum-style) with a Merkle-Patricia trie for efficient proofs.
- **Difficulty adjustment.** A target block time (10 minutes in Bitcoin, 12 seconds in Ethereum before the merge) and a formula that retunes difficulty every `N` blocks based on how long the last `N` took. We'll do this one on paper in Chapter 5.
- **Fee markets.** Our blocks accept any transaction for free. Real chains use fees to rank transactions and to pay miners/validators. Fee design is an active research area.
- **Finality rules.** Our `choose_longest` never considers a block "final" — a long enough reorg could always replace it in principle. Chapter 7 goes into what finality actually is, and why different chains choose different flavors of it.

None of those change the core idea. They change the constants, the data structures, and the engineering. The shape you just built — *signed transactions, hash-linked blocks, a puzzle, a fork-choice rule* — is every blockchain in existence, with varying amounts of additional machinery bolted on.

The next three chapters open up the two most important pieces: the puzzle (Chapter 5: Proof of Work), its capital-based alternative (Chapter 6: Proof of Stake), and the question of when, exactly, a block is safe to trust (Chapter 7: Finality).
