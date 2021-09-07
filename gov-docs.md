# GOVERNANCE DOCS

This is a lightweight spec and usecases description of the working design for governances using the basic spec conventions & patterns that are being adopted on MLabs projects, namely:
- forwarding minting policy pattern
- state/permission/signed/distributed/record token patterns
- Pagination pattern
- distributed list & distributed map patterns

This document will define the working low-level details for a Governance/Proposal contract structure that can:
- fit all transactions within onchain transaction size and compute limits.
- fairly divide rewards collected over a given reward period (epoch) 
- provide accurate voting records for a given point in time which is verifyable onchain
- involve minimal action from a trusted/central authority
- effectively prevent gaming the system though infinite votes or overallocated rewards
- minimize transaction costs.
- be extended without code change to support an extensible set of governance actions

this is a concept description for a system which uses _record tokens_, minted/updated whenever a user deposits or withdraws to manage a transaction history.

## High-level Workflow

User Roles: Staker, Admin, Pollster

### Scenario 1 - infinite vote mitigation, vote delegation, lockless voting, and democratic procedure.

1) Staker A Deposits $GOV for the first time by calling the `GovernanceValidator` with redeemer `DepositAct`

  effects: 
  - a BalanceRecord is minted, reflecting the balance after this tx.
  - a UserStakeDetail is minted, which points at the BalanceRecord with a txOutRef

note: step 1 can be followed by any number of subsequent deposits/withdraws, minting a new BalanceRecord each time.

2) Staker B Deposits $GOV as well, creating a similar paper trail.


3) the Admin deploys a ProposalFactory

4)Staker calls the ProposalFactory Validator with redeemer `CreateProposalAct`

  effects:
  - new IndividualProposal Validator is initialized

5) Staker A attempts to vote, in order to do so the staker may need to call `IndividualProposal` with Redeemer `ProveNoDuplicatesAct`  n times where n scales logarithmically with the number of previously cast votes.

  constraints:
  - if Staker A has previously cast a vote, this will fail

  effects:
  - `ProofVoting` token is minted, Datum is updated each time this is called.

6) Staker A then provides the `ProofVoting` token to `IndividualProposal` with Redeemer `VoteWithProofAct`.

  effects:
  - `ProofVoting` is burned.
  - `VoteRecord` is minted  representing the user's vote onchain.


7) Staker B wishes to delegate voting rights to Staker A, Staker B calls `GovernanceValidator` with `DelegateVoteAct` redeemer.

  constraints:
  - Staker B must not have already delegated their votes

  effects:
  - a `DelegateVote` Token is minted, carrying datum to track this, and sent to the GovernanceValidator


8) Staker A wishes to use a delegated vote, Staker A must supply the `DelegateVote` token to `PreVoteAct`, instead of testing for Staker A's address, the Validator uses the address from the datum on the `DelegateVote` Token to check for duplicates, if a previous vote is found, the transaction fails. otherwise, Staker A can then vote, the `Vote {..} :: ProofVoting` Datum will point to the address of Staker B (the address from `DelegateVote` Datum)

  constraints: neither Staker B nore Staker A on behalf of staker B may have previously voted on this `IndividualProposal`

  effects:
  - a `ProofVoting` token is minted, with `Vote` data constructor for Datum, citing the address for Staker B

9) at the end of the voting period, the Pollster can show that the proposal was successful by calling `CountVoteAct` n times where n scales logarithmically with number of voters. each time providing the `BalanceRecord`s necessary to prove the balance of $GOV tokens at the time the voting period ended.
  
  effects:
  - the vote totals are updated to reflect the balance of each voter at the time the voting period ended.

10) following the review period, the Pollster/Admin can Execute the proposal

  constraints:
  - votes must be fully counted (this is verifiable within ProposalState)
  - Proposal success parameters must be met
  - proposal must not have been executed previously

### Scenario 2 - reward dispersal

1) Staker Deposits $GOV for the first time by calling the `GovernanceValidator` with redeemer `DepositAct`

  effects: 
  - a BalanceRecord is minted, reflecting the balance after this tx.
  - a UserStakeDetail is minted, which points at the BalanceRecord with a txOutRef

note: step 1 can be followed by any number of subsequent deposits/withdraws, minting a new BalanceRecord each time.

2) the Admin provides rewards by calling the `GovernanceValidator` with redeemer `ProvideRewardAct`

