# Mina Verifier Principles

This document explains how the Mina verifier in this repo decides whether to accept a **candidate chain** on-chain. It focuses on the roles of slot, epoch, epoch lock_checkpoint, economic/cryptographic finality, short-range/long-range forks, consensus rules, and density. The bridge context matters here: **we only verify whether a candidate tip is more secure than the currently bridged tip (stored on MinaStateSettlementContract)**.

---

## 1. Overall

In this [repo](https://github.com/yetanotherco/aligned_layer/blob/cf7cd54265f0ec39d158f00c4ca400ffaa9887d4/operator/mina/lib/src/lib.rs#L38), the Mina verifier flow can be summarized in three steps:

1. **Data integrity checks**  
   The public inputs must be self-consistent (the candidate chain is contiguous, hashes match, and the bridge tip hash matches the current state).
2. **Consensus safety comparison**  
   Apply Mina’s fork-choice rule to compare `candidate_tip` vs `bridge_tip` and decide which is more secure.
3. **SNARK proof verification**  
   Verify the SNARK proof for the candidate tip (Pickles recursion ensures its ancestors are valid too).

> Think of it as:  
> **first ensure no forged data → then pick the better chain by consensus → finally confirm correctness with SNARKs.**

---

## 2. Time and Cycles: Slots and Epochs

### slot (time tick)
- Mina divides time into fixed-length **slots**.  
- On mainnet, each slot is about **3 minutes**.  
- Each slot has a chance to produce a block (block producers are chosen via VRF).

### epoch (a larger cycle made of slots)
- An **epoch** is a fixed number of slots (given by the consensus constant `slots_per_epoch`; on mainnet it is **7140 slots/epoch**, about 14.9 days).  
- Epochs are the unit for stake/producer configuration: the producer set for one epoch depends on the **previous epoch’s** stake distribution.

### lock_checkpoint (the “anchor” within an epoch)
Mina **locks** the producer set for the next epoch during the **last 1/3** of the current epoch (i.e., almost the 10th day in about 14.9 days).  
In plain terms: **during the final third, the “producer list” is fixed**, and the next epoch uses that list.

In our fork-choice logic, `lock_checkpoint` is used to determine whether two chains are **short-range forks** (i.e., derived from the same producer set).

### The 2/3 vs 1/3 boundary

- **First 2/3**: the staker list is still “rolling.”  
- **Last 1/3**: the staker list is locked and written to `next_epoch_data.lock_checkpoint`.  
- **Next epoch**: blocks align to this locked list via `staking_epoch_data.lock_checkpoint`.  

If these locked lists match, it’s short-range; otherwise, it’s long-range.

### Visual sketch

```
One epoch (slots_per_epoch)
┌───────────────────────────────────────────────────────────────┐
│  First 2/3: list changes                Last 1/3: list locked │
└───────────────────────────────────────────────────────────────┘

epoch N (last 1/3 produces next_epoch_data.lock_checkpoint)
      → fixes producer set and staking rewards for epoch N+1
                  ↓
epoch N+1 (blocks align to staking_epoch_data.lock_checkpoint)
      → produce blocks, accumulate staking rewards
                  ↓
epoch N+2 (distributes rewards from epoch N+1)
```

---

## 3. Block Validity & Economic finality

### Block Validity
- Provided by **SNARK proofs**.  
- If the candidate tip’s SNARK proof verifies, the tip state is cryptographically valid and its ancestors are included via recursion.
- In the Mina State Verifier within alignedlayer, this is done by `verify_block(...)`, with Pickles recursion.
- Note: the SNARK circuit mainly covers **block validity** (e.g., protocol-state and VRF/slot-related constraints). **Fork-choice rules** (short/long-range, density) are *not* in the circuit; they are applied at the node/verifier level.

### Economic finality
- Provided by **consensus rules**.  
- A block can be cryptographically valid but still not the *most secure* chain (e.g., a parallel fork).
- So we need consensus rules to decide which chain is harder to reorganize.

---

## 4. Short-Range vs Long-Range Forks

Mina splits forks into two classes, which determines how we compare chains:

### Short-range fork
Two tips are short-range if:

1. **Same epoch**, and
2. `staking_epoch_data.lock_checkpoint` is identical  
   (meaning both chains use the same producer set)

Or:

1. **Adjacent epochs**, and
2. the newer epoch’s `staking_epoch_data.lock_checkpoint`
   equals the older epoch’s `next_epoch_data.lock_checkpoint`

> This means the chains share the same producer set and only diverge briefly.

### Long-range fork
If the above does not hold, it’s long-range.  
This can imply different stake distributions, so we require stricter rules (density).

### Honest node decision (fork point before lock_checkpoint)
If the fork point happens **before** the epoch’s `lock_checkpoint`, the producer sets may differ.  
Mina treats this as a **long-range fork**.

Honest nodes then decide:

- **Prefer higher density** (denser chains are more secure);
- If densities tie, fall back to **length** (longer is better);
- If length ties, use **VRF output/state hash** as a tiebreak.

This keeps honest nodes from being fooled by a sparse chain created after an early fork.

TIPS: In reality, Mina nodes mark `lock_checkpoint` as a checkpoint (Just like checkpoint in other POS chains), which means honest nodes **DIRECTLY reject** the fork chain whose fork point happens **before** the epoch’s `lock_checkpoint`.

---

## 5. What Is Density?

In long-range cases, Mina compares **density** to judge security.

### Intuition
Density measures:  
**how many valid blocks were produced in a recent time window.**

Denser chains mean higher participation and are harder to fake.

### Key data in code
The consensus state includes:
- `sub_window_densities`: block counts per sub-window.
- `min_window_density`: the recorded minimum window density.

In long-range forks, we compute a **relative minimum window density** to compare chains.

---

## 6. Mina’s Consensus Rules (as implemented here)

The fork-choice logic in this repo boils down to:

1. **Check short-range?**  
   - Yes: use **length rule** (longer wins)
   - No: use **density rule** (denser wins because it implies more honest participation and more valid blocks, making sparse attacker chains less viable)
2. If density ties, fall back to length.
3. If length ties, use VRF output and state hash as tiebreakers.

### Length rule (`select_longer_chain`)
Compare by `blockchain_length`:
- Longer wins
- If equal, compare the hash of `last_vrf_output`
- If still equal, compare the hash of `consensus_state`

### Density rule (`relative_min_window_density`)
Compute the candidate chain’s relative min window density and compare with the bridge tip:
- Higher density → more secure
- Equal density → fall back to length
- Lower density → not secure

---

## 7. Put It Back Into the Verifier

The verifier’s core question is:

> **“Is the candidate tip more secure than the current bridged tip?”**

So the flow is:

1. **Ensure the candidate chain is contiguous and hashes match**  
2. **Compare candidate vs bridge tip using consensus rules**  
   - short-range → length-first  
   - long-range → density-first  
3. **Verify the candidate tip’s SNARK proof**

Only when **both economic finality and cryptographic finality** pass do we accept the candidate tip.

---

## 8. Summary

One-line memory aid:

> **Short fork: length first; long fork: density first.  
> If density ties, compare length;  
> if length ties, compare VRF;  
> and finally SNARK proves the state is correct.**


