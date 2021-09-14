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
  - a UserStakeDetail is minted
  - GovernanceState.totalStakers is updated (only the first time).

note: step 1 can be followed by any number of subsequent deposits/withdraws

2) Staker B Deposits $GOV as well, creating a similar paper trail.


3) the Admin deploys a ProposalFactory

4)Staker calls the ProposalFactory Validator with redeemer `CreateProposalAct`

  effects:
  - new IndividualProposal Validator is initialized

5) Staker A attempts to vote, in order to do so the staker may need to call `IndividualProposal` with Redeemer `VoteAct`

  constraints:
  - if Staker A has previously cast a vote for this IndividualProposal, this will fail


  effects:
  - ProposalState is consumed, updating a running tally of votes.
  - Staker A's UserStakeDetail.lockedBy is updated to include a reference to the Proposal and the intention ( `For | Against`) such that the vote is easily reversed from this information


6) Staker B wishes to delegate voting rights to Staker A, Staker B calls `GovernanceValidator` with `DelegateVoteAct` redeemer.

  constraints:
  - Staker B must not have already delegated their votes

  effects:
  - Staker B's UserStakeDetail.delegateTo field is updated to the desired delegatee's address.


7) Staker A wishes to use a delegated vote, Staker A must supply the `UserStakeDetail` UTXO of the delegated address  to `VoteAct`, instead of testing for Staker A's address, the Validator uses the address from `UserStakeDetail.delegateTo`  .

  constraints: neither Staker B nore Staker A on behalf of staker B may have previously voted on this `IndividualProposal`
  (this operation is unique per UserStakeDetail, since the locking mechanism is checked against values there)
  Note: this means there is a hard upper limit of Proposals that a user can vote on at any given time, however Validator and offchain code should only check against non-exipired Proposals (and automatically remove the expired items.)

  effects:
  - Staker B's `UserStakeDetail` is updated to reflect the vote on Staker B's behalf (lockedBy field in particular).

  
8) No counting phase is necessary since counting occurs at the time of the vote only. the success/failure of the proposal can be known at any time  since we increment counts as the votes are cast., anyone can call `ExecuteAct`

  constraints:
  - votes must be fully counted (this is verifiable within ProposalState)
  - Proposal success parameters must be met
  - proposal must not have been executed previously

### Scenario 2 - reward dispersal

1) Staker Deposits $GOV for the first time by calling the `GovernanceValidator` with redeemer `DepositAct`

  effects: 
  - a UserStakeDetail is minted

note: step 1 can be followed by any number of subsequent deposits/withdraws, minting a new BalanceRecord each time.

2) the Admin provides rewards by calling the `GovernanceValidator` with redeemer `ProvideRewardAct`

effect
- GovernanceState is consumed and we increment `rewardsTotal`

3) at least 5 days from the previous Epoch end time, the Administrator (or anyone) calls `GovernanceValidator` with redeemer `TriggerEpochAct` n times where n scales linearly with the number of staking users.

  effects:
  - a `LastEpochProof` is minted, then iterated over the `UserStakeDetail`s until complete, progressively summing the totals of each stake. (there may be several LastEpochScriptState tokens minted during this process to acheive Log n behavior, however they will be burned as we conclude the folding operation.)
  - for each user, we mint a `UserStakeSnapshot` which allows us to pinpoint the user's balance at the end of the reward cycle, the `UserStakeSnapshot` uses the Epoch number stored on `LastEpochProof` to assign an epoch number to the snapshot, denoting the stake share for that user in the given epoch, which we can then use for fair reward dispersal.
  - once we can merge `LastEpochProof` to by proving that the sum includes the entire range of users, we can mint `LastEpochScriptState`

  - (at the end) - `GovernanceState.rewardsTotal` is set to zero, so the next epoch will include seperate funds.