3) at least 5 days from the previous Epoch end time, the Administrator (or anyone) calls `GovernanceValidator` with redeemer `TriggerEpochAct` n times where n scales linearly with the number of staking users.

  effects:
  - a `LastEpochScriptState` is minted, then iterated over the `UserStakeDetail`s until complete, progressively summing the totals of each stake.
  - (at the end) - `GovernanceState.rewardsTotal` is set to zero, so the next epoch will include seperate funds.

4) Staker calls `GovernanceValidator` with `ClaimRewardAct`

  constraints: 
  - if the user has past unclaimed epochs other than the one being claimed (available from UserStakeDetail), then we must reject and those past epochs must be claimed _first_ (automatic from offchain code).
  - user must be able to prove stake balance through a combination of BalanceRecord and UserStakeDetails, transactions occuring ON the ending timstamp of the epoch are determined to be _after_ this epoch.

  effects:
  - the user receives a share of the rewards for the epoch proportional to the proven balance relative to the overall total of the stake for that epoch.

## One-shot Tokens

These are non-fungible tokens that should be parameterized only on  UTXO inputs, as is shown in the `Currency` use case example in the Plutus MonoRepo.

One-shot tokens within the system are:

### GovernanceState token 

  Token Type: State token for script

  purpose: allows us to manage stateful data relating to totals of important funds held by the Governance Validator

  carries datum: 
  ```
  GovernanceState { rewardsTotal: Value 
                  , lastEpoch :: Natural
                  , lastEpochEnded :: PosixTime
                  , totalStakers :: Natural
                  }
  ```
 
  initialized to 
  ```
  GovernanceState { rewardsTotal = Value.empty 
                  , lastEpoch = 0
                  , lastEpochEnded = currentTime - 3 days
                  , totalStakers = 0
                  }
  ```

  mint: can always mint.
  
  burn: can never be burned

  inputs:
- fee/collateral UTXO

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState (MINTED) -> Governance Validator Script
  
## Governance Minting Policy
parameters: 
```
GovStateMintingParams { governanceScriptAddress :: ValidatorHash
                      , governanceStateCurrencySymbol :: CurrencySymbol
                      , ownerAddress :: PubKeyHash
                      }
```

Purpose: The Governance Minting Policy mints various tokens used primarily by the Governance Validator.

the `governanceScriptAddress` and `governanceStateCurrencySymbol` parameters are used to securely identify the correct state tokens and script instance to prevent supplying structures in their place.

### UserStakeDetail token

  Token Type: State token for user

  purpose: allows us to manage stateful data relevant to a User's interaction with the Governance Validator, which is a stake pool structure (users deposit $GOV Tokens (minted externally) into the pool so that their votes can be counted and rewards can be accurately distributed).

  carries datum: 
  ```
  UserStakeDetail { userAddress :: Address
                  , amountStaked :: Natural
                  , lastReward :: PosixTime 
                  , lastBalanceRecord :: TxOutRef
                  , lastEpochClaimed :: Natural
                  , delegated :: Bool
                  , index :: Natural
                  }
  ```
  
  initialized to 
  ```
  UserStakeDetail { userAddress = DepositAct.address -- determined by validator
                  , amountStaked = DepositAct.amount -- determined by validator
                  , lastReward = currentTime
                  , lastBalanceRecord = firstBalanceRecord -- determined by validator
                  , lastEpochClaimed = GovernanceState.lastEpoch
                  , delegated = False
                  , index = GovernanceState.totalStakers -- 0 indexed
                  }
  ```

  mint & burn: must include the GovernanceState UTXO
  - cannot be burned

  inputs:
