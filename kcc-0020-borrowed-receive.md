# KCC20 Borrowed Receive Guide

Borrowed Receive is defined in
[KCC20 Section 5](kcc-0020.md#5-borrowed-receive).

## Motivation

KIP-9 requires each new UTXO to include KAS for storage.

For token transfers, this means the sender must fund every new recipient token
UTXO, or the recipient must co-sign and provide the required KAS.

Borrowed Receive allows the sender to use an existing recipient KCC20 UTXO as
the receive target. Instead of creating a new recipient token UTXO, the sender
consumes the existing UTXO and recreates it in place with a larger token amount.
The recipient's normal owner authorization is not used, the KAS value cannot
decrease, and the owner and extended state remain unchanged.

Because the existing UTXO is consumed, every borrowed receive changes its
outpoint. Without restrictions, an attacker could repeatedly send tiny token
amounts to churn that outpoint. A wallet, signing device, or service relying on
its locally known UTXOs would then have to reconnect and rediscover the current
UTXO before using it.

The borrow scheme controls who may cause this limited state change, or under
what conditions it is allowed. It cannot remove the need to learn the new
outpoint after an actual borrow, but it can prevent arbitrary or low-value
borrows from making locally stored UTXO information stale.

## Borrow authorization

Each KCC20 state contains a `borrow_scheme` and a 32-byte `borrow_guard`.
`borrow_scheme` selects the authorization rule, while `borrow_guard` holds its
parameter or evolving state.

Some schemes require an authorization parameter when borrowing. This parameter
is referred to as `borrow_witness` and its meaning depends on the selected
scheme.

| Scheme | Meaning of `borrow_guard` | `borrow_witness` | Successor `borrow_guard` |
| --- | --- | --- | --- |
| `disabled` | Unused | Borrowing is rejected | Not applicable |
| `amount-threshold` | First eight bytes contain the threshold | Empty | Unchanged |
| `schnorr-signature` | Dedicated 32-byte public key | 65-byte Schnorr transaction signature | Unchanged |
| `hash-chain` | Current 32-byte hash | Its 32-byte preimage | Revealed preimage |

Every borrowed receive preserves `borrow_scheme`. A normal owner-authorized
transfer may replace both `borrow_scheme` and `borrow_guard`.

### Disabled

The `disabled` scheme prevents the UTXO from being borrowed and does not use
`borrow_guard`. Receiving tokens therefore requires a separate output.

### Amount threshold

The `amount-threshold` scheme permits borrowing without sender-specific
authorization. The first eight bytes of `borrow_guard` contain the threshold
that the token increase must strictly exceed; the remaining bytes are unused.
This makes dust-level token spam ineffective, although sufficiently large
transfers can still change the outpoint. No borrow witness is required.

### Schnorr signature

The `schnorr-signature` scheme stores a dedicated borrow public key in
`borrow_guard`. A sender with access to the corresponding borrow signing key may
authorize repeated borrows. This supports controlled reuse by recurring trusted
senders without granting permission to spend the recipient's tokens or reduce
the UTXO's KAS value. Other senders cannot make the locally recorded outpoint
stale through Borrowed Receive.

### Hash chain

The `hash-chain` scheme uses `borrow_guard` as the current commitment to an
OTP-like sequence of one-time borrow authorizations. A wallet can preallocate a
finite number of authorizations in advance. Each borrowed receive consumes one
authorization and advances `borrow_guard`, without requiring a new signature
from the recipient.

The idea originates in Rivest and Shamir's [PayWord and MicroMint: Two Simple
Micropayment Schemes](https://people.csail.mit.edu/rivest/pubs/RS96a.pdf).

To prepare a chain for `n` borrows, the wallet chooses a random value `x_0` and
computes:

```text
x_1 = Hash(x_0)
x_2 = Hash(x_1)
...
x_n = Hash(x_(n-1))
```

The initial `borrow_guard` is `x_n`, which commits to the complete sequence. A
borrow against guard `x_i` reveals `x_(i-1)` and proves the authorization by
requiring `Hash(x_(i-1)) == x_i`. The revealed value becomes the successor's
`borrow_guard`, ready for the next borrow.

The authorizations are revealed in reverse order and cannot be reused. The
wallet may provide them to an authorized sender in advance, allowing up to `n`
ordered borrows while the wallet remains offline. The chain is exhausted after
`x_0` is revealed.
