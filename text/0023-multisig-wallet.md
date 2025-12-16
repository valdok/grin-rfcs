
- Title: multisig-wallet
- Authors: [valdok](https://github.com/valdok)
- Start date: Dec 16, 2025

---

## Summary
Support an M/N multisig wallet.

## Motivation
The idea is to allow a wallet to be co-owned by several users, whereas a quorum of N/M is both required and sufficient to build a valid transaction (i.e. send or receive funds).


## Reference-level explanation

The core idea is based on Shamir's secret sharing (SSS) scheme, which allows the reconstruction of a secret by a quorum of M/N. And due to the homomorphic nature of ECC, the secret will not be reconstructed in a plain way. It will only be used to compute the shared pubkey, and to calculate the signatures required to build transactions (such as Schnorr's signature, and UTXO rangeproof)

Normal wallet has a master secret (usually derived from the seed phrase), which is then used to derive various keys, including the blinding factors for coins (like BIP44).

The multisig wallet requires 2 changes:

1. It will have 2 keys. One will be used in a deterministic key derivation (like HKDF / BIP44). The other, called a **co-factor**, will be constant, and act as a multiplier. It will be split among N co-owners via SSS scheme.
2. Key derivation can proceed accoring to BIP44, but the key path must include the coin value, not just the coin number.
More about this later, but in principle this should guarantee that each collectively built transaction spends and produces coins exactly as agreed, without amount being replaced/tampered.


### Key concepts

Here we'll define key terms and concept
 - **Secret key** - EC (elliptic curve) scalar
 - **Pubkey** - EC point (group element).
 - **Actor** - a user that co-owns a wallet.
 - **Moniker** - a unique user name/alias, that the actor chooses for identification (such as `Alice`, `Bob`, and etc.)

 **Message authentication**
 
 The protocol doesn't always require a secret P2P communication between individual actors, most messages are shared between all the actors and considered non-secret. However some messages must be protected against tampering. Hence we assume that each message broadcasted by an actor
 is signed by its private key, and then verified by others vs its pubkey

**Scalar derivation**
We'll need a function to generate a scalar from arbitrary arguments. Basically it's implemented via a hash function (such as SHA-256). However additional steps are required to make sure the result is a valid EC scalar (clamping).
We'll call this function $Hash_s(msg)$

**Moniker -> scalar function**
While actors use monikers for identification, the SSS operates on arguments in terms of EC scalars. Each actor will be assigned a scalar $x_i = Hash_s(moniker_i)$. There should be no duplicates. In a highly unlikely case of collision, the actor will have to choose another moniker.

**Mutual antisymmetric secret**

For each pair of actors $i \neq j$ we'll define this function:

$\delta(i,j,context) = Hash_s( DH(i,j) | context) \cdot (x_i - x_j)$

whereas $DH(i,j)$ stands for Diffie-Hellman shared secret, $context$ is arbitrary message, and $x_i$ is derived from the actor's moniker.
Not that this function is antisymmetric: $\delta(i,j) + \delta(j,i) = 0$


### Wallet initialization/recovery

In this process, M initial users initialize/restore the wallet

(1) Each actor generates (or restores) its master key $sk_i$ (perhaps derived from its private seed phrase), and then broadcasts a message with those fields:
- Pubkey: $P_i = G \cdot sk_i$
- Moniker

This message **must** be signed by the actor pubkey. Not only to protect against tampering, but also to prevent a rogue-key attack.

(2) After actors exhange this information, everyone verifies there're indeed M different actors, and there're no duplicates. Then the following SSS polynomial is defined:

$$
sk(x) = \sum_{i=1}^{M} sk_i \cdot \prod_{i \neq j} \frac{x - x_j}{x_i - x_j}
$$

Or, in terms of pubkeys:

$$
P(x) = \sum_{i=1}^{M} P_i \cdot \prod_{j \neq i} \frac{x - x_j}{x_i - x_j}
$$

The secret co-factor is defined as: $sk_{cf} = sk(0)$

The co-factor image is defined as this: $P_{cf} = P(0)$

**Note**: The $P_{cf}$ is computed by every actor individually, and is known among actors. In contrast the $sk_{cf}$ is never computed in a plain form.

(3) Finally each actors initalizes its multisig wallet. The master secret used in BIP44/HKDF is initialized from $(P_{cf} | M)$. Each actor can individually calculate any pubkey:

$$
Pubkey(path) = BIP44(path) \cdot P_{cf}
$$

### Adding an actor

A quorum of M current actors can add an additional actor. For this the following steps are performed.

(1) New actor picks and shares a unique moniker. Unlike initial actors, that can generate/restore an arbitrary secret key, new actor's key is uniquely defined by its moniker.

(2) Each actor computes ands sends **privately** the following key share:

$$
sk_{share_i} = sk_{i} \cdot \prod_{j \neq i} \frac{x_{new} - x_j}{x_i - x_j} + \sum_{j \neq i} \delta(i,j, context_{msg})
$$

whereas $context_{msg}$ includes all the parameters of the ritual: $P_{cf}$, $x_{new}$, and the selected quorum of M existing actors. This is to make sure $\delta(i,j, context_{msg})$ is unique in each ritual

The new actor then performs the summation of all the shares:

$$
sk_{new} = \sum_{i} sk_{share_i}
$$

Finally it verifies the correctness of its secret key, by checking if its appropriate pubkey sits on the same SSS polynome.

$$
G \cdot sk_{new} = P_{x_{new}}
$$

### Building a transaction

