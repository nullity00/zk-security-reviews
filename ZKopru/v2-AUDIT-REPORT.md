* Original document: https://hackmd.io/Bq588JDoSSWFyiKmfUjbdA?view
* Writer: Igor Gulamov

# zkopru audit
$$
\DeclareMathOperator*{\amper}{\&}
$$

[TOC]

## Introduction

Igor Gulamov conducted the audit of zkopru smart contracts and circuits.

This review was performed by an independent reviewer under fixed rate.


## Scope

Smart contracts at https://github.com/zkopru-network/zkopru/tree/audit-v2/packages/contracts and circuits at https://github.com/zkopru-network/zkopru/tree/audit-v2/packages/circuits.


## Issues

We find 1 major and 1 critical issues. These issues are fixed.

We consider commit [7020a0136bf0f8fc4e521bc76615bdfa62691a7d](https://github.com/zkopru-network/zkopru/tree/7020a0136bf0f8fc4e521bc76615bdfa62691a7d) as a safe version from the informational security point of view.


### Critical

#### 1. [packages/contracts/contracts/consensus/BurnAuction.sol#L273](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/consensus/BurnAuction.sol#L273)

Anybody can claim the balance from the auction.

**Fixed at [bd4c08afaf4f3966e4d1b3a15f059176f003c885](https://github.com/zkopru-network/zkopru/commit/bd4c08afaf4f3966e4d1b3a15f059176f003c885)**

### Major

#### 1. [packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L334](https://github.com/zkopru-network/zkopru/blob/d2f1d4628148fa8211e445c028f2c04d9a99bd5b/packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L334)

Node $110$ is not empty at the example, but it will be computed as empty at $\text{level}=1$.

We propose replacing `emptyNode` with `lastNotEmptyNode`.

```solidity=304
uint256 lastNotEmptyNode = treeSize + leaves.length - 1;
```

```solidity=315
if (nodeIndex <= lastNotEmptyNode) {
```

```solidity=334
lastNotEmptyNode >>= 1;
```
**Fixed at [c3a447c5ba18ad06e94a05892bb3b5f829487a01](https://github.com/zkopru-network/zkopru/commit/c3a447c5ba18ad06e94a05892bb3b5f829487a01)**

### Warnings

#### 1. [packages/circuits/lib/ownership_proof.circom#L12-L13](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/ownership_proof.circom#L12-L13)

`EdDSAPoseidonVerifier` is assuming, that $A$ point is on the curve. Also, we recommend adding on curve checks for $R8$.

**Will not fix**
    
    Checks will be implemented on client side.


#### 2. [packages/circuits/lib/zk_transaction.circom#L214](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/zk_transaction.circom#L214) [packages/contracts/contracts/zkopru/controllers/UserInteractable.sol#L24](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/UserInteractable.sol#L24)

Limits are inconsistent on contract and circuit sides. We propose using the same limits on both sides.

**Fixed at [97591efae6312e575207e2919e66037f0c3e8491](https://github.com/zkopru-network/zkopru/pull/295/commits/97591efae6312e575207e2919e66037f0c3e8491)**


#### 3. [packages/contracts/contracts/zkopru/storage/Reader.sol#L129](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/storage/Reader.sol#L129)

Valid proofs could be easily forged for zero-valued VK. We propose adding a check, that the value is not default, or replacing default values with randomly generated values.

**Will not fix**

    We'll assume that the VKs will be stored correctly during its initialization process. 
    If it's not stored correctly, users should not use that contract.

#### 4. [packages/circuits/lib/if_else_then.circom](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/if_else_then.circom)

Unoptimized code. We propose use the following equality to reduce number of mul gates.

$$
\amper\limits_{i=1}^n x_i = \text{is_zero}\left(n - \sum\limits_{i=1}^n \text{int}(x_i)\right)
$$

**Will not fix**

#### 5. [packages/circuits/lib/erc20_sum.circom#L17-L26](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/erc20_sum.circom#L16-26)

Unoptimized code. We propose rewriting it as

```javascript=16
    component sum[n];
    var lc = 0;
    for(var i = 0; i < n; i++) {
        sum[i] = IfElseThen(1);
        sum[i].obj1[0] <== addr;
        sum[i].obj2[0] <== note_addr[i];
        sum[i].if_v <== note_amount[i];
        sum[i].else_v <== 0;
        lc += sum[i].out;
    }
    out <== lc;
```

**Will not fix**


#### 6. [packages/circuits/lib/inclusion_proof.circom#L47-L57](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/inclusion_proof.circom#L47-L57)

2nd `Mux1` component is unnecessary. We propose rewriting it as

```javascript=47
        left[level] = Mux1();
        left[level].c[0] <== nodes[level];
        left[level].c[1] <== siblings[level];
        left[level].s <== path_bits.out[level];

        branch_nodes[level].left <== left[level].out;
        branch_nodes[level].right <== nodes[level] + siblings[level] - branch_nodes[level].left;
```

**Will not fix**


#### 7. [packages/circuits/lib/zk_transaction.circom#L180-L202](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/zk_transaction.circom#L180-L202)

Suboptimal circuit. We propose rewriting it as

```javascript
typeof_new_note_is_zero[i] = IsZero();
typeof_new_note_is_zero[i].in <== typeof_new_note[i];
revealed_token_addr[i] <== new_note_token_addr[i] * (1-typeof_new_note_is_zero[i].out);
revealed_erc20_amount[i] <== new_note_erc20[i] * (1-typeof_new_note_is_zero[i].out);
revealed_erc721_id[i] <== new_note_erc721[i] * (1-typeof_new_note_is_zero[i].out);
```

**Will not fix**


#### 8. [packages/circuits/lib/zk_transaction.circom#L211](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/zk_transaction.circom#L211)

Unused variable `range_limit`.

**Will not fix**

#### 9. [packages/contracts/contracts/zkopru/libraries/SNARK.sol#L34-L44](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/SNARK.sol#L34-L44)

Unnecessary checks. In-field and on curve checks are applied inside precompiled contracts. `SNARK_SCALAR_FIELD` checks are enough for the Groth16 verifier.

**Will not fix**

#### 10. [packages/contracts/contracts/zkopru/libraries/SNARK.sol#L53](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/SNARK.sol#L53)

We propose initialize `vkX` with value from `vk.ic[0]`.

**Will not fix**

#### 11. [packages/contracts/contracts/zkopru/controllers/validators](https://github.com/zkopru-network/zkopru/tree/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/validators)

Only some of `Block` fields are used. We propose optimizing data loading to the memory. Loading the whole block into the memory is suboptimal.

**Will not fix**

### Comments

#### 1. Suboptimal $\mathcal{W}$ serialization

For example, $\mathcal{N}.\mathsf{v_{eth}}$ is limited by [`RANGE_LIMIT`](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/UserInteractable.sol#L24), so, it's enough to serialize it into 28 bytes instead of 32 bytes.

#### 2. Representing NFTs as partial cases at the circuit side could be unnecessarily 

We propose to flatten NFT address and id into one key and use it as ERC20 address with `totalSupply=1`.


#### 3. [packages/circuits/lib/zk_transaction.circom#L256-L290](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/circuits/lib/zk_transaction.circom#L256-L290)

Input and output sums check is suboptimal. We propose rewriting it as one array with positive inputs and negative outputs and check, that the total sum is zero for each asset type.

Also, you may skip ETH check, just check all sums of all notes (without taking into account the asset type).

#### 4. [packages/contracts/contracts/zkopru/storage/Storage.sol#L5](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/storage/Storage.sol#L5) 

[packages/contracts/contracts/zkopru/storage/Storage.sol#L9](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/storage/Storage.sol#L9) 

[packages/contracts/contracts/zkopru/storage/Reader.sol#L10](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/storage/Reader.sol#L10)

Unnecessary import.

#### 5. [packages/contracts/contracts/zkopru/libraries/SMT.sol](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/SMT.sol)

160 bits are enough for collision resistance. 

#### 6. [packages/contracts/contracts/zkopru/libraries/Deserializer.sol#140](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L140) 

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L203](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L203)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L254](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L254)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L331](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L331)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L356](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L356)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L383](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L383)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L512](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L512)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L565](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L565)

[packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L633](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/libraries/Deserializer.sol#L633)

`free_mem` pointer grows, but this memory will be never used in future. We propose fixing the memory leak.

#### 7. [packages/contracts/contracts/zkopru/controllers/validators/MigrationValidator.sol#L134](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/validators/MigrationValidator.sol#L134)

[packages/contracts/contracts/zkopru/controllers/validators/MigrationValidator.sol#L169](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/validators/MigrationValidator.sol#L169)


Usage of in-memory struct is unnecessary here. We recommend replacing it with `bytes32`.

#### 8. [packages/contracts/contracts/zkopru/controllers/validators/UtxoTreeValidator.sol?#L58-L61](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/validators/UtxoTreeValidator.sol?#L58-L61)

[packages/contracts/contracts/zkopru/controllers/validators/WithdrawalTreeValidator.sol#L51-L54](https://github.com/zkopru-network/zkopru/blob/31c457f2d3c6da2c73fd9c41e7f86466287bcdc9/packages/contracts/contracts/zkopru/controllers/validators/WithdrawalTreeValidator.sol#L51-L54)

[packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L184-L186](https://github.com/zkopru-network/zkopru/blob/d2f1d4628148fa8211e445c028f2c04d9a99bd5b/packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L184-L186)

We propose rewriting it as 

```solidity=
uint256 res = (a + b - 1) / b;
```

#### 9. [packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L260-L272](https://github.com/zkopru-network/zkopru/blob/d2f1d4628148fa8211e445c028f2c04d9a99bd5b/packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L260-L272)

[packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L117-L128](https://github.com/zkopru-network/zkopru/blob/d2f1d4628148fa8211e445c028f2c04d9a99bd5b/packages/contracts/contracts/zkopru/libraries/MerkleTree.sol#L117-L128)

We propose renaming `nextSiblings` and adding description, how it works on zero nodes.
