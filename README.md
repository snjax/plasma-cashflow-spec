# ZK-Accelerated Plasma cashflow

## Abstract 

Plasma cashflow brings the idea of plasma slice space, the critical problem of that design is that the size of exclusion proofs is growing excessively during the plasma lifetime. 


## Slices 

This proposal is based on the UTXO model where each unspent output defines ownership of a specific slice. Length of the slice is equal to a value of the asset that was deposit to plasma.

## Block structure 

Each block of plasma is Sum Merkle Tree root, where leaves are voids or transactions, affecting the interval. Also root interval is always equal all interval of plasma (10000000 at the example below).


![block_structure](https://raw.githubusercontent.com/snjax/drawio/master/plasma%20cashflow%20state.svg?sanitize=true)

There are 3 types of transactions: 1 to 1 trasnfer, 2 to 1 join, 2 to 2 double signed atomic swap. We pack them all into following struct

``` solidity
struct Transaction {
    Input[2] inputs,
    Output[2] outputs,
    uint64 maxBlockIndex,
    Signature[2] signatures
}
```

to simplify snarky verification of the transaction.


## Comission for transaction

We are planning to use fiat model for comission. So, any user can transfer some plasma coins to plasma operator. Then the operator increase user's gas balance. User can spend the gas for transactions according to the arrangement with the operator. If something goes wrong, the user can exit from the plasma.


## Defragmentation procedure

Cashflow spec plasmas have defragmentation problem, where different users have multiple ranges. 

![fragmentation](https://raw.githubusercontent.com/snjax/drawio/master/state%20fragmentation.svg?sanitize=true)

We can present bounds between states of users as something like state channels:

![fragmentation2](https://raw.githubusercontent.com/snjax/drawio/master/State%20fragmentation%202.svg?sanitize=true)

Here is solution taking $n$ transactions and up to $\log(n)$ blocks for the problem, based on atomic swaps, where n is number of "channels" between fragmented slices.

At the following example the defragmentation takes 2 blocks and 3 transactions.


![fragmentation2](https://raw.githubusercontent.com/snjax/drawio/master/State%20fragmentation%203.svg?sanitize=true)

Plasma operator can reward users participating in the defragmentation via gas.

## Exit game for plasma cashflow

### Force inclusion of the transaction

This idea is proposed in [Plasma Cashflow](https://hackmd.io/DgzmJIRjSzCYvl4lUjZXNQ?view#%F0%9F%9A%AA-Exit) spec. A transaction may be force included into a block by following game:

1. Somebody can present the unsigned transaction and the bond 
2. Anybody can commit signatures (with auto refund the gas)
3. Anybody can challenge the transaction by presenting spend of inputs (including withdrawals) or non-inclusion proof for inputs
4. If the transaction is challenged, the challenger takes the bond. Else if all signatures collected, the transaction is included into a special block with the same exit priority as the oldest input of the transaction. In both cases (signatures collected or not) the transaction is considered to be removed from all other blocks

![force_include](https://gist.githubusercontent.com/snjax/16308108653e02079636e30fb88cb3f4/raw/cc82ffe5a42ef2331a0c41e85cbf6576353c0215/force%2520include(1).svg?sanitize=true)

### Standard exit game

1) Somebody starts the exit process from segment $[x_1, x_2]$ and attach his output to this process and set the bond.
2) (1) can be challenged by non-inclusion proof or spend (including withdrawals) immediately and get the bond
3) Anybody can start a special challenge request.
4) If the game is challenged by challenge requests, the bond is split to all 

### Challange request

1) Anybody present output $\in [x_1, x_2]$ older than the output of tx exiting
2) Anybody can challenge him presenting the spend. The spend must not be newer than output exiting in standard exit game.
3) Anybody can replace the challenge presented an older challenge
4) Challenger (1) can continue challenge game selecting utxo $\in [x_1, x_2]$ from tx in remained challenge
5) Same as (2) or (3)
6) The first challenger in (2-3) gets the gas refund. The second challenger in (5) get the bond. Or challenge request is considered to be completed.

<a href="https://gist.githubusercontent.com/snjax/66147fced3bca090884e6e7c048bc679/raw/fdcf7355aa9b1eca8aacafbf7433e7360af67cd7/exit_game(2).svg?sanitize=true" target="_blank">