- fee/collateral UTXO
- GovernanceState UTXO (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState UTXO -> Governance Validator
- UserStakeDetail (MINTED) -> Governance Validator

### BalanceRecord Token

Token Type: a Record token which allows us to document the user's past balance updates such.

Purpose: Used to prove balances & vote weights for reward dispersals and proposal vote counts - these records pass references to form a linked-list/chain structure across UTXO's with the head/tip at the UserStakeDetail, and new insertions sit between the previous `UserStakeDetail.lastBalanceRecord`, and the `UserStakeDetail` itself.

carries Datum:
```
BalanceRecord
  { userAddress :: Address
  , balanceAfterTx :: Natural
  , lastBalanceRecord :: Maybe TxOutRef
  , originalTxOutRef :: Maybe TxOutRef
  , timestamp :: PosixTime
  }
```
initialized to:
```
BalanceRecord
  { userAdderss = DepositAct.address / WithdrawAct.address
  , balanceAfterTx = balanceAfterTx
  , lastBalanceRecord = lastBalanceRecordTxOutRef
  , originalTxOutRef = Nothing -- we can't assign the original UTXO here until the next time it is consumed.
  , timestamp = currentTime
  }
  -- determined by validator
  ```
  
minting:
- must include GovernanceState

cannot be burned

inputs:
- fee/collateral UTXO
- GovernanceState UTXO (from Governance Validator)
- (OPTIONAL) UserStakeDetail UTXO (from Governance Validator)
- (OPTIONAL) Previous BalanceRecord (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState UTXO -> Governance Validator
- UserStakeDetail (MINTED?) -> Governance Validator
- (OPTIONAL) Previous BalanceRecord -> GovernanceValidator (unchanged)
- BalanceRecord (MINTED) -> Governance Validator

### DelegateVote Token
Purpose: To provide a permission token allowing one explicit user to vote on behalf of another

Token Type: Permission token from one staker to another, residing at the GovernanceValidator's address.

carries datum:
```
DelegateVote:
  { from :: Address
  , to :: Address
  }
```
initialized to:
```
DelegateVote
  { from = DelegateVoteAct.from
  , to = DelegateVoteAct.to
  }
```

mint & burn:
- must have UserStakeDetail for `from` user provided to tx.

inputs:
- fee/collateral UTXO
- UserStakeDetail UTXO (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail -> Governance Validator (modify `UserStakeDetail.delegated` to True on mint, False on burn)
- DelegateVote (MINTED) -> GovernanceValidator

### LastEpochProof token

Token Type: a Permission token that witnesses a progressive fold over `BalanceRecord`s and `UserStakeDetail`s to find the total stake amount at the end of the reward period.

Purpose: this allows us to confirm that each user's stake is accounted for when totalling stakes for reward dispersals without double-inclusion, or leaving any out, and without having GovernanceState in contention for common operations.

carries Datum:
```
LastEpochProof
  { countedIndexRangeStart :: Natural
  , countedIndexRangeEnd :: Natural
  , runningTotalStake :: Value
  , epochEndedTime :: PosixTime 
  }
```
Initialized to:
```
LastEpochProof
  { countedIndexRangeStart = 0 -- or an optimum starting position
  , countedIndexRangeEnd = 0 -- or an optimum starting position
  , runningTotalStake = -- index 0's proven balance in this case.
  , epochEndedTime = currentTime
  }
```

mint & burn:
- the UserStakeDetail.index must match the value used as the starting value for `countedIndexRangeStart` and unless a set of records is provided, also `countedIndexRangeEnd`
- if a set is provided, they must fill out a numerical order and the transaction must fit on chain.
- each set of UserStakeDetail, Balance record must confirm the user's address and the BalanceRecords must refer to to the correct `BalanceRecord.timestamp`. the user's Balance is the `BalanceRecord.balanceAfterTx` of the BalnceRecord with the _earlier_ timestamp, so long as the range of the two timestamps includes the currentTime


inputs:
- fee/collateral UTXO
- one or more sets of:
  - UserStakeDetail UTXO (from Governance Validator)
  - 1-2 BalanceRecord UTXO (from Governance Validator) (in most cases this will just be the one BqalanceRecord especially if the fold is just starting)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- all sets of 
  - UserStakeDetail -> Governance Validator
  - 1-2 BalanceRecord UTXO -> GovernanceValidator (update `originalTxOutRef` if it was previously a `Nothing`)
- LastEpochProof (MINTED) -> Governance Validator


### LastEpochScriptState token (1 per epoch)
  
  Token Type: State token for script, only needed on specific actions. 
  
  purpose: This datum is needed as part of the reward distribution procedure, to record the exact state of the rewards and stakepool sizes so that they might be used to calculate reward dividends for each staker of $GOV at the end of a given reward period/epoch.
  
  similar to BalanceRecords, these form a distributed linked-list system with the head at UserStakeDetail, where new records are inserted between the UserStakeDetail and the previous record.
  
  carries datum: 
  ```
  LastEpochScriptState 
                       { stakeTotal :: Natural
                       , rewardsTotal :: Value,
                       , endTime :: PosixTime
                       , epochNumber :: Natural
                       , usersCounted :: Natural
                       }
  ```
  
  initialized to 
  ```
  LastEpochScriptState { stakeTotal = 0
                       , rewardsTotal = 0
                       , endTime = currentTime
                       , epochNumber = lastEpochNumber + 1 (zero-indexed)
                       , usersCounted = 0
                       }
  ```
  
  mint & burn: 
  - must include the GovernanceState UTXO
  - must include the LastEpochProof UTXO
  - LastEpochProof contain an index range that goes from zero to `GovernanceState.totalStakers`
  
  inputs:
- fee/collateral UTXO
- GovernanceState UTXO (from Governance Validator)
- LastEpochProof UTXO (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState UTXO -> Governance Validator
- LastEpochScriptState (MINTED) -> Governance Validator
- LastEpochProof UTXO -> GovernanceValidator 

## Governance Validator
parameters: 
```
GovernanceValidatorParams { governanceStateCurrencySymbol :: CurrencySymbol,
                          , ownerAddress :: PubKeyHash
                          , govTokenAssetClass :: AssetClass
                          }
```

Datum: 
```
GovernanceState
```
defined as a State token datum under _One Shot Tokens_

Redeemer:
```
= DepositAct { amount :: Natural
             , address :: Address 
             }
| WithdrawAct { amount :: Natural
              , address :: Address 
              }
| DelegateVote 
  { from :: Address, to :: Address }
| UndelegateVote 
| EpochCountingAct
| ProvideRewardAct { value :: PlutusTx.Value }
| ClaimRewardAct { address :: Address 
                 , claimForEpoch
                 }
| TriggerEpochAct
```

Purpose:
The Governance Validator controls transactions relating to two main protocol features
- measuring Vote weight of $GOV holders
- distributing dividends and rewards to $GOV holders

The Governance Validator is meant to integrate with any number of `ProposalFactory` validators and their corresponding `IndividualProposal`s 

scope notes: 
- since the Governance Minting Policy is Parameterized on the address of the Governance Validator, the governance validator can compute the `govMintingPolicyCurrencySymbol`.
- whenever referring to UserStakeDetail, it must be verified that the contract always contains exactly 1 user stake detail token per user address.

### DepositAct

Purpose: The user adds $GOV to their staked totals, may mint UserStakeDetail, mints BalanceRecord.

the `DepositAct.address` field is purposefully available to be a wallet or script address as a user may want to delegate control over stake and over rewards to an external contract.

Validation rules:
- user must have provided 1 or more UTXO's containing a total of at least `DepositAct.amount` of $GOV (Deposit UTXOS)
- user may have included a `UserStakeDetail` from the Governance validator script matching their `DepositAct.address`.
- user may have included a previous BalanceRecord, which can be verified by the `UserStakeDetail.lastBalanceRecord` txOutRef.

  if not, then one should be Minted from `GovernanceMintingPolicy`, with the user's `address` as the `address` parameter, currentTime as `lastReward` parameter, and `Nothing` as the `lockedUntil` and `lockedBy` parameters, 
  the `GovernanceState` token is only required in this case

  in both cases, the resulting UserStakeDetail should be sent (back) to the Governance Validator script address, with the new lastBalanceRecord txOutRef tagged
In order to minimize transaction sizes, the $GOV stake pool UTXOs should consolidate with previous deposits from all users, rather than directly transfer).
- any Wallet can deposit $GOV for Any address onchain. there is no restriction.
- any pre-existing BalanceRecord UTXO supplied must either have a txOutRef OR `BalanceRecord.originalTxOutRef` field in datum matching the `UserStakeDetail.lastBalanceRecord` field, or fail 

*known issue: we don't explicitly prevent duplicate addresses here,  hypothetically if this became an issue we could provide a schema endpoint to perform a merger using existing redeemers*

inputs:
- fee/collateral UTXO (from USER)
- deposit UTXO (1 or more) (from USER)
- (Optional) - consolidated stake deposit UTXO (from Governance Validator Script)
- (Optional) GovernanceState token  (from Governance Validator Script)
- (optional UserStakeDetail Token) (from Governance Validator Script)
- (optional BalanceRecord Token) (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- deposit UTXO -> Governance Validator script (consolidated)
- (Optional) GovernanceState token -> Governance Validator script (increment `GovernanceState.totalStakers`)
- UserStakeDetail Token (MINTED?) -> Governance Validator script (increment `UserStakeDetail.amountStaked` by `DepositAct.amount` OR mint fresh with `DepositAct.amount` as the initial value (other values should be initialized to defaults, `UserStakeDetail.address` should match `DepositAct.address`, `UserStakeDetail.lastBalanceRecord should point to the utxo containing a newly minted BalanceRecord).
- (optional) BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- BalanceRecord Token (MINTED) -> Governance Validator (set `balanceAfterTx` field to the new balance the user holds in the stakepool following the tranaction)

### WithdrawAct

Purpose: The user removes $GOV from their staked totals , may burn UserStakeDetail.
Validation Rules:
- if UserStakeDetail for the user's address is not included in the tx, fail.
- if amount specified in WithdrawAct.amount is greater than the balance of the `UserStakeDetail.amount`, fail
- transaction must be signed by `UserStakeDetail.address` if it is a PubKeyHash, or include a utxo from the validator if it is a ValidatorHash.
- if `UserStakeDetail.lockedUntil` is a future PosixTime, fail.
- any pre-existing BalanceRecord UTXO supplied must either have a txOutRef OR `BalanceRecord.originalTxOutRef` field in datum matching the `UserStakeDetail.lastBalanceRecord` field, or fail 


inputs:
- fee/collateral UTXO (from USER)
- (optional) GovernanceState token  (from Governance Validator Script)
- UserStakeDetail token  (from Governance Validator Script)
- Staked $GOV UTXO (from Governance Validator Script)
- (optional BalanceRecord Token) (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail Token -> Governance Validator script (decrement `UserStakeDetail.amountStaked` by `WithdrawAct.amount`)
- Withdrawal $GOV UTXO (contains WithdrawAct.amount of GOV) -> User Wallet
- (Optional) Remainder UTXO (contains any $GOV from input which exceeds `WithdrawAct.amount` -> Governance Validator Script
- (optional) BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- BalanceRecord Token (MINTED) -> Governance Validator (set `balanceAfterTx` field to the new balance the user holds in the stakepool following the tranaction)

### DelegateVote
Purpose: To allow a user to vote on one's behalf.

Validation rules:
- user address matching `UserStakeDetail.userAddress` (`DelegateVoteAct.from`) must validate this transaction

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail token  (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail token -> GovernanceValidator script (set `UserStakeDetail.delegated` to True)
- DelegateVote token (MINTED) -> GovernanceValidator script

### UndelegateVote
Purpose: to prevent a user who previously had vote delegation powers from using them any further.

Validation rules:
- user address matching `UserStakeDetail.userAddress` (`DelegateVote.from`) must validate this transaction

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail token  (from Governance Validator Script)
- DelegateVote token (from GovernanceValidator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail token -> GovernanceValidator script (set `UserStakeDetail.delegated` to True)
- DelegateVote token (BURNED)

### ProvideRewardAct

Purpose: receive rewards distributed by scripts or individuals for $GOV stakers, queue these rewards for fair distribution at the end of the next reward period/epoch.

Validation Rules:
- user may include one or more UTXOs with rewards in collections of arbitrary native tokens, (reward UTXOs) these should total to match ProvideRewardsAct.value
- in a real implementation, we need to limit the tokens allowed for reward, otherwise this could be an attack vector.

inputs:
- fee/collateral UTXO (from USER)
- Reward UTXOs (from USER - contains arbitrary tokens)
- GovernanceState token (from Governance Validator Script)
- optional previous reward UTXO for this epoch (from Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script (increment `GovernanceState.rewardsTotal` by `ProvideRewardsAct.value`)
- reward UTXOs -> Governance Validator Script (consolidate to single utxo per epoch)

### ClaimRewardAct

Purpose: a User who staked $GOV during a previous reward period  claims their share of the rewards.

Validation rules
- `LastEpochScriptState.epochNumber` must be equal to (`UserStakeDetail.lastEpochClaimed` +1)
- `BalanceRecord.address` on both records must match `UserStakeDetail.address`
- `UserStakeDetail.address must validate this transaction`
- one BalanceRecord must hold a timestamp _prior_ to the `LastEpochScriptState.endTime`, the `BalanceRecord.balanceAfterTx`  field in this UTXO is used as the source of truth for the user's balance at the end of the epoch being claimed.
- one BalanceRecord must hold a timestamp _after OR equal to_ `LastEpochScriptState.endTime` _OR this second BalanceRecord can be excluded if and only if the other BalanceRecord has a txOutRef or a datum field `BalanceRecord.originalTxOutRef` matching the `UserStakeDetail.lastBalanceRecord`
- amount transferred to user is a value defined by: (`BalanceRecord.balanceAfterTx` / `LastEpochScriptState.stakeTotal`) * `LastEpochScriptState.rewardsTotal`

inputs:
- fee/collateral UTXO (from USER)
- reward UTXO (1 or more) (from Governance Validator Script)
- UserStakeDetail token (from governance Validator Script)
- BalanceRecord Token (from Governance Validator Script)
- (Optional) BalanceRecord Token (from Governance Validator Script )
- LastEpochScriptState (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- Reward UTXO remainder -> Governance Validator Script
- UserStakeDetail Token -> Governance Validator script (increment `UserStakeDetail.lastEpochClaimed`)
- BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- LastEpochScriptState Token -> Governance Validator

### EpochCountingAct

Purpose: the first step in distributing rewards, mints an optimum number of utxos containing `LastEpochScriptStateProof`

note: this will rely on a constant `epochLength :: DiffMilliSeconds`

Validation rules:
- `GovernanceState.lastEpochEnds` must be at least `epochLength` less than the `currentTime`.
- anyone can call this
- if this is not called with a `LastEpochProof` token + datum supplied, then one must be minted
- if this is called with a `LastEpochProof`, then we can only accept an ordered subset of `UserStakeDetail` whose `index` is immediately adjacent to the `LastEpochProof.countedIndexRangeStart` or `LastEpochProof.countedIndexRangeEnd`
- if two `LastEpochProof` are supplied with adjacent ranges, we can merge them.
- the UserStakeDetail.index must match the value used as the starting value for `countedIndexRangeStart` and unless a set of records is provided, also `countedIndexRangeEnd`
- if a set is provided, they must fill out a numerical order and the transaction must fit on chain.
- each set of UserStakeDetail, Balance record must confirm the user's address and the BalanceRecords must refer to to the correct `BalanceRecord.timestamp`. the user's Balance is the `BalanceRecord.balanceAfterTx` of the BalnceRecord with the _earlier_ timestamp, so long as the range of the two timestamps includes the currentTime

note: with this we should be able to scale this logarithmically.

inputs:
- fee/collateral UTXO remainder (from USER)
- GovernanceState token (from Governance Validator script)
- 1 or more: LastEpochProof UTXO (from Governance Validator Script)
- one or more sets of:
  - UserStakeDetail UTXO (from Governance Validator)
  - 1-2 BalanceRecord UTXO (from Governance Validator) (in most cases this will just be the one BalanceRecord especially if the fold is just starting)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script 
- LastEpochProof UTXO(s) (Burned?) ->  Governance Validator Script (modify range fields to include valid indexes from userStakeDetail, or from merging `LastEpochProof`
- all sets of 
  - UserStakeDetail -> Governance Validator
  - 1-2 BalanceRecord UTXO -> GovernanceValidator (update `originalTxOutRef` if it was previously a `Nothing`)

### TriggerEpochAct
Purpose: to allow a user who has totaled up stakes for a reward period to present a `LastEpochProof` and allow `LastEpochScriptState` to be minted, which in turn allows users to claim rewards from this period.

Validation rules:
- `GovernanceState.lastEpochEnds` must be at least `epochLength` less than the `currentTime`.
- anyone can call this
- `LastEpochProof` must carry a range from 0 to `GovernanceState.totalStakers`

inputs:
- fee/collateral UTXO remainder (from USER)
- GovernanceState token (from Governance Validator script)
- LastEpochProof UTXO (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script (set `GovernanceState.rewardsTotal` to `Value.empty`, set `GovernanceState.lastEpochEnds` to current time,  increment `GovernanceState.lastEpoch` )
- LastEpochProof UTXO (BURNED)
- LastEpochScriptState state token (MINTED)-> Governance Validator Script (set `LastEpochScriptState.stakeTotal` to `GovernanceState.stakeTotal`, and `LastEpochScriptState.rewardsTotal` to `GovernanceState.rewardsTotal`, `LastEpochScriptState.epochNumber` to `GovernanceState.lastEpoch` + 1)

## Proposal Minting Policy (example)
parameters:
```
ProposalMintingParams { proposalFactoryAddress :: Address 
                      , governanceMintingCurrencySymbol :: CurrencySymbol
                      }
```

Purpose: The Proposal Minting Policy manages state tokens relating to a hypothetical Proposal Factory and/or IndividualProposal script.

note: this will not actually be implemented, it's intended as a guide for those who might need to integrate with the Governance Validator.

### ProposalState Token
  Token Type: State token for IndividualProposal Validator script.
  
  Carries Datum:
  ```
  ProposalState
    { votingEnds :: PosixTime
    , totalVoteAddresses :: Natural
    , voteAddressesCounted :: Natural
    , totalVotesInFavor :: Natural
    , totalVotesAgainst :: Natural
    , reviewTimeEnds :: PosixTime
    , executed :: Bool
    } 
  ```
  initialized to:
```
  ProposalState
    { votingEnds = currentTime + votingPeriodLength
    , totalVoteAddresses = 0
    , voteAddressesCounted = 0
    , totalVotesInFavor = 0
    , totalVotesAgainst = 0
    , reviewTimeEnds = currentTime + (votingPeriodLength*2)
    , executed = False
    } 
  ```
  
  Minting & Burning:
- must include an arbitrary utxo from the `proposalFactoryAddress`.

inputs:
- fee/collateral UTXO remainder (from USER)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- ProposalState -> new Proposal address determined by script


### VoteProof Token

Token type: Permission token. Similar to `LastEpochProof`, it witnesses a fold over `VoteRecord`s 

Purpose: to allow us to prove onchain that the user has not yet voted, such that the user can then vote.

carries datum:
```
VoteProof
  { countedIndexRangeStart :: Natural
  , countedIndexRangeEnd :: Natural
  , addressToCompare :: Address
  }
```
initialized to
```
VoteProof
  { countedIndexRangeStart = 0
  , countedIndexRangeEnd = 0
  , addressToCompare = VoteProofAct.address
  }
```

minting & burning:
- must include a UTXO from the proposal address
- can always be burned

inputs:
- fee/collateral UTXO remainder (from USER)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- VoteProof -> Proposal address 


### VoteRecord Token
Token Type: Record token witnessing a vote that occured

Purpose: displays a user's vote, which is tabulated after a voting period ends

carries datum :
``` 
VoteRecord
  { address :: Address
  , inFavor :: Bool
  , index :: Natural
  }
```
initialized to:
```
  VoteRecord
    { address = VoteAct.Address
    , inFavor = VoteAct.inFavor
    , index = ProposalState.totalVoteAddresses 
    }
```

minting & Burning:
- must include a ProposalState token
- can never be burned

inputs:
- fee/collateral UTXO remainder (from USER)
- VoteProof (from IndividualProposal Validator)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- VoteProof (Burned)
- VoteRecord -> Proposal address 


## ProposalFactory (Example)
Parameters:
```
ProposalFactoryParams { govAddress :: Address
                      . governanceMintingCurrencySymbol :: CurrencySymbol
                      } 
```

Purpose:
A ProposalFactory will integrate with the Governance script directly, providing the structure and logic to make a governance decision on a particular kind of question.
Since this is an example, certain details that would otherwise be described on other parts of the spec are not available at this time. 

This is an example for any service  that may perform an integration with the Governance Validator script

A Proposal Factory should carry specific logic pretaining to success conditions, quorum, minimum proposal requirements, the data which needs approval, and the voting period time limits for a proposal type, as well as any effects necessary to implement the change described (such as updating a configuration value for another script).

Datum:
`()`

Redeemer:
```
CreateProposalAct
```

### CreateProposalAct
Purpose:
this would mint a new ProposalState token and send it to the precomputed address for the IndividualProposal Validator Script

Validation rules:
these may vary based on proposal logic,  an example might be that the user's $GOV UTXO's must total more than a constant Minimum Proposal Amount.

inputs:
- fee/collateral UTXO (from USER)
- $GOV token UTXO for user (from Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- $GOV token UTXO for user -> Governance Validator Script
- ProposalState token (MINTED)-> IndividualProposal Validator Script

## IndividualProposal (Example)
Parameters: 
```
data IndividualProposalParams a = IndividualProposalParams
  { proposalFactoryAddress :: Address
  , governanceMintingCurrencySymbol :: CurrencySymbol
  , proposalMintingCurrencySymbol :: CurrencySymbol
  , proposalSpecificData :: a
  }
```

Purpose:
This script manages the act of voting on a particular proposal, the precise data being voted on, and determines if the Proposal is successful.

Datum:
```
ProposalState 
```
defined at _Proposal Minting Policy_

Redeemer:
```
= VoteAct 
    { inFavor :: Bool
    }
| PreVoteAct
| CountVoteAct
| ExecuteAct

```

### PreVoteAct
Purpose: to prove, iteratively that a user has not voted earlier, which is required to vote.

Validation Rules
- if this transaction does not include `VoteProof`  as an input, then one must be minted.
- if this transaction contains 2 or more `VoteProofs` which have immediately adjacent ranges, we can burn all but one, and merge the ranges.
- the transaction may supply `VoteRecord`s in immediately adjacent ranges to the one in `VoteProof` when this happens the range can include them if an only if they do note hold `VoteProof.addressToCompare` in `VoteRecord.address`
- if a `DelegateVote` token is supplied and we are minting the VoteProof, initialize using the `DelegateVote.from` address as the `VoteProof.addressToCompare`, if and only if `DelegateVote.to` validates the transaction.

inputs:
- fee/collateral UTXO (from USER)
- (Optional, 1 or more) VoteProof (from IndividualProposal Validator)
- (Optional) DelegateVote Token (from Governance Validator)
- 1 or more VoteRecord (from IndividualProposal Validator)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- (Optional) DelegateVote Token -> Governance Validator
- VoteProof (MINTED?) (BURNED?) -> IndividualProposal Validator (set range to include `VoteRecord.index`
- 1 or more VoteRecord (from IndividualProposal Validator)

### VoteAct
Purpose: to record a user's vote on a particular proposed value.

Validation Rules
- `UserStakeDetail.address` must validate the transaction
- transaction must occur before `ProposalState.votingEnds`
- `UserStakeDetail.amountStaked` must be greater than 0
- must include a `VoteProof` whose index range covers `ProposalState.totalVoteAddresses`

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail for user (from Governance Validator script)
- ProposalState (from IndividualProposal)
- BalanceRecord (early) - (From Governance Validator script)
- (optional) BalanceRecord (late) - (From Governance Validator script)
- VoteProof UTXO (from Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail -> Governance validator Script (set `UserStakeDetail.lockedBy` to IndividualProposal Validator Address and `UserStakeDetail.lockedUntil` to the end of the voting time for the proposal OR the current value of `lockedUntil` whichever is greater/further in the future.
- ProposalState -> IndividualProposal (increment `ProposalState.totalVoteAddresses`)
- VoteProof UTXO (BURNED)
- VoteRecord (MINTED) -> IndividualProposal (set address = `UserStakeDetail.address`, set `inFavor` to `VoteAct.inFavor`)

### CountVote
Purpose: to allow a user to progressively include votes in proposal count, such that we can confirm all votes are counted and declare a proposal successful.

Validation rules:
- `ProposalState.votingEnds` must have passed
- the `VoteRecord` may not have `consumed` as `True`
- the late `BalanceRecord` must carry a `lastBalanceRecord` to the early `BalanceRecord` OR the early balance record is directly referenced by the `UserStakeDetail` and the late `BalanceRecord` can be omitted.
- anyone can call this

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail for user (from Governance Validator script)
- ProposalState (from IndividualProposal)
- VoteRecord (from IndividualProposal)
- BalanceRecord (early) - (From Governance Validator script)
- (optional) BalanceRecord (late) - (From Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail -> Governance validator Script
- VoteRecord -> IndividualProposal (set `consumed` flag to `True`)
- ProposalState -> IndividualProposal (depending on the `VoteRecord.inFavor` flag, increment `ProposalState.totalVotesInFavor` or `ProposalState.totalVotesAgainst` by the early `BalanceRecord.balanceAfterTx` amount, increment `ProposalState.voteAddressesCounted` by one).
- 1-2 BalanceRecord UTXOs -> Governance Validators (set `originalTxOutRef` fields to the input txOutRefs if this was previously a nothing)


### ExecuteProposal
Purpose: To allow a successful proposal to take an action which confirms it's success - it may update runtime configuration of a validator, or spend funds, or any number of other things not reliant on this governance spec.

Validation rules:
- `ProposalState.voteAddressesCounted` must match `ProposalState.totalVoteAddresses`
- `ProposalState.reviewTimeEnds` must have passed
- `ProposalState.totalVotesInFavor` + `ProposalState.totalVotesAgainst` must meet some constant `quorum`
- `PropsalState.totalVotesInFavor` must be greater than `ProposalState.totalVotesAgainst`
- `ProposalState.executed` must be `False`, so a proposal is executed exactly once.

inputs:
- fee/collateral UTXO (from USER)
- ProposalState (from IndividualProposal)
- (other UTXO's related to the execution effect)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- ProposalState -> IndividualProposal (set `ProposalState.executed` to `True`)
- (other UTXO's related to the execution effect)
