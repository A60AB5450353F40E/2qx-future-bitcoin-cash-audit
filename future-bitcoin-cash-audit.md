# Introduction

This is an audit report on "Future Bitcoin Cash" system of Bitcoin Cash (BCH) smart contracts, made at request of 2qx.

```
Future Bitcoin Cash Audit

2024-04-26  2qx to BCA

This is a memorandum of understanding between 2qx, ("Client") and bitcoincashautist (a.k.a. BCA, "the Auditor") for a review and written audit report of "Future Bitcoin Cash" contracts.

The Client proposes the audit will proceed in two phases. 

First, and initial assessment will be given of potential flaws or mistakes in the workflow or logic of the contract system. Second, once the Client has revised the contracts with improvements and corrections, the Auditor may prepare a written report on the security of the contracts.

The scope of the report should address near and long term security concerns, but may be limited to the on-chain operation of the permanent contracts. The report may omit any consideration of one-time use bootstrapping contracts (i.e. the Battery). The report should address post-quantum and quantum-resistant concerns of the double-sha256 script locking mechanisms on the long term viability of the system. The report should note any potential for DDoS or other network attack vectors. 

The Auditor's report must be submitted in markdown format to be hosted by the Client in the final app. 

The Auditor's fee will be conveyed in Future Bitcoin fungible tokens. The fee will be conveyed as 2 FBCH UTXOs from each of the premiere vaults of the five timescale tranches, a total of 10 FBCH. To prevent the Client from the unsavory misfortune of holding a BCH-pegged liability, the Auditor will custody the 10 BCH necessary for creation of his FBCH tokens between the completion of the audit and the completion of the app. Once the Auditor retrieves his 2 FBCH x 5 from the premiere vaults, those FBCH tokens shall be considered by all parties to be free and unencumbered from any obligations or earmarks, i.e. that Auditor has been fully paid and may degen his Futures at will.

This is a non-binding memorandum of understanding without contingency.
```

Signed: `H1ErlyrktMt2a5tRzWHfjQkbpNtOCtH/QWcKaR3bRcCkRLmQDuJ3ufDDMe2/83bclcZmyIcHeDk7v4Uku3vhZpw=`

# Overview