![exit game](https://raw.githubusercontent.com/snjax/drawio/master/exit_game.svg?sanitize=true)

</a>

<a href="https://raw.githubusercontent.com/snjax/drawio/master/exit_game.svg?sanitize=true" target="_blank">See the schema with better resolution.</a>

### Known issues

The operator can create global exit from withheld block. The users begin to challenge the operator, but some of challengers are malicious and will be challenged by the operator.

The honest user view a lot of challenges, but he do not know, is there at least one not malicious challenge. So, the user is forced to participate in challenge game and this behavior is like MoreVP's issue with small timeframe to exit.

### The solution

* Multiple challenge game rounds
    for this case user can deside to attach to diferent rounds with different probabilities (for example, 1%, 10%, 100%). Such approach limits maximum number of honest users affected by griefing.

* Radical increasing challenge bond cost and distribute the most part of the cost to users, who wins the challenge at the last round.


# Plasma Cashflow data model




### Merkle trees and proofs

### General data types

We use this types as strucures or RLP structures and compute hash of them as is (without merkelization or something like this).
#### TXInput

Here is input to a transaction. You may consider it as pointer to any output in the blockchain. Note: amount may be lesser than the output. It is valid, if one output has many inputs with different parts of the segment.

```solidity
struct TXInput
{
    bool isNull,
    uint256 owner, 
    uint64 blockIndex, 
    uint32 txIndex,
    uint256 txContentHash,
    uint8 outputIndex, 
    Segment amount
}
```
#### TXOutput

You may consider it as owned segment.

```solidity
struct TXOutput
{
    bool IsNull,
    uint256 owner, 
    Segment amount
}
```

#### Segment

Segment with begin and end.

```solidity
struct Segment
{
    uint128 begin,
    uint128 end
}
```

#### Signature

```solidity
struct Signature {
    bool isNull,
    uint8 v,
    uint256 r,
    uint256 s
}
```



### Complex data types





#### Transaction

Transaction may be passed into functions as following object:

``` solidity
struct Transaction {
    Input[2] inputs,
    Output[2] outputs,
    uint64 maxBlockIndex,
    Signature[2] signatures
}
```

Some of inputs, outputs or signatures may be NULL. 

##### Transaction hash computation

Arguments are limited by 2736030358979909402780800718157159386076813972158567259200215660948447373041 (it is about 250 bits). This is dot order for baby jubjub curve.



We linearize TransactionContent and compute the hash.
It is enough to store only TransactionContentHashes at leaves of SumMerkleTree, because signatures are cryptographically bounded to inputs of the transactions. If the operator do not provide signatures, blocks are considered to be withheld.





### Plasma state

#### Plasma chain


```solidity
hashmap(uint64 => uint256) sumMerkleRoot;
```

All blocks are transactional. For deposits and withdrawals we are going to use `txFix` data s


#### Exit state

```solidity
hashmap(uint256 => bool) exitStateHashmap;
```

The list of unchallanged and not finalized exits


#### Deposit state

```solidity
OrderedLinkedList deposit;
```

Here is list of deposited segments.



### General methods

```solidity
function deposit(OrderedLinkedListItem depositSlot) external payable returns(bool);
```

Transaction may be banned and transaction may be increased priority for txContentHash 

### Priority increasing game

#### PriorityState
``` Solidity
struct PriorityState {
    TransactionContent txContent, //unsigned transaction
    uint256 txContentHash, 
    Signature s, // at least one signature
    uint256 timestamp
}

//storage

mapping (uint256 => bool) priorityChallenges // mapping keccak256(priorityState) => bool of all active priority challenges

mapping (uint256 => uint64) txFix // priority fix for transacion. 2^64-1 for banned transactions, zero is default value

```



```Solidity
function priorityBegin(
    TransactionContent txContent,
    uint256txContentHash,
    Signature s
)

struct priorityChallengeHash(
    PriorityState state,
    Groth16Proof snarkProof // proof that hash is invalid
)

// spend of one input of state.txContent
// txContentHash, txBlockIndex is information abot spending tx
struct priorityChallengeSpend(
    PriorityState state,
    uint256[3] txContentHash, //ContentHash of 2 inputs and spending tx
    uint64[3] txBlockIndex, // BlockIndex of 2 inputs and spending tx
    Groth16Proof snarkProof // proof of inclusion tx into , spending part of state.point
) external returns (bool);


// accept signature differs from state.s
function prioritySignatureCollect(
    PriorityState state,
    Signature s
)

```

### Exit game

#### ExitState

Here is exit state. `TXInput` is not an input of any included transaction. You may consider it as pointer to any output or input of withdrawal transaction.

Hashing algorithm is standard (keccak256 of blob). It is enough

```
struct ExitState {
    TXInput point,
    uint256 timestamp
}
```

```Solidity
function withdrawalBegin(
    Input point 
) external payable returns (bool);

function withdrawalChallangeSpend(
    ExitState state, 
    uint256[3] txContentHash,
    uint64[3] txBlockIndex, 
    Groth16Proof snarkProof // proof of inclusion tx into , spending part of state.point
) external returns (bool);

function withdrawalChallangeExistance(
    ExitState state,
    Groth16Proof snarkProof // proof of exclusion state.point from state.point.blockIndex
) external returns (bool);

function withdrawalChallangeHash(
    ExitState state,
    Groth16Proof snarkProof // proof that txContextHash is wrong
) external returns (bool);

function withdrawalChallangeConcurrent(

) returns (bool);


// finalize. If transaction is banned by txContentHash, reject exit procedure
function withdrawalEnd(ExitState state)
```


## SNARKs

### zkSNARK proof

```solidity
struct Groth16Proof
{
    uint256 A;
    uint256[2] B;
    uint256 C;
}
```
We can pack G1 point into one uint256 to make the commitment shorter.
We do not need to calculate the snarks immidiately, we can use something like truebit instead.
