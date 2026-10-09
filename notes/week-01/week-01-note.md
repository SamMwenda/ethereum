# Week 01: Ethereum Fundamentals

**Legend**
- ✅ = you had this right
- ⚠️ = not explained well / needs fixing
- ➕ = expansion or a concept your notes missed
- 🎯 = how to improve your understanding

---

## 1. Blocks

### What you got right
- ✅ A block batches many transactions together.
- ✅ Each block stores its **parent hash**, so blocks are cryptographically chained.
- ✅ Batching exists so nodes don't have to synchronize on every single transaction.
- ✅ Blocks are proposed on a 12-second cadence.

### Expanded
- ➕ **Why batching matters:** reaching agreement among thousands of nodes is expensive. One round of agreement per block is far cheaper than one per transaction.
- ➕ **What a block header contains** (beyond the parent hash):
  - `stateRoot`: fingerprint of the entire world state after executing the block
  - `transactionsRoot`: Merkle root of the block's transactions
  - `receiptsRoot`: Merkle root of the transaction receipts
  - `number`, `timestamp`
  - `gasLimit`, `gasUsed`, `baseFeePerGas`
  - `prevRandao`: randomness from the beacon chain
  - `feeRecipient`: where the priority fees go
- ➕ **The block hash is the hash of the header only**, not of the whole block. Because the header commits to the transaction root, changing any transaction changes the hash.
- ➕ **Why the chain is tamper-evident:** altering an old block changes its hash, which breaks the parent hash stored in the next block, and so on up the chain. An attacker would have to redo everything after it.

### Slots, epochs and finality
- ⚠️ **Your note says "understand slots here" but never defines them.**
- **Slot** = a 12-second window in which exactly one validator is chosen as the proposer. The proposer is picked pseudo-randomly using RANDAO.
- **Epoch** = 32 slots (about 6.4 minutes).
- **Committees:** in each slot, a group of validators is assigned to **attest**, meaning they vote on what they believe is the head of the chain. Each validator attests once per epoch.
- ⚠️ **A slot is an opportunity, not a guarantee.** If the proposer is offline, the slot is empty (a *missed slot*) and no block appears. So "a block every 12 seconds" really means "a *chance* of a block every 12 seconds".
- **Finality** (Casper FFG):
  - A checkpoint is *justified* when at least 2/3 of total stake votes for it.
  - It is *finalized* when the next checkpoint is also justified.
  - This takes about 2 epochs, roughly 13 minutes.
  - Finalized blocks can't be reverted without at least 1/3 of all staked ETH being burned (slashed).
- ➕ **Two layers run side by side:**
  - The **execution layer** (Geth, Nethermind, Besu, Reth) runs the EVM and handles transactions.
  - The **consensus layer** (Prysm, Lighthouse, Teku, Nimbus, Lodestar) handles slots, attestations and fork choice.
  - They talk to each other through the Engine API. Every node runs both.

### Forks and the fork-choice rule
- ⚠️ **Wording fix:** "two identical blocks" can't exist at the same position, because identical blocks would have the same hash and *be* the same block. You mean **two competing blocks** (for example, two blocks with the same parent, or a proposer who equivocated).
- ⚠️ **Your notes never name the fork-choice rule. It is LMD-GHOST** (Latest Message Driven, Greedy Heaviest Observed SubTree).
  - Each validator's *latest* attestation counts as a vote for a branch.
  - The vote is weighted by the validator's **effective balance** (staked ETH), not by headcount.
  - Nodes follow the branch with the most attested weight.
  - Gasper = LMD-GHOST (liveness, picks the head) + Casper FFG (finality).
- ⚠️ **Your question: why stake-weighted, not validator-count-weighted?**
  - **Sybil resistance.** If votes were counted per validator, an attacker could split 32 ETH into many tiny "validators" for free and gain influence without risking more money.
  - Weighting by stake ties influence to **economic value at risk**, and that value can be slashed.
  - Pre-Pectra, effective balance was capped at 32 ETH, so "more stake" and "more validators" were nearly the same thing. That is why the difference rarely shows up in simple explanations.
  - Post-Pectra (EIP-7251), validators can have effective balance up to 2048 ETH, so the stake-vs-count distinction is real. This is worth a line in your notes.