Future Bitcoin Cash (FBCH) fungible tokens will essentially be instances of [wrapped BCH tokens](https://wrapped.cash/) with additional clause that they can't be unwrapped until the timelock expires.

The contracts system will consist of 4 contracts: Battery, Gantry, Vault, and Coupon.

From the point-of-view of someone getting paid in FBCH, his only concern is to audit the Vault contract and its genesis transaction, because it is the Vault that holds the backing BCH and underwrites the emitted FBCH.

Coupon is a simple one-time covenant that can be used with some Vault instance in order to get a discount when minting FBCH.

Battery and Gantry contracts are constructor contracts, contracts whose purpose is to create new instances of other contracts or tokens, according to specification.
The Battery will spawn a sequence of Gantries, and each Gantry will spawn a sequence of Vaults.

This will make it easier for any holder to audit FBCH genesis, as we only need to audit the contract system once, and from then on anyone interested in soundness of his FBCH can simply check that it has a Battery TXO of the correct category as ancestor, instead of having to analyze each Vault instance.

# Vault

We will start with the most important contract - one which underwrites every emitted unit of FBCH.

## Vault Summary

We find the contract to be doing what it intends to be doing:

>Vault - Store coins locked for tokens until maturation date.

Put simply, the contract can be used as follows:

- if timelock has expired then it can be used to unwrap/wrap BCH from/to token
- if timelock has not expired then it can be used only to wrap BCH to token

and we found no way to break the contract, assuming it was instantiated properly (entirety of supply given to the contract in token category genesis TX).

We have some suggestions for improvement and optimization, please see analysis below.

## Vault Contract Analysis

```
contract Vault(bytes4 locktime, bytes32 tokenCategory) {
```

There is no need to specify locktime as `bytes4`.
Natively the locktime is treated as a script number, so `int` type would be more appropriate, and use of `int(locktime)` could be avoided in the code below and save a byte (OP_BIN2NUM) in compiled bytecode.
As we will see later, the Vault bytecode is intended to be constructed by the Gantry contract, and use of fixed-size constructor parameter here could make that easier.
We're not convinced it is the best approach, because it is easy enough to generate a push sequence from a script number, especially if the number can be guaranteed to be greater than 16.

```
        bool tokensRedeemed = (
          tx.outputs[this.activeInputIndex].tokenAmount 
          - tx.inputs[this.activeInputIndex].tokenAmount
          ) > 0;
```

This is a good way to check that the user is attempting to unwrap.
Usage of `this.activeInputIndex` in this way works similar to SIGHASH_ONE.

```
        bool toVault = tx.outputs[this.activeInputIndex].lockingBytecode 
                 == new LockingBytecodeP2SH32(hash256(this.activeBytecode));
```

On its own, this code would force a wrongly instantiated thread (P2SH20) to mutate to P2SH32 on first usage, but it would also mean that the check becomes unnecessary from then on.
Note that, because of other code below, instantiating the contract as P2SH20 would break the unwrap path.

Generally, we recommend against enforcing P2SH32 by the contract's code.
Recursive covenant can be enforced more simply with:

```
        bool toVault = tx.outputs[this.activeInputIndex].lockingBytecode 
                 == tx.inputs[this.activeInputIndex].lockingBytecode;
```

This way, if properly instantiated as P2SH32 it will still remain P2SH32 for the lifetime of the contract thread.
Likewise, if wrongly instantiated as P2SH20 it will remain P2SH20.
Instead of encumbering even properly instantiated contracts with extra checks it should be responsibility of the app to instantiate the contract properly.

In any case, the `toVault` here is redundant, because code further below makes it so that, for properly instantiated contract (P2SH32), it's not possible for `toVault` to ever be false.

```
        if(toVault && tokensRedeemed){
            require(tx.time >= int(locktime));
            require(tx.outputs[this.activeInputIndex].tokenCategory == tokenCategory);
        } 
```

Good approach to enforce locktime only for the case of token redemption.
However, the token category check is redundant and can be removed, unless the intent is to have contract address be specifically usable only with the particular token category.
This check wouldn't prevent dusting the contract address with BCH or other token categories - it would just make all such UTXOs unspendable.

We recognize that hard-coding the categoryID would make it easier to use Electrum API to look up history of a particular token locked with the contract, and understand the choice due to current state of CashTokens infrastructure.

```
        require(
          tx.inputs[this.activeInputIndex].tokenCategory 
          == 
          tx.outputs[this.activeInputIndex].tokenCategory
          );

        require(
          tx.outputs[this.activeInputIndex].lockingBytecode 
          == 
          tx.inputs[this.activeInputIndex].lockingBytecode
          );

        require(
          tx.inputs[this.activeInputIndex].tokenAmount + 
          tx.inputs[this.activeInputIndex].value 
          == 
          tx.outputs[this.activeInputIndex].tokenAmount + 
          tx.outputs[this.activeInputIndex].value
         );
```

Straight-forward way to enforce unwrapping/wrapping BCH from/to token, as already seen in [wrapped.cash](https://bitcoincashresearch.org/t/wbch-bch-wrapped-as-cash-token/1196).
This code makes some of the above code redundant, as already indicated above.

## Vault Contract Design Notes

As it is, the contract can be used to wrap BCH even after timelock expiry.
We suggest to consider adding a NFT state to the Vault contract so that it can't be used for wrapping after timelock expires.

As it is, the contract will always leave 1 UTXO, even if never used.
We suggest to consider adding a clean-up spending path, so the contract can be burned at end of its lifetime, if all tokens are returned after the timelock.

## Notes on Vault Edge Cases

The creator can instantiate the category with any number of UTXOs and split fungible token supply across them.

Unless the creator has access to hashrate (which would allow him to circumvent network relay rules), each UTXO will have to be created with a dust amount of BCH.

Someone with hashrate could then wrap some BCH by interacting with 1 of the available UTXOs, and then (if timelock has expired) use the obtained wrapped BCH to deplete all other BCH UTXOs down to 0 BCH and make the threads unusable for regular users.
However, one thread is still guaranteed to remain usable, because entirety of token-backing BCH will end up with that 1 UTXO.

Similarly, after timelock expiry, anyone can spend all contract UTXOs together and move entirety of BCH (sans the dust) to just 1 UTXO, without needing to lock up any of his own BCH.
This would force wrapped token holders to use only that 1 thread until someone would wrap using one of the other threads.

Any regular user can obtain some wrapped BCH and then use it to create more UTXOs of the contract by funding them each with at least 1 token unit and dust amount of BCH.
There's no incentive to ever do this, because the user would only be losing money if he would do this.

If the category is instantiated with too low fungible token supply, then it would place a limit on how much BCH can be wrapped in total, and if multiple UTXOs exist it would allow for the possibility of reduction of number of threads.
This is trivial to prevent: at genesis, each UTXO should enough fungible tokens to be able to wrap whole BCH supply on its own.

At the limit, wrapping is multi-threaded until timelock, and unwrapping is single-threaded.
If single-threaded use is acceptable, then the contract needs no change.

If multi-threaded is desirable, then this contract code needs to be expanded, and that would come with caveats because the trade-off would be increasing contract size and with that the TX fee.

Adopting the above suggestion of prohibiting wraps after timelock expiry would somewhat improve the multi-threadedness because only previous token holders could deplete individual threads of BCH and only if they had previously obtained enough FBCH.

## Notes on Vault P2SH Collision

With hard-coded categoryID, the creator of category & contract would have enough degrees of freedom to attempt to brute-force categoryID (on creation) in order to find two redeem scripts that hash to the same P2SH20 address.
With P2SH32 that would be computationally infeasible therefore P2SH32 is recommended for the original contract code.

The original contract code is such that a P2SH20 variant unusable for users anyway, and creator would have nothing to gain from having an alternative, secret, redeem script available to him.

Users of the contract can't attempt a collision search anyway, because contract code enforces exact bytecode to be passed on.

If P2SH32 enforcement code would be removed, users should be aware of the need to verify that the contract was instantiated as P2SH32, as part of their due diligence on the app.

Users are expected to perform some due diligence on the contract in any dapp they use, anyway, so having the P2SH32 check in the contract code itself doesn't relieve them of their due diligence.

## Notes on Vault Quantum Security

The P2SH32 variant is quantum secure against quantum preimage search because it would still be a 2<sup>128</sup> problem, even for quantum computers.
It is still theoretically vulnerable to quantum collision search, as it would be a 2<sup>85</sup> problem for quantum computers.

Wihout some scientific breakthrough, a 2<sup>85</sup> problem would still be [impractical](https://crypto.stackexchange.com/questions/102574/could-grovers-algorithm-perform-a-search-in-n-2-for-a-match-in-a-particular-sub).

In any case, due to nature of the contract, collision search attempt is possible only before contract instantiation and therefore P2SH32 contracts created before availability of hypothetical quantum computers would remain safe to use even after availability of such computers.

## Notes on Vault Proper Category Initialization

The genesis transaction MUST NOT create any token UTXOs locked with any other bytecode than the Vault contract's P2SH32 address.

The genesis transaction SHOULD NOT create any NFTs.

The genesis transaction SHOULD create enough fungible token supply so that entirety of BCH supply can be wrapped.

## Vault Alternative Contract

Below we present contract code with the above suggestions implemented.

Because hardcoded categoryID is removed, this contract's bytecode pattern doesn't offer enough degrees of freedom for a P2SH20 collision search.
The locktime offers only 4 bits of freedom, and some values would make it unusable or unconvincing, further reducing practically usable degrees of freedom.

Permutations of lines of contract code (and resulting bytecode) would possibly allow a few more bits for the search, but it would be obvious to anyone that the below presented code was changed without a plausible reason.

For this reason, the below code is expected to be safe to use even as P2SH20.

```
contract Vault(int locktime, int maxSupply) {

    function TerminateOrSwap() {

        if (tx.outputs[this.activeInputIndex].lockingBytecode == 0x6a) {

            // If the vault is being terminated then
            // locktime must be expired...
            require(tx.time >= locktime);
            // ...and the exact amount of tokens already returned to
            // the Vault instance which is getting burned...
            require(tx.inputs[this.activeInputIndex].tokenAmount ==
                maxSupply);
            // ...and the remaining dust either burned or donated
            // to miners.
            require(tx.outputs.length == 1);

        } else {

            // If tokens are flowing back into this contract
            if(tx.outputs[this.activeInputIndex].tokenAmount >
                tx.inputs[this.activeInputIndex].tokenAmount
            ) {
                // enforce a BIP65 timelock 
                require(tx.time >= locktime);
            }

            // Pass on the token category & capability
            require(tx.inputs[this.activeInputIndex].tokenCategory ==
                tx.outputs[this.activeInputIndex].tokenCategory);
            // Pass on the contract
            require(tx.outputs[this.activeInputIndex].lockingBytecode == 
                tx.inputs[this.activeInputIndex].lockingBytecode);
            // Pass on bch+token balance
            require(tx.inputs[this.activeInputIndex].tokenAmount +
                tx.inputs[this.activeInputIndex].value == 
                tx.outputs[this.activeInputIndex].tokenAmount + 
                tx.outputs[this.activeInputIndex].value);
        }
    }
}
```

# Coupon

## Coupon Summary

Coupon is a simple one-time covenant that releases its funds only if some minimum amount of BCH is paid into a particular locking bytecode.

We found an error in the contract that would break it from performing the intended purpose.
With the error fixed, the contract can be considered fit for purpose of incentivizing interaction with Vault contracts.

## Coupon Contract Analysis

```
require(tx.outputs[0].value >= amount);
```

This doesn't check whether something was added to the Vault.
It is possible that the Vault already had enough BCH because another user previously interacted with it.
In this case the coupon would be free money, since user wouldn't have to add his own BCH to the Vault.

The above should be changed to `require(tx.outputs[0].value - tx.inputs[0].value >= amount);`.

```
require(tx.outputs[0].lockingBytecode == lock);
```

The covenant only requires a matching locking script, however it doesn't check for a particular instance of the Vault.
This allows for use of any Vault instance with matching parameters, or even for creation of a brand new Vault instance in the same TX that spends the coupon, or a broken (unspendable) instance.
Even so, the spender will still be required to lock his own BCH, which will remain locked until Vault's timelock expires.

We can prohibit Vault genesis and broken instances simply by requiring the locking bytecode on the input instead of on the output:

```
require(tx.inputs[0].lockingBytecode == lock);
```

When `lock` is targeting the Vault bytecode, the output locking bytecode will still be enforced by transience: the Coupon requires Vault be spent in index 0 input, and the Vault being spent in index 0 input will require that it is passed on to index 0 output.

Anyone can create an instance of Vault before hand and spend the Coupon with his instance.
If it is desired to target a particular instance, then the Coupon should require a particular categoryID, but that would increase the size of the Coupon contract.

Alternatively, the Vault could be redesigned so that each instance carries an NFT such that it can't be separated from the Vault bytecode, in which case the Coupon could target the NFT and the locking bytecode requirement could be dropped.

```
require(this.activeInputIndex+1 == tx.inputs.length);
```

Flexible way to restrict TX to just 1 Coupon. This allows the user to add any number of other inputs between index 0 and the last index.

## Coupon Alternative Contract

```
contract Coupon(

  // Minimum spent to claim the coupon.
  int amount,
  
  // Destination for the zeroth (constraining) output.
  bytes lock

){

  function apply() {
    
    // assure at the minium amount is sent as the first output
    require(tx.outputs[0].value - tx.inputs[0].value >= amount);;

    // Check that the Coupon is interacting with an existing Vault instance
    require(tx.inputs[0].lockingBytecode == lock);
  
    // The coupon must be spent as the last input, 
    //   therefore only coupon may be spent at a time.
    require(this.activeInputIndex+1 == tx.inputs.length);

  }

}
```

# Gantry

The Gantry ensures that Vault genesis transactions are properly initializing Vault token categories.
The benefit of having instances of Vault created by a Gantry is that any instance of Vault can be easily audited:

1. Look up the TX where TXID == vaultCategoryID, that is the Vault's "pre-genesis" TX.
2. Verify that the 1st output of the pre-genesis TX has the Gantry contract code.

This is proof enough that the Vault was created according to specification, because Gantry code is known, and by auditing that it properly enforces specification we can validate Vault genesis TX without actually seeing it.

This is convenient because then we don't need to use an indexer to look up the genesis TX, and so any FBCH can be audited simply by querying any node for the pre-genesis TX.

## Gantry Summary

We find the contract to be doing what it intends to be doing.

>Gantry - Create vault contracts with fungible tokens in a uniform way.

The contract is intended as a single thread recursive covenant that uses a mutable NFT's commitment to keep track of the next Vault instance's `timelock` parameter.
We found no way to break the contract, assuming it was instantiated properly (single mutable NFT created and given to the contract in token category genesis TX).

We have some suggestions for improvement and optimization, please see analysis below.

## Gantry Contract Analysis

```
require(tx.inputs[0].lockingBytecode == tx.outputs[0].lockingBytecode);
```

Using a hard-coded input & output index without additional guards is dangerous because on its own it would make it possible to spend 2 UTXOs with the same contract as both index 0 and index 1, possibly making them interact in unintended ways, or even allow a forged UTXO to be spent as index 0 and cheat the covenant.

As it is, the contract doesn't limit the number of inputs or explicitly enforce this contract to be executed as input 0, which makes it hard to reason about whether it can be broken or not.

Because the contract code commits to a hardcoded `categoryID` and requires it on both index 0 input and index 0 output, then if placed on an NFT of the same category the NFT can't escape from the covenant, assuming the category was instantiated properly (created exactly 1 mutable NFT output in genesis TX).
However, this kind of constraint leaves the contract open to some benign side-effects: the same contract can be placed on a pure BCH or another token category, in which case those UTXOs could be freely spent as inputs added to a TX spending the proper UTXO in index 0.

When UTXO has to pass its own state to a new output, it's recommended to use `[this.activeInputIndex]` index instead of `[0]`.
Both will result in same size of compiled bytecode, but the former makes the programmer's intent more obvious.

If the contract thread has to live in index 0 then we recommend explicitly requiring it:

```
require(this.activeInputIndex == 0);
```

to be sure that it is *this* UTXO that is satisfying other requirements on index 0 input, and not some other UTXO.

```
require(tx.inputs[0].tokenCategory.split(32)[0] == tokenCategory);
```

There is no need to hardcode `tokenCategory`, for same reasons as with the Vault contract.

If Gantry is correctly instantiated (genesis TX created single output of the category with mutable NFT and encumbered with the Gantry contract), then the next line:

```
require(tx.inputs[0].tokenCategory == tx.outputs[0].tokenCategory);
```

will suffice to enforce correctly passing on the token category.

```
require(tx.outputs[0].tokenAmount == 0);
```

If the Gantry category was instantiated without fungible tokens, then the above line is unnecessary.

```
int step = int(stepBytes);
int locktime = int(bytes4(tx.inputs[0].nftCommitment));
require(tx.outputs[0].nftCommitment == bytes4(locktime+step));
```

This enforces the mutable NFT to increment the stored state, but it forces the script number to be stored as `bytes4` fixed width LE integer in the NFT's commitment.
It could work the same if script number was stored directly, to avoid casting overheads.
However, benefit of storing a number as fixed width LE integer is that generic block explorers etc. are likely to parse and display the number correctly (as opposed to parsing script numbers, support for which is not widespread).
The NFT and contract code will be slightly bigger than the alternative but since there will only be 1 NFT of the category in existence this is a good trade-off.

```
        if((locktime/step)%10==0){ 
            require(tx.outputs.length == 1);
        } else {
```

Every 10th spend from this contract just increments the NFT's state without doing anything else.
The reason for this is to avoid overlap with other instances of this contract, as we will see when we get to analyzing the Battery contract.

```
            bytes theVault = 
                0x20 + tx.inputs[0].outpointTransactionHash +   // new fungible category
                0x04 + bytes4(tx.inputs[0].nftCommitment) +     // locktime
                vaultUnlockingBytecode;    
            bytes vaultLockingBytecode = 0xaa20 + hash256(theVault) + 0x87;
```

This generates the locking bytecode for the Vault instance created in this spend.

We already commented that Vault could remove the categoryID, and use native script numbers.
In that case, the above code could be reworked to:

```
            int lockTime = int(tx.inputs[0].nftCommitment)
            bytes theVault = 
                bytes(lockTime).length + bytes(lockTime) +     // push locktime
                vaultUnlockingBytecode;    
            bytes vaultLockingBytecode = 0xaa20 + hash256(theVault) + 0x87;
```

Assuming Gantry was instantiated correctly (with a 4-byte timestamp and initial commitment greater than 16), we can be sure that the generated push sequence will be correct.

```
            require(tx.outputs[1].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[1].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[1].tokenAmount == 300000000000000);     
```

The bytecode and category checks are sound.
It would be good to require a dust BCH amount as well, else someone with access to hashrate could temporarily block normal operation by depleting the outputs of dust.

This still allows for creation of immutable NFTs with commitment decided by the spender.
We can restrict it to empty commitment with `require(tx.outputs[1].nftCommitment == 0x);`.
However, this will still allow creation of empty immutable NFTs.
Those will be a benign side-effect, and as it is the contract can't prohibit them due to how CashTokens introspection opcodes were designed (more info [here](https://github.com/cashtokens/cashtokens/issues/29)).

```
            bytes announcement = new LockingBytecodeNullData([
                0x46424348,
                bytes(tx.inputs[0].nftCommitment.split(4)[0])
                ]);
            require(tx.outputs[8].lockingBytecode == announcement);
```

If instantiated properly then there is no need for `.split(4)[0]` when contract code already enforces the commitment to be `bytes4`.
Because we know that proper instances will have exact 4 bytes of commitment, then we can hardcode the OP_RETURN & push opcodes:

```
bytes announcement = 0x6a044642434804 + tx.inputs[this.activeInputIndex].nftCommitment;
```

which will compile to `<0x6a044642434804> OP_ACTIVEINPUTINDEX OP_UTXOCOMMITMENT OP_CAT`.

```
            // Is this check necessary if the op_return is used?
            require(tx.outputs[8].value == 0);
```

It is recommended, and adding `require(tx.outputs[8].tokenCategory == 0x);` would be good, too.

```
            require(int(tx.outputs.length) == 9);  
```

This ensures that tokens of the new category can't "leak" outside the Vault contract.
If number of generated outputs is to be changed then this must be updated as well.
Also, the `int()` cast is redundant and can be removed.

## Gantry Contract Design Notes

As it is, anyone can evolve the Gantry contract to create lots of instances Vault with timelocks far into the future.
If this is undesirable, then use of Gantry should be timelocked, too.

Similarly, if Gantry didn't issue Vaults for a while and last one created is in the past, then later it could be minting more instances of Vault with timelocks already expired, just to evolve until the instance that's actually desired.
If this is undesirable, then timeskip option could be added to the contract.

## Notes on Gantry Edge Cases

If the `step` parameter is negative, then contract could still be used to mint some proper instances of Vault, if the genesis output's NFT is initialized with some commitment that encodes a timestamp in the future.
However, the contract will become useless as soon as the last created Vault has `locktime` in the present, and later create broken instances if arrives to below 0.

If the `step` parameter is 0, then the contract will be useless because it can create unlimited instances of Vault with same locktimes.

If the commitment is bigger than 4 bytes, then the contract will be broken.

If the commitment is less than 4 bytes, the contract will work and only the genesis output will have a commitment less than 4 bytes, because first spend will enforce padding to 4 bytes.

If the `commitment` + `step` would become greater than 4 bytes, then the contract thread will halt / become unspendable.

## Notes on Gantry P2SH Collision and Quantum Security

Same comments as for Vault apply.

## Notes on Gantry Proper Category Initialization

The genesis transaction MUST create one NFT with mutable capability and locked with the Gantry contract's P2SH32 address.

The `step` parameter MUST be a positive, greater than 0 integer, and less than INT4_MAX value.

The `commitment` SHOULD be initialized as 0-padded LE 4-byte integer.

The `commitment` MUST be a small enough value to allow at least one spend from the contract.

The Gantry genesis transaction MUST NOT create multiple token UTXOs of the category.

The Gantry genesis transaction MUST NOT create fungible tokens.

## Gantry Alternative Contract

```
contract Gantry(
    int step, 
    bytes vaultUnlockingBytecode
) {
    function execute() {
        // Gantry covenant and the associated NFT baton must be spent as index 0
        // input and passed on to index 0 output, funded with some dust BCH in order
        // to avoid griefing by someone with access to hashrate
        require(this.activeInputIndex == 0);
        require(tx.inputs[this.activeInputIndex].lockingBytecode ==
            tx.outputs[this.activeInputIndex].lockingBytecode);
        require(tx.inputs[this.activeInputIndex].tokenCategory ==
            tx.outputs[this.activeInputIndex].tokenCategory);
        require(tx.outputs[this.activeInputIndex].value > 800);

        // Read a bytes4 LE commitment and convert to script number
        int locktime = int(tx.inputs[this.activeInputIndex].nftCommitment);

        // Allow time-skip in case Vault to be minted has already expired
        if (locktime <= tx.locktime) {
            locktime = (tx.locktime / step + 1) * step;
        }

        int nextLocktime = locktime + step;

        // Prevent the spender from rolling over from height-based timelock
        // to timestamp-based timelock.
        require(nextLocktime < 500000000);

        // Locktime stored in mutable NFT commitment MUST be incremented by <step>
        // and stored as bytes4 LE uint again.
        require(tx.outputs[this.activeInputIndex].nftCommitment ==
            bytes4(nextLocktime));

        // Every 10th step, skip creating Vault and just increment the commitment
        if((locktime / step) % 10 == 0) { 
            require(tx.outputs.length == 1);
        } else {
            // Construct redeem bytecode for the Vault instance being created
            bytes theVault = 
                bytes(bytes(locktime).length) + bytes(locktime) + // int locktime
                vaultUnlockingBytecode;
            // Construct P2SH32 locking bytecode from redeem bytecode
            bytes vaultLockingBytecode = 0xaa20 + hash256(theVault) + 0x87;

            // Verify creation of Vault genesis outputs

            // [1]
            require(tx.outputs[1].lockingBytecode == vaultLockingBytecode);
            require(tx.outputs[1].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[1].nftCommitment == 0x);
            require(tx.outputs[1].tokenAmount == 2100000000000000);
            require(tx.outputs[1].value > 800);

            // [2]
            require(tx.outputs[2].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[2].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[2].nftCommitment == 0x);
            require(tx.outputs[2].tokenAmount == 2100000000000000);     
            require(tx.outputs[2].value > 800);

            // [3]
            require(tx.outputs[3].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[3].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[3].nftCommitment == 0x);
            require(tx.outputs[3].tokenAmount == 2100000000000000);     
            require(tx.outputs[3].value > 800);

            // [4]
            require(tx.outputs[4].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[4].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[4].nftCommitment == 0x);
            require(tx.outputs[4].tokenAmount == 2100000000000000);     
            require(tx.outputs[4].value > 800);

            // [5]
            require(tx.outputs[5].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[5].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[5].nftCommitment == 0x);
            require(tx.outputs[5].tokenAmount == 2100000000000000);     
            require(tx.outputs[5].value > 800);

            // [6]
            require(tx.outputs[6].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[6].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[6].nftCommitment == 0x);
            require(tx.outputs[6].tokenAmount == 2100000000000000);     
            require(tx.outputs[6].value > 800);

            // [7]
            require(tx.outputs[7].lockingBytecode == vaultLockingBytecode);       
            require(tx.outputs[7].tokenCategory == tx.inputs[0].outpointTransactionHash);
            require(tx.outputs[7].nftCommitment == 0x);
            require(tx.outputs[7].tokenAmount == 2100000000000000);     
            require(tx.outputs[7].value > 800);

            // Tag this FT mint for indexers 
            //
            // 6a              OP_RETURN
            // 04 46 42 43 48  <'FBCH'>
            // 04 90 05 10 00  <locktime>
            bytes announcement = 
                0x6a044642434804 + 
                tx.inputs[this.activeInputIndex].nftCommitment;
            require(tx.outputs[8].lockingBytecode == announcement);
            require(tx.outputs[8].tokenCategory == 0x);
            require(tx.outputs[8].value == 0);

            // Ensure no other outputs can be created
            require(tx.outputs.length == 9);
        }           
    }
}
```

# Battery

## Battery Summary

Constructs a series of properly initialized Gantry instances, which will all have the same categoryID inherited from the Battery.

We found errors in the contract that would break it from performing the intended purpose.

## Battery Contract Analysis

```
require(tx.time >= int(startTime));
```

There is no need for this, this will just make it so that the contract system can't be deployed ahead of first Vault's expiration.

```
require(tx.inputs.length == 1);
```

There is no need for this if contract correctly verifies the outputs.

```
        // Get the commitment 
        bytes4 stepBytes = bytes4(tx.inputs[0].nftCommitment);
        int step = int(tx.inputs[0].nftCommitment);
```

No need to convert to `bytes4` if the Battery was instantiated properly.
It will already be 4 bytes, and we need to keep it as bytes because we will need it later when constructing the instance of Gantry.
Also, the `step` can be converted from `stepBytes` instead of using TX introspection again.
Suggested:

```
        // Get the commitment 
        bytes stepBytes = tx.inputs[0].nftCommitment;
        int step = int(stepBytes);
```

However, in the next lines:

```
        // Set the gantry commitment to a block on increment in the near future.
        bytes4 gantryStart = bytes4(startTime - (startTime % step) + step);
        require(tx.outputs[1].nftCommitment == gantryStart);
```

we do need to convert to `bytes4` because here we are preparing a script num value to be written into a new output so have to enforce desired format.

```
        // Get the redeem bytecode of the gantry instance
        bytes gantryRedeemBytecode = bytes(vaultUnlockingBytecode.length) + vaultUnlockingBytecode +
                                     0x04 + stepBytes                                +  // stepBytes
                                     0x20 + tx.inputs[0].tokenCategory.split(32)[0]  +  // This tokenCategory
                                     gantryUnlockingBytecode;    
```

Assumption of 0x04 as push opcode for `stepBytes` is safe if Battery was initialized properly, since the covenant will guarantee it will stay 4 bytes further on.

```
        require(
            // The second output is the gantry lockingBytecode
            0xaa20 + hash256(gantryRedeemBytecode) + 0x87
            == tx.outputs[1].lockingBytecode
        );
```

Place a P2SH32 Vault on the 1st output.

```
        bytes gantryCategory, bytes gantryCapability = tx.outputs[1].tokenCategory.split(32);

        // Assure the gantry baton is a non-minting NFT
        require(int(gantryCapability) == 0);
        
        // Assure the gantry category matches the battery category
        require(gantryCategory == tx.inputs[0].tokenCategory.split(32)[0]);
```

This would force the NFT to be immutable and break the Gantry because it needs mutable capability to function.
Fixed below, and suggested a simpler way to achieve it:

```
    bytes gantryCategory =
        tx.inputs[this.activeInputIndex].tokenCategory.split(32)[0] +
        0x01;
    require(tx.outputs[1].tokenCategory == gantryCategory);
```

The below:

```
        // Prevent all the value from being cleared off
        require((tx.inputs[0].value - (tx.outputs[0].value + tx.outputs[1].value)) < 1000);
```

only makes sure that a max. of 1000 sats can be extracted as fee or another output.
It doesn't prevent someone with hashrate from griefing the outputs by setting one of them to below dust value, or from giving most balance to the Gantry instead of back into Battery.
Also, since there's no constraint on the number of outputs, NFTs of the category can escape the contract system.

We should add:

```
require(tx.outputs.length == 2);
```

and rework the code that follows to:

```
        // Enforce exact dust amount on the Gantry so that remainder must go
        // back into Battery or pure BCH change at index 0.
        require(tx.outputs[1].value == 800);

        if(step > end) {
            // Calculate and enforce next baton's step,
            require(bytes4(step / 10) == tx.outputs[0].nftCommitment);
            // token category & capability (pass on minting NFT),
            require(tx.inputs[0].tokenCategory == tx.outputs[0].tokenCategory);
            // and contract code.
            require(tx.inputs[0].lockingBytecode == tx.outputs[0].lockingBytecode);
        } else {
            // Burn the minting baton while allowing any remaining BCH
            // to be extracted to output 0.
            require(tx.outputs[0].tokenCategory == 0x);
        }
```

## Battery Alternative Contract

```
contract Battery(

    // Correct contract initialization will have minting NFT's commitment
    // set to bytes4(step), which will be the step set for 1st minted gantry,
    // and will then get decremented for the next one until end step is reached.

    // The end is the lowest step to create a Gantry for.
    int endStep,

    // Base time from which to calculate each Gantry's starting point, e.g.:
    // --| baseTime
    //   |------------------------------------| gantry 0 start
    //   |------------| gantry 1 start
    //   |----| gantry 2 start
    int baseTime,

    // Redeem bytecode tail of the gantry contracts
    bytes gantryReedemBytecodeTail,

    // Redeem bytecode tail of the vault contracts
    bytes vaultReedemBytecodeTail,

) {

    function execute() {

        // Get the current step, we will mint a Gantry for this step
        bytes stepBytes = tx.inputs[this.activeInputIndex].nftCommitment;
        int step = int(stepBytes);

        // Set the gantry's starting time at correct offset from baseTime
        bytes4 gantryStart = bytes4(baseTime - (baseTime % step) + step);
        require(tx.outputs[1].nftCommitment == gantryStart);

        // Construct the full redeem bytecode for the Gantry instance
        bytes gantryRedeemBytecode =
            bytes(vaultReedemBytecodeTail.length) + vaultReedemBytecodeTail +
            0x04 + stepBytes +
            gantryReedemBytecodeTail;

        require(
            // The second output must have the P2SH32 of the gantry redeem bytecode
            0xaa20 + hash256(gantryRedeemBytecode) + 0x87
            == tx.outputs[1].lockingBytecode
        );

        // Ensure that Gantry inherits a mutable NFT so that it may update the
        // commitment as it mints its Vaults.
        bytes gantryCategory =
            tx.inputs[this.activeInputIndex].tokenCategory.split(32)[0] +
            0x01;
        require(tx.outputs[1].tokenCategory == gantryCategory);

        // Exactly 2 outputs, so token state or BCH can't leak out.
        require(tx.outputs.length == 2);

        // Enforce exact dust amount on the Gantry so that remainder must go
        // back into Battery or pure BCH change at index 0.
        require(tx.outputs[1].value == 800);

        // Fee allowance = 1000
        require(tx.outputs[0].value >
            tx.inputs[this.activeInputIndex].value -
            1800);

        if(step > endStep) {
            // Calculate and enforce next baton's step,
            require(tx.outputs[0].nftCommitment == bytes4(step / 10));
            // token category & capability (pass on minting NFT),
            require(tx.outputs[0].tokenCategory ==
                tx.inputs[this.activeInputIndex].tokenCategory);
            // and contract code.
            require(tx.outputs[0].lockingBytecode ==
                tx.inputs[this.activeInputIndex].lockingBytecode);
        } else {
            // Burn the minting baton while allowing any remaining BCH
            // to be extracted to output 0.
            require(tx.outputs[0].tokenCategory == 0x);

            // Note: output 1 still mints a Gantry in this same TX,
            // and it will be the last one to get minted.
        }
    }
}
```

# Final Review

After the above review cycle, 2qx has implemented most suggestions and produced a new set of contracts, all of them found here: 

- https://github.com/2qx/future-bitcoin-cash/tree/3fab223e6472a24fda8b8c4cebc120a9293d7102/contracts

The contract system is expected to perform as intended.

[**Vault**](https://github.com/2qx/future-bitcoin-cash/blob/3fab223e6472a24fda8b8c4cebc120a9293d7102/contracts/vault.v2.cash)

Contract has implemented most suggestions and is expected to perform as intended.

The termination path has not been implemented, meaning it will be the lightest possible contract, and the trade-off is that each thread will leave a persistent UTXO in chain state.

[**Coupon**](https://github.com/2qx/future-bitcoin-cash/blob/3fab223e6472a24fda8b8c4cebc120a9293d7102/contracts/coupon.v2.cash)

Contract has implemented most suggestions and is expected to perform as intended.

It targets just the bytecode (and not a particular category) so it can be used with any instance of a target Vault.

[**Gantry**](https://github.com/2qx/future-bitcoin-cash/blob/3fab223e6472a24fda8b8c4cebc120a9293d7102/contracts/gantry.v2.cash)

Contract has implemented most suggestions and is expected to perform as intended.

It doesn't implement time-skip, meaning the contract is simpler but it will have to be called in sequence even if some Vaults it spawns may never be used or have expired already.

[**Battery**](https://github.com/2qx/future-bitcoin-cash/blob/3fab223e6472a24fda8b8c4cebc120a9293d7102/contracts/battery.v2.cash)

Contract has implemented suggested [alternative contract](#battery-alternative-contract) verbatim and is expected to perform as intended.