4) Staker calls `GovernanceValidator` with `ClaimRewardAct`

  constraints: 
  - if the user has past unclaimed epochs other than the one being claimed (available from UserStakeDetail), then we must reject and those past epochs must be claimed _first_ (automatic from offchain code).
  - user must be able to prove stake balance through a `UserStakeSnapshot`,

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
                  , lastBalanceRecordTimeStamp :: PosixTime
                  , lastEpochClaimed :: Natural
                  , lockedBy :: [(ValidatorHash, (Bool, Natural)] -- the vote, for/against and the balance that user had at the time the vote was cast.
                  , delegatedTo :: Maybe Address
                  , index :: Natural
                  }
  ```
  
  initialized to 
  ```
  UserStakeDetail { userAddress = DepositAct.address -- determined by validator
                  , amountStaked = DepositAct.amount -- determined by validator
                  , lastReward = currentTime
                  , lastBalanceRecordTimeStamp = firstBalanceRecordTimeStamp -- determined by validator
                  , lastEpochClaimed = GovernanceState.lastEpoch
                  , lockedBy = []
                  , delegatedTo = Nothing
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

### UserStakeSnapshot Token

Token Type: a Record token which allows us to document the user's past balance updates such that the user has mechanisms to fairly claim rewards.

Purpose: used to prove stake at the _end_ of a given reward period. Minted by an administrator performing the `TriggerEpochAct`

carries Datum:
```
UserStakeSnapshot
  { userAddress :: Address
  , stakeDetailIndex :: Natural
  , stakedAmount :: Natural
  , snapshotForEpoch :: Natural
  }
```
initialized to:
```
UserStakeSnapshot
  { userAddress = DepositAct.address / WithdrawAct.address / UserStakeDetail.Address
  , stakeDetailIndex = UserStakeDetail.index
  , stakedAmount = balance
  , snapshotForEpoch = LastEpochScriptState.epochNumber
  }
  -- determined by validator
  ```
  
minting:
- must include GovernanceState

cannot be burned

inputs:
- fee/collateral UTXO
- GovernanceState UTXO (from Governance Validator)
- UserStakeDetail UTXO (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState UTXO -> Governance Validator
- UserStakeDetail (MINTED?) -> Governance Validator
- BalanceRecord (MINTED) -> Governance Validator


### LastEpochProof token (Log n Tokens per epoch)

Token Type: a Permission token that witnesses a progressive fold over `UserStakeDetail`s to find the total stake amount at the end of the reward period.

Purpose: this allows us to confirm that each user's stake is accounted for when totalling stakes for reward dispersals without double-inclusion, or leaving any out, and without having GovernanceState in contention for common operations.

carries Datum:
```
LastEpochProof
  { countedIndexRangeStart :: Natural
  , countedIndexRangeEnd :: Natural
  , epochNumber :: Natural
  , runningTotalStake :: Value
  , epochEndedTime :: PosixTime 
  }
```
Initialized to:
```
LastEpochProof
  { countedIndexRangeStart = 0 -- or an optimum starting position
  , countedIndexRangeEnd = 0 -- or an optimum starting position
  , epochNumber = GovernanceState.lastEpoch + 1
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
- GovernanceState (from Governance Validator)
- one or more sets of:
  - UserStakeDetail UTXO (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState UTXO -> Governance Validator
- all sets of 
  - UserStakeDetail -> Governance Validator
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
| DelegateVoteAct
  { address :: Maybe Address }
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
- user may have included a `UserStakeDetail` from the Governance validator script matching their `DepositAct.address`, OR user must have provided the GovernanceState token so that a UserStakeDetail is minted for them (with index corresponding to the current `GovernanceState.totalStakers`)

  in both cases, the resulting UserStakeDetail should be sent (back) to the Governance Validator script address, with the new lastBalanceRecord txOutRef tagged
- any Wallet can deposit $GOV for Any address onchain. there is no restriction.
- any pre-existing BalanceRecord UTXO supplied must either have a txOutRef OR `BalanceRecord.originalTxOutRef` field in datum matching the `UserStakeDetail.lastBalanceRecord` field, or fail 

*known issue: we don't explicitly prevent duplicate addresses here,  hypothetically if this became an issue we could provide a schema endpoint to perform a merger using existing redeemers*

inputs:
- fee/collateral UTXO (from USER)
- deposit UTXO (1 or more) (from USER)
- (Optional) GovernanceState token  (from Governance Validator Script) (required on first call
- (optional UserStakeDetail Token) (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- deposit UTXO -> Governance Validator script (consolidated)
- (Optional) GovernanceState token -> Governance Validator script (increment `GovernanceState.totalStakers`)
- UserStakeDetail Token (MINTED?) -> Governance Validator script (increment `UserStakeDetail.amountStaked` by `DepositAct.amount` OR mint fresh with `DepositAct.amount` as the initial value (other values should be initialized to defaults, `UserStakeDetail.address` should match `DepositAct.address`).

### WithdrawAct

Purpose: The user removes $GOV from their staked totals , may burn UserStakeDetail.
Validation Rules:
- if UserStakeDetail for the user's address is not included in the tx, fail.
- if amount specified in WithdrawAct.amount is greater than the balance of the `UserStakeDetail.amountStaked`, fail
- transaction must be signed by `UserStakeDetail.address` if it is a PubKeyHash, or include a utxo from the validator if it is a ValidatorHash.


inputs:
- fee/collateral UTXO (from USER)
- (optional) GovernanceState token  (from Governance Validator Script)
- UserStakeDetail token  (from Governance Validator Script)
- Staked $GOV UTXO (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail Token -> Governance Validator script (decrement `UserStakeDetail.amountStaked` by `WithdrawAct.amount`)
- Withdrawal $GOV UTXO (contains WithdrawAct.amount of GOV) -> User Wallet
- (Optional) Remainder UTXO (contains any $GOV from input which exceeds `WithdrawAct.amount` -> Governance Validator Script

### DelegateVoteAct
Purpose: To allow a user to vote on one's behalf.

Validation rules:
- user address matching `UserStakeDetail.userAddress` must validate this transaction

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail token  (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail token -> GovernanceValidator script (set `UserStakeDetail.delegatedTo` to `DelegateVoteAct.address`)

### ProvideRewardAct

Purpose: receive rewards distributed by scripts or individuals for $GOV stakers, queue these rewards for fair distribution at the end of the next reward period/epoch.

Validation Rules:
- user may include one or more UTXOs with rewards in collections of arbitrary native tokens, (reward UTXOs) these should total to match ProvideRewardsAct.value
- in a real implementation, we need to limit the token AssetClasses allowed for reward, otherwise this could be an attack vector.

inputs:
- fee/collateral UTXO (from USER)
- Reward UTXOs (from USER - contains arbitrary tokens)
- GovernanceState token (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script (increment `GovernanceState.rewardsTotal` by `ProvideRewardsAct.value`)
- reward UTXOs -> Governance Validator Script 

### ClaimRewardAct

Purpose: a User who staked $GOV during a previous reward period  claims their share of the rewards.

Validation rules
- `LastEpochScriptState.epochNumber` must be equal to (`UserStakeDetail.lastEpochClaimed` +1)
- `UserStakeSnapshot.epochNumber` must match `LastEpochScriptState.epochNumber`
- `UserStakeDetail.address must validate this transaction`
- amount transferred to user is a value defined by: (`UserStakeSnapshot.amountStaked` / `LastEpochScriptState.stakeTotal`) * `LastEpochScriptState.rewardsTotal`

inputs:
- fee/collateral UTXO (from USER)
- reward UTXO (1 or more) (from Governance Validator Script)
- UserStakeSnapshot token (from governance Validator Script)
- LastEpochScriptState (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- Reward UTXO remainder -> Governance Validator Script
- UserStakeDetail Token -> Governance Validator script (increment `UserStakeDetail.lastEpochClaimed`)
- BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- LastEpochScriptState Token -> Governance Validator

### EpochCountingAct
-- note: this may need to be split into the setup action (mints `LastEpochProof`) and the iterative action (burns `LastEpochProof` as they are progressively merged)

Purpose: the first step in distributing rewards, mints an optimum number of utxos containing `LastEpochProof`

note: this will rely on a constant `epochLength :: DiffMilliSeconds`

Validation rules:
- `GovernanceState.lastEpochEnds` must be at least `epochLength` less than the `currentTime`.
- a trusted user/`owner address` would call this.
- if this is not called with a `LastEpochProof` token + datum supplied, then one must be minted
- if this is called with a `LastEpochProof`, then we can only accept an ordered subset of `UserStakeDetail` whose `index` is immediately adjacent to the `LastEpochProof.countedIndexRangeStart` or `LastEpochProof.countedIndexRangeEnd`
- if two `LastEpochProof` are supplied with adjacent ranges, we can merge them.
- the `UserStakeDetail.index` must match the value used as the starting value for `countedIndexRangeStart` and unless a set of `UserStakeDetail` is provided, also `countedIndexRangeEnd`
- if a set is provided, they must fill out a numerical order and the transaction must fit on chain.
- a `UserStakeSnapshot` is minted, with values copied directly from `UserStakeDetail` and/or `LastEpochProof`

note: with this we should be able to scale this logarithmically.

inputs:
- fee/collateral UTXO remainder (from USER)
- GovernanceState token (from Governance Validator script)
- 1 or more: LastEpochProof UTXO (from Governance Validator Script)
- one or more sets of:
  - UserStakeDetail UTXO (from Governance Validator)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script 
- LastEpochProof UTXO(s) (Minted/Burned?) ->  Governance Validator Script (modify range fields to include valid indexes from userStakeDetail, or from merging `LastEpochProof`
- all sets of 
  - UserStakeDetail -> Governance Validator
- UserStakeSnapshot UTXO  (MINTED) -> GovernanceValidator

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



### VoteRecord Token
-- note: this token is currently deprecated, although if the current strategy  causes contention issues on `ProposalState` during the voting process, it may be added back as an optimization.

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
| UnVoteAct
| ExecuteAct

```

### VoteAct
Purpose: to record a user's vote on a particular proposed value.

Validation Rules
- `UserStakeDetail.address` must validate the transaction
- transaction must occur before `ProposalState.votingEnds`
- `UserStakeDetail.amountStaked` must be greater than 0
- `UserStakeDetail.lockedBy` must not include this proposal's ValidatorHash


inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail for user (from Governance Validator script)
- ProposalState (from IndividualProposal)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail -> Governance validator Script (set `UserStakeDetail.lockedBy` to IndividualProposal Validator Address and the voting parameters/`UserStakeDetail.amountStaked` at the time of voting so that we have sufficient capability to reverse the vote accurately.
- ProposalState -> IndividualProposal (increment `ProposalState.totalVoteAddresses`, as well as the `inFavour` or `against` fields)

### UnVoteAct
Purpose: to reverse a user's vote so that they might vote again with a different balance.

Validation Rules:
- If the Proposal has been executed or expired, remove from `UserStakeDetail.lockedBy` ONLY, do not tamper with the `ProposalState` in this case.
- Otherwise, we need to reverse the vote in ProposalState by the amount/intention specified by the entry for this Proposal's ValidatorHash, in `UserStakeDetail.lockedBy`

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail for user (from Governance Validator script)
- ProposalState (from IndividualProposal)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail -> Governance validator Script (reverse `lockedBy` field
- ProposalState -> IndividualProposal (decrement `ProposalState.totalVoteAddresses`, `inFavour` or `against` fields according to `UserStakeDetail.lockedBy` entry)

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