- ➕ **Tie-breaking:** if two branches have exactly equal weight, the tie is broken deterministically (by block root) so all nodes converge.
- ➕ **Proposer boost:** a timely block gets a temporary extra weight, which makes it harder to reorg a block that arrived on time.
- ➕ **Reorgs:** when a node switches from one branch to a heavier one, the abandoned blocks are dropped and their transactions return to the mempool. Small reorgs (1 to 2 blocks) are normal. Finalized blocks never reorg.
- ➕ **PoW vs PoS:** the old "uncle" blocks no longer exist. Competing blocks are simply orphaned.

🎯 **To improve:** draw a diagram with a block A and two children B and B′, add attestations of different weights, and trace which branch wins. Then repeat with a "many tiny validators vs few large validators" scenario.

---

## 2. Gas

### What you got right
- ✅ Gas measures computational work on Ethereum.

### ⚠️ Needs much more depth (this is the thinnest part of your notes)
- ⚠️ **Refine the definition:** gas is the unit of *effort* for every operation, covering computation, storage, and data. It is **not** a price. Price is quoted separately in gwei per unit of gas.
- **Why gas exists:**
  - It stops infinite loops from freezing the network (the halting problem). Execution simply runs out of gas.
  - It makes spam and denial-of-service attacks costly.
  - It prices scarce block space fairly.
- **Examples of cost:** a plain ETH transfer costs 21,000 gas. Writing a new storage slot (`SSTORE`) is about 20,000 gas. Simple arithmetic is a few gas.
- **Units:** 1 ETH = 10⁹ gwei = 10¹⁸ wei.

### Transaction fee mechanics (EIP-1559, type-2 transactions)
- `gasLimit`: the maximum gas you allow the transaction to consume.
- `gasUsed`: what it actually consumed.
- `baseFeePerGas`: set by the protocol per block and **burned**.
- `maxPriorityFeePerGas`: your tip to the validator.
- `maxFeePerGas`: the most you'll pay in total per gas.
- **Fee paid** = `gasUsed × (baseFee + priorityFee)`. The effective tip is `min(maxPriorityFee, maxFee − baseFee)`.
- **Base fee adjustment:**
  - Each block has a gas *target* of 50% of the block gas limit.
  - If the previous block was fuller than the target, the base fee rises by up to 12.5%.
  - If it was emptier, the base fee falls by up to 12.5%.
- **Block gas limit:** a cap on total gas per block, in the tens of millions (check the current value on a gas tracker). Blocks are *elastic*: they can be up to 2× the target in busy periods.
- ⚠️ **Failed transactions still cost gas.** A revert or out-of-gas stops execution and undoes state changes, but the work was done, so you still pay for it. Unused gas is refunded.
- ➕ **Blob gas (EIP-4844):** blobs have a separate fee market, used mainly by rollups (L2s) to post data cheaply.

🎯 **To improve:** pick a real transaction on Etherscan and compute the fee by hand from gas used, base fee, and tip. Also note how much ETH was burned.

---

## 3. Core Ideas

### 3.1 Hash

#### What you got right
- ✅ Transactions have a hash, and pending transactions sit in the mempool.

