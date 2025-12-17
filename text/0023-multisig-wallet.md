
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

Normal wallet has a master secret (usually derived from the seed phrase), which is then used to derive various keys, including the blinding factors for coins (like \vf).

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

Note that during this summation, all the $\delta(i,j)$ terms are cancelled-out. They are needed to blind each actor's secret key, but don't affect the sum of all the shares.

Finally the new actor verifies the correctness of its secret key, by checking if its appropriate pubkey sits on the same SSS polynome.

$$
G \cdot sk_{new} = P_{x_{new}} = P(x_{new})
$$


### Building a transaction

MW transaction consists of inputs, outputs at least one kernel. In order to build a valid transaction, a subset of M/N actors are selected. Each actor gets the transaction parameters:
- list of input coins (coin number + value)
- list of output coins (coin number + value)
- any additional metadata (current blockchain height, fee, the identity of the tx peer, memo, etc.)

Each actor receives and realizes the tx parameters. In particular, the tx balance (outputs minus inputs) is calculated. And then it decides if it wishes to take a part. Then the selected actors perform an MPC to build the transaction.

Inputs are represented by Pedersen commitments (or, alternatively, Switch commitments). Since they are EC points, the knowledge of $sk_{cf}$ is not necessary, they can be calculated by each actor from $P_{cf}$ alone.

$$
C(number, value) = BIP44(CoinPath(number, value)) \cdot P_{cf} + H \cdot value
$$

Now we'll define how output TXOs and the transaction kernel are created and signed

#### UTXO

The UTXO is represented by its commitment and the Rangeproof (a.k.a. bulletproof). The commitment is computed exactly as for inputs, by each actor individually. So, only the rangeproof requires MPC.

During the rangeproof creation, various pseudo-random EC scalars are generated. There're 2 sources of the randomness:
1. Randomness to obscure the UTXO value, also used to "color" it, make it possible to recognize by the owner, and recover the parameters (coin number and the value).
2. Randomness to protect the UTXO blinding factor.

In standard wallet (not multisig) it's possible to use a single source of randomness for both purposes. It also means that UTXO recognition is in fact its full reverse-engineering.

In multisigned wallet it's essential to use 2 distinct sources of randomness. The (1) is derived from the $P_{cf}$ (which is known to all the actors) and the UTXO commitment. The (2) is derived by each actor individually to obscure its key share.
Moreover, since we're tralking about multisig, each actor will rely on non-deterministic random.

The rangeproof protocol between (P)rover and (V)erifier is defined like this:
- P: Commitment, A, S (commitments)
- V: y, z (challenges)
- P: T1, T2 (commitments)
- V: x (challenges)
- P: $Tau_X$ (scalar)
- (continued)

The $Tau_X$ is a linear combination of the UTXO blinding factor, and 2 nonces, whereas EC points T1 and T2 are commitments to those nonces. So, the modified algorithm to create the rangeproof looks like this:

- Each actor proceeds according to the scheme, up to where T1,T2 should be revealed.
- Each actor generates 2 nonces, using true random (i.e. non-deterministic), uses them to calculate its share of T1,T2, and reveals its shares.
- Once M actors reveal their shares, they're aggregated to compute the final T1,T2.
- Each validator derives the next nonce, and uses it to compute and reveal its share of $Tau_X$
- Once M actors reveal their shares, they're aggregated to compute the  $Tau_X$
  - Note: it's possible to verify the correctness of each partial contribution to $Tau_X$
- All the consequent steps are performed individually by each actor (no more MPC is required)


#### Transaction Kernel

In order to create and sign the transaction kernel, the MPC is required as well. Even in normal transactions, where both the sender and the receiver are standard wallets, the MPC is used to create and sign the transaction kernel.

Here we'll generalize this to the case where either side of the transaction (either the sender or the receiver or both) is a multisig wallet.

The difference of the inputs-outputs blinding factors contributes to the transaction excess, and should be compensated by the transaction kernel. Each actor calculates its share of this excess,
this is its share of the secret key that's used to sign the kernel.