#### ⚠️ Corrections and gaps
- ⚠️ A transaction isn't simply "given" a hash at creation. The **transaction hash is `keccak256` of the *signed*, serialized (RLP-encoded) transaction**. So it only exists after signing, and it changes if any field or the signature changes.
- ⚠️ **Hash in general isn't defined in your notes.** A cryptographic hash function is:
  - deterministic (same input, same output)
  - one-way (can't recover the input)
  - fixed-size (Keccak-256 gives 32 bytes = 64 hex characters)
  - sensitive to tiny changes (the avalanche effect)
  - collision-resistant
- ➕ **Where hashes are used on Ethereum:**
  - transaction IDs
  - block hashes (hash of the header)
  - Merkle Patricia Trie roots (state, transactions, receipts)
  - address derivation
  - function selectors (the first 4 bytes of `keccak256("transfer(address,uint256)")`)
  - event topics
  - storage slot calculation for mappings
- ➕ **Keccak-256 is not exactly NIST SHA3-256.** They differ in padding, so the outputs differ. Ethereum uses the original Keccak.
- ⚠️ **The mempool isn't one global list.** Each node keeps its own pool of pending transactions and gossips them to peers over the p2p network. Different nodes can see slightly different pools.
- ➕ **Block builders** pick and order transactions from public and private channels (MEV-Boost / proposer-builder separation). Some transactions never touch the public mempool.
- ➕ **Replacing a pending transaction:** resubmit with the **same nonce** and a sufficiently higher fee (roughly 10% or more). This produces a *new* hash. To cancel, send a 0 ETH transaction to yourself with the same nonce.

### 3.2 Public / private keys

#### What you got right
- ✅ A private key is 64 hex characters (256 bits / 32 bytes).
- ✅ Public key derives from private key, never the reverse.
- ✅ Address = last 20 bytes (40 hex chars, `0x` prefix) of `keccak256(publicKey)`.
- ✅ EOAs have keys; contract addresses come from `CREATE` / `CREATE2`.

#### Expanded
- ➕ **The curve:** keys use **secp256k1** elliptic-curve cryptography. The public key is the private key multiplied by a fixed point on the curve. Reversing that is the *elliptic curve discrete logarithm problem*, which is computationally infeasible.
- ➕ **Public key size:** 64 bytes (the x and y coordinates). The address hashes this 64-byte value, with no `0x04` prefix.
- ➕ **Checksum addresses (EIP-55):** the mixed upper/lower-case you see in addresses encodes a checksum, which catches typos.
- ➕ **Contract address formulas:**
  - `CREATE`: `keccak256(rlp([sender, nonce]))[12:]`. Depends on the creator's nonce.
  - `CREATE2`: `keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))[12:]`. Predictable *before* deployment.
- ⚠️ **Contracts have no private key.** They can't sign anything; they only act when called, directly or indirectly, from a transaction an EOA signed.
- ➕ **Account structure:** every account has 4 fields: `nonce`, `balance`, `storageRoot`, `codeHash`. EOAs have empty code.
- ⚠️ **Missing: seed phrases and HD wallets.**
  - A 12/24-word recovery phrase (BIP-39) generates many private keys (BIP-32/44).
  - The Ethereum derivation path is `m/44'/60'/0'/0/x`.
  - Whoever has the phrase or the private key controls the funds. Never share either.
- ⚠️ **The EOA/contract line is blurring.**
  - **ERC-4337** (account abstraction) lets smart contract wallets act like accounts.
  - **EIP-7702** (Pectra) lets an EOA temporarily delegate its behavior to contract code.
  - Worth a short "modern note" in your file.

### 3.3 Signature

#### What you got right
- ✅ Every transaction must be signed with the sender's private key.
- ✅ Blind signing was a real problem because calldata is opaque hex.
- ✅ Decoding calldata into human-readable text addresses it.

#### ⚠️ Gaps
- ⚠️ **Explain what a signature actually proves:** authenticity (the key holder approved it), integrity (the content wasn't altered), and non-repudiation. The private key never leaves the wallet; only the signature is broadcast.
- ➕ **ECDSA signature = (r, s, v).** The signature is computed over the hash of the unsigned transaction fields.
- ➕ **Why transactions have no "from" field:** the sender's public key (and so the address) is *recovered* from the signature and the signed hash. This is `ecrecover`. Great insight to keep.
- ➕ **Replay protection:** the `chainId` (EIP-155) in the signed data stops a transaction from being replayed on another chain. The nonce stops replay on the same chain.
- ⚠️ **Name the specifics of the blind-signing fix** (check what your course called it):
  - Structured "clear signing" metadata, such as **ERC-7730** descriptors, tells wallets how to render calldata for a given contract.
  - Wallets use the contract **ABI** (function selector + encoded arguments) to decode calldata.
  - **EIP-712** gives typed, readable *off-chain* messages (permits, logins, orders).
  - **EIP-191** (`personal_sign`) is for plain-text message signing.
  - Many wallets also **simulate** the transaction to show balance changes.
- 🎯 **To improve:** write the full path in one line: `private key → sign(hash(tx)) → (r,s,v) → broadcast → node recovers sender`.

### 3.4 Proof of Stake

#### What you got right
- ✅ Ethereum moved from PoW to PoS (The Merge, September 2022).
- ✅ It is about 99%+ more energy efficient (the usual quoted figure is ~99.95%).
- ✅ Validators stake 32 ETH for the right to propose blocks.
- ✅ Misbehaving validators are slashed; honest ones earn rewards.

#### ⚠️ Corrections and gaps
- ⚠️ **"Deterministic block addition every 12 seconds" is slightly off.** What's deterministic is the *slot schedule*. Blocks can still be missed. In PoW, block times were random (a probabilistic lottery); in PoS, the timing is a fixed rhythm.
- ⚠️ **Validators do more than propose.** Their main job is **attesting** (voting) every epoch. Proposing is only occasional.
- ➕ **How validators are chosen:** proposers and committees are selected pseudo-randomly using RANDAO, weighted by effective balance.
- ➕ **Rewards come from:** attestations (the bulk), proposing blocks, sync committee duty, and priority fees/MEV.
- ⚠️ **Distinguish three kinds of bad behavior:**
  - **Going offline** gives small penalties and, in a prolonged non-finality event, the *inactivity leak*. This is **not** slashing.
  - **Slashing** is for provably dangerous acts: proposing two different blocks in the same slot, or double/surround voting in attestations. It brings an immediate penalty, a correlated penalty (bigger if many validators are slashed together), and forced exit.
  - **The 1/3 rule:** attacking finality requires at least 1/3 of all stake, and those attackers get slashed.
- ➕ **Why PoS needs valuable ETH:** security comes from economic stake that can be destroyed. This is "skin in the game".
- ➕ **Entering and leaving:** you deposit via the deposit contract and enter an activation queue. Exits and withdrawals go through exit/withdrawal queues (withdrawals enabled since Shapella, 2023).
- ⚠️ **Fee-format link needs correcting:** the "priority fee split" comes from **EIP-1559** (London, August 2021), which was introduced *before* the Merge. It fits PoS well, but it isn't a PoS invention.
  - Base fee → burned.
  - Priority fee → block proposer.
- ➕ **Transaction types to know:** type 0 (legacy), 1 (access list), 2 (EIP-1559 dynamic fee), 3 (blob), 4 (EIP-7702 set-code).
- ➕ **MEV and proposer-builder separation:** specialized *builders* assemble blocks; proposers pick the most profitable via relays. Mention it briefly, since it explains why proposers earn more than just tips.

### 3.5 Nonce

#### What you got right
- ✅ For EOAs: number of transactions sent from that account.
- ✅ For contract accounts: number of contracts it has created.
- ✅ It ensures no transaction can happen twice.

#### Expanded
- ➕ **Ordering:** nonces must be used strictly in sequence. If you send nonce 5 before 4 has been mined, nonce 5 waits in the queue.
- ➕ **Stuck transactions:** a low-fee transaction at nonce N blocks all later ones. The fix is to replace it with the same nonce and a higher fee.
- ⚠️ **Sharpen the replay claim:**
  - The nonce prevents replay on the *same chain* (same sender, same nonce can only be included once).
  - **ChainId** prevents replay on *other chains*.
- ➕ **It feeds address generation:** with `CREATE`, the creator's nonce is part of the new contract address.
- ⚠️ **Don't confuse with the PoW block nonce:** the old mining "nonce" in block headers was the number miners varied to find a valid hash. It's obsolete in PoS (the field is zeroed). Different concept, same word.

---

## 4. Concepts missing from your Week 1 notes

- ➕ **Accounts and world state:** EOA vs contract accounts; account fields; the state trie (Merkle Patricia Trie) and how `stateRoot` fingerprints everything.
- ➕ **The EVM:** a stack-based virtual machine that executes bytecode; every opcode has a gas cost.
- ➕ **Anatomy of a transaction:** `to`, `value`, `data`, `nonce`, `gasLimit`, `maxFeePerGas`, `maxPriorityFeePerGas`, `chainId`, `accessList`, signature `(v, r, s)`.
- ➕ **Receipts and logs:** the outcome of execution (status, gas used, emitted events). This is how dapps learn what happened.
- ➕ **Nodes and clients:** full vs archive vs light nodes; execution + consensus client pairing; JSON-RPC as the interface wallets use.
- ➕ **Finality vs inclusion:** "included in a block" is not the same as "finalized". Know the difference.
- ➕ **Units:** wei, gwei, ETH.
- ➕ **L2s/rollups (one-line awareness):** they batch transactions off L1 and post data as blobs, which is why blob gas exists.

---

## 5. Transaction lifecycle (tie everything together)

- **1. Create:** build the transaction (to, value, data, nonce, fees, chainId).
- **2. Sign:** wallet signs with the private key → `(r, s, v)`; the transaction hash is now defined.
- **3. Broadcast:** send via JSON-RPC to a node, which gossips it to peers.
- **4. Mempool:** nodes validate it (signature, nonce, balance) and hold it as pending.
- **5. Selection:** a builder/proposer picks it, prioritizing higher tips.
- **6. Execution:** the EVM runs it, consuming gas; state changes are applied; a receipt is produced.
- **7. Proposal:** the slot's proposer publishes the block (parent hash included).
- **8. Attestation:** the committee votes; LMD-GHOST makes the block the head of the chain.
- **9. Finality:** after about 2 epochs (~13 minutes), the block is finalized.
- **10. Fees settle:** the base fee is burned; the tip goes to the proposer; unused gas is refunded.

---

## 6. Quick reference

- **Private key:** 32 bytes = 64 hex characters
- **Public key:** 64 bytes (secp256k1 x,y)
- **Address:** last 20 bytes of `keccak256(pubkey)` = 40 hex characters
- **Slot:** 12 seconds
- **Epoch:** 32 slots (~6.4 minutes)
- **Finality:** ~2 epochs (~12.8 minutes)
- **Transfer gas:** 21,000
- **Validator stake:** 32 ETH minimum (up to 2048 ETH effective balance post-Pectra)
- **Fork choice:** LMD-GHOST (stake-weighted); finality: Casper FFG

---

## 7. Self-test questions

- 1. Why does a block's parent hash make tampering with history detectable?
- 2. Is a slot the same thing as a block? Explain.
- 3. Two competing blocks at the same height: how does a node decide, and why stake rather than validator count?
- 4. Why does a failed transaction still cost gas?
- 5. What's the difference between the base fee and the priority fee, and where does each go?
- 6. How can a node know who sent a transaction when there's no "from" field?
- 7. What does the nonce protect against, and what does the chainId protect against?
- 8. What is the difference between being offline and being slashed?
- 9. Why can't a contract account sign a transaction, and how does EIP-7702 change this picture?
- 10. What exactly is hashed to produce a transaction hash, and why does replacing a pending transaction change its hash?

---

## 8. Corrections at a glance

- **"Two identical blocks":** → *two competing blocks* (same hash means the same block).
- **"Every 12 seconds, deterministic":** → slot timing is fixed; blocks can be missed.
- **"Transaction given a hash at creation":** → hash exists after signing (hash of the signed, serialized transaction).
- **"Gas = computational power":** → unit of *work* (compute + storage + data); price is gwei per gas.
- **"Priority fee is a PoS feature":** → it's EIP-1559, which predates the Merge.
- **"Nonce means no transaction happens twice":** → same-chain replay is prevented by the nonce; cross-chain replay is prevented by chainId.
- **"Validators propose blocks":** → they mostly attest; proposing is occasional.