An important addition to the classical MW is the so-called transaction **offset**. The total transaction blinding factor excess is split into 2 parts. One is compensated by the transaction kernel, and the other part is revealed in a plain form (EC scalar). This makes it infeasible for the attacker to reverse engineer the coinjoined transactions. To support this, each actor splits its key share into 2 parts as well. 

Then the principle is similar to that of UTXO. During the first round each actor creates a non-deterministic nonce, reveals its image, the image of its share to the kernel excess, and the offset. Then, once all the nonces and kernel commitment shares are known and aggregated (from both sides of the transaction), the kernel is fully built. During the second round, each actor derives the signature challenge, and reveals its share of the blinded secret key.

#### E2E flow

Here we'll consider an example where Alice sends funds to Bob, both are in fact multisig wallets

- Alice side
  - A quorum of $M_A$ actors that co-own the Alice wallet decides to send funds to Bob. A list of input coins is selected, fee is decided, the coin number for the exchange coin is choosen.
  - The information is distributed among the actors.
  - They generate appropriate nonces (2 for output UTXO, one for tx kernel)
  - Round 1: they reveal the images: T1,T2 for the UTXO, kernel Commitment and the noce image.
  - The semi-build kernel is sent to the Bob, among with the general transaction info
- Bob side
  - A quorum of $M_B$ Bob's actors decides to accept the funds. They choose the coin number for the output UTXO
  - Each actor generates the appropriate nonces (2 for output UTXO, and 1 for kernel).
  - Round 1: they reveal the images: T1,T2 for the UTXO, kernel Commitment and the noce image.
  - Round 2: each actor receives the aggregates for the UTXO and the kernel. And reveals its partial signatures, and its share to the offset
  - The aggregated kernel with the signed Bob's UTXO is sent to Alice
- Alice side
  - Round 2: each actor receives the aggregates for the UTXO and the kernel. And reveals its partial signatures, and its share to the offset
- The transaction is aggregated, and can be sent to the network

As can be seen from the above, the roles of Bob and Alice actors are symmetric. Both participate in 2 rounds of MPC.

## Security considerations

### Adding the new actor, and the role of $\delta(i,j)$

During the new actor addition, each current actor reveals its secret key in a plain form to the new actor. It's multiplied by the SSS coefficient, but this coefficient is widely known. So without the addition of $\delta(i,j)$ terms each actor secret key would be trivial to compute.

This is why $\delta(i,j)$ terms are essential, they perform the perfect blinding of the actor secret key. Moreover, even if several current actors collude with the new actor, still its key is perfectly blinded as long as there is at least a single honest actor that doesn't disclose its $\delta(i,j)$ term.

The only situation where the actor key can be compromised is wherer all M-1 other actors collude with the new one. But this essentially means that they togetther form a quorum of M malicious actors. And obviously a quorum of M actors can calculate and compromise any key.

### UTXO and kernel signing, why rely on non-deterministic random

It's generally considered better to rely on deterministic nonce generation scheme, such as RFC-6979. However such schemes can't be used as-is in multisig rituals. There're advanced schemes, such as those used in MuSig-DN, that rely on ZKP to verify that each actor generated its nonce deterministically.

For the sake of simplicity, we'll stick to random (non-deterministic) nonces. In the future it'll be possible to upgrade the protocol, and use the PRF (pseudo-random function) together with Bulletproof ZKP to generate and verify the correctness of the nonces.
In either case, the general flow remains the same.
And as long as the main principle holds: each nonce is used to only answer one challenge - the secret keys are safe.

### Rogue key attack
The only situation where such an attack is possible is during the wallet initialization by the quorum of M inital actors. If not mitigated, an actor can essentially cancel the keys of other actors, and gain an exclusive access.
As we mentioned, this is mitigated by the fact that all messages sent by the actors are supposed to be signed by them.

### Wagner attack

