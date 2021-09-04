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

### Scenario 1 - infinite vote mitigation, lockless voting, and democratic procedure.

1) Staker Deposits $GOV for the first time by calling the `GovernanceValidator` with redeemer `DepositAct`

effects: 
- a BalanceRecord is minted, reflecting the balance after this tx.
- a UserStakeDetail is minted, which points at the BalanceRecord with a txOutRef

note: step 1 can be followed by any number of subsequent deposits/withdraws, minting a new BalanceRecord each time.

2) the Admin deploys a ProposalFactory

3) Staker calls the ProposalFactory Validator with redeemer `CreateProposalAct`

effects:
- new IndividualProposal Validator is initialized

4) during the voting period, the Staker attempts to vote twice by calling `IndividualProposal` Validator with redeemer `VoteAct` twice.

constraint:
- the staker must not have withdrawn their balance in order to register a vote

effects:
- a VoteRecord is minted, reflecting that the Staker's address has voted for the proposal (this happens twice in this scenario)

5) After or during the voting period, the pollster (or any other user) may call `IndividualProposal` with redeemer `ProveDuplicate`

constraint: the Pollster must provide two voting records for the same address & proposal.

effect:
- the vote is corrected
- (possibly) a portion of the Staker's $GOV stake is transferred to the caller/pollster

note: this can happen interchangeably with step 6 in case our Pollster fails to do this fails to do this (but should be automatic in off-chain code).

6) the Pollster proves votes onchain by calling `IndividualProposal` with redeemer `CountVote`

constraints:
- we can probably only do this 1 or 2 voters at a time
- vote weights are confirmed by presenting BalanceRecords/UserStakeDetails covering the period of time where the voting period ended.

effects:
- the included vote & balance records are accumulated on ProposalState.
- the included vote records are marked as consumed to prevent manipulation.

7) a review period continues for some time after the voting period, to give other users an opportunity to prove duplicate votes.

8) following the review period, the Pollster/Admin can Execute the proposal

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

3) at least 5 days from the previous Epoch end time, the Administrator (or anyone) calls `GovernanceValidator` with redeemer `TriggerEpochAct`
effects:
- a `LastEpochScriptState` is minted, recording the rewards and stake pool size at the time this occured
- `GovernanceState.rewardsTotal` is set to zero, so the next epoch will include seperate funds.

4) Staker calls `GovernanceValidator` with `ClaimRewardAct`
constraints: 
- if the user has past unclaimed epochs other than the one being claimed (available from UserStakeDetail), then we must reject and those past epochs must be claimed _first_ (automatic from offchain code).
- user must be able to prove stake balance through a combination of BalanceRecord and UserStakeDetails, transactions occuring ON the ending timstamp of the epoch are determined to be _after_ this epoch.
effect:
- the user receives a share of the rewards for the epoch proportional to the proven balance relative to the overall total of the stake for that epoch.

## One-shot Tokens

These are non-fungible tokens that should be parameterized only on  UTXO inputs, as is shown in the `Currency` use case example in the Plutus MonoRepo.

One-shot tokens within the system are:

### GovernanceState token 

  Token Type: State token for script

  purpose: allows us to manage stateful data relating to totals of important funds held by the Governance Validator

  carries datum: 
  ```
  GovernanceState { stakeTotal: Natural
                  , rewardsTotal: Value 
                  , lastEpoch :: Maybe Natural
                  , lastEpochEnded :: PosixTime
                  }
  ```
 
  initialized to 
  ```
  GovernanceState { stakeTotal = 0
                  , rewardsTotal = Value.empty 
                  , lastEpoch = Nothing
                  , lastEpochEnded = currentTime - 3 days
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
                  , lockedUntil :: Maybe PosixTime
                  , lockedBy :: MaybeAddress
                  , lastBalanceRecord :: TxOutRef
                  , lastEpochClaimed :: Natural
                  }
  ```
  
  initialized to 
  ```
  UserStakeDetail { userAddress = DepositAct.address -- determined by validator
                  , amountStaked = DepositAct.amount -- determined by validator
                  , lastReward = currentTime
                  , lockedUntil = Nothing
                  , lockedBy = Nothing
                  , lastBalanceRecord = firstBalanceRecord -- determined by validator
                  , lastEpochClaimed = GovernanceState.lastEpoch
                  }
  ```

  mint & burn: must include the GovernanceState UTXO

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
  { userAddrss = DepositAct.address / WithdrawAct.address
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


### LastEpochScriptState token (1 per epoch)
  
  Token Type: State token for script, only needed on specific actions. 
  
  purpose: This datum is needed as part of the reward distribution procedure, to record the exact state of the rewards and stakepool sizes so that they might be used to calculate reward dividends for each staker of $GOV at the end of a given reward period/epoch.
  
  similar to BalanceRecords, these form a distributed linked-list system with the head at UserStakeDetail, where new records are inserted between the UserStakeDetail and the previous record.
  
  carries datum: 
  ```
  LastEpochScriptState { stakeTotal :: Natural
                       , rewardsTotal :: Value,
                       , endTime :: PosixTime
                       , epochNumber :: Natural
                       }
  ```
  
  initialized to 
  ```
  LastEpochScriptState { stakeTotal = GovernanceState.stakeTotal
                       , rewardsTotal = GovernanceState.rewardsTotal 
                       , endTime = currentTime
                       , epochNumber = lastEpochNumber + 1 (zero-indexed)
                       -- values determined by Validator.
                       }
  ```
  
  mint & burn: must include the GovernanceState UTXO
  
  inputs:
- fee/collateral UTXO
- GovernanceState UTXO (from Governance Validator)

  outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState UTXO -> Governance Validator
- LastEpochScriptState (MINTED) -> Governance Validator

## Governance Validator
parameters: 
```
GovernanceValidatorParams { governanceStateCurrencySymbol :: CurrencySymbol,
                          , ownerAddress :: PubKeyHash
                          , danaTokenAssetClass :: AssetClass
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

  in both cases, the resulting UserStakeDetail should be sent (back) to the Governance Validator script address, with the new lastBalanceRecord txOutRef tagged
In order to minimize transaction sizes, the $GOV stake pool UTXOs should consolidate with previous deposits from all users, rather than directly transfer).
- any Wallet can deposit $GOV for Any address onchain. there is no restriction.
- any pre-existing BalanceRecord UTXO supplied must either have a txOutRef OR `BalanceRecord.originalTxOutRef` field in datum matching the `UserStakeDetail.lastBalanceRecord` field, or fail 


inputs:
- fee/collateral UTXO (from USER)
- deposit UTXO (1 or more) (from USER)
- (Optional) - consolidated stake deposit UTXO (from Governance Validator Script)
- GovernanceState token  (from Governance Validator Script)
- (optional UserStakeDetail Token) (from Governance Validator Script)
- (optional BalanceRecord Token) (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- deposit UTXO -> Governance Validator script (consolidated)
- GovernanceState token -> Governance Validator script (increment `GovernanceState.stakeTotal` by `DepositAct.amount`)
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
- if the UserStakeDetail token is being burned, then the user `WithdrawAct.amount` must equal `UserStakeDetail.amountStaked` and vice versa, the token must be burned they are equal. This is because the user's staked amount will be zero and we shouldn't keep unneccessary data/utxos around.
- any pre-existing BalanceRecord UTXO supplied must either have a txOutRef OR `BalanceRecord.originalTxOutRef` field in datum matching the `UserStakeDetail.lastBalanceRecord` field, or fail 


inputs:
- fee/collateral UTXO (from USER)
- GovernanceState token  (from Governance Validator Script)
- UserStakeDetail token  (from Governance Validator Script)
- Staked $GOV UTXO (from Governance Validator Script)
- (optional BalanceRecord Token) (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script (decrement `GovernanceState.stakeTotal` amount by `WithdrawAct.amount`)
- UserStakeDetail Token -> Governance Validator script (decrement `UserStakeDetail.amountStaked` by `WithdrawAct.amount`)
- Withdrawal $GOV UTXO (contains WithdrawAct.amount of Dana) -> User Wallet
- (Optional) Remainder UTXO (contains any $GOV from input which exceeds `WithdrawAct.amount` -> Governance Validator Script
- (optional) BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- BalanceRecord Token (MINTED) -> Governance Validator (set `balanceAfterTx` field to the new balance the user holds in the stakepool following the tranaction)

### ProvideRewardAct

Purpose: receive rewards distributed by scripts or individuals for $GOV stakers, queue these rewards for fair distribution at the end of the next reward period/epoch.

Validation Rules:
- user may include one or more UTXOs with rewards in collections of arbitrary native tokens, (reward UTXOs) these should total to match ProvideRewardsAct.value

inputs:
- fee/collateral UTXO (from USER)
- Reward UTXOs (from USER - contains arbitrary tokens)
- GovernanceState token (from Governance Validator Script)
- optional previous reward UTXO for this epoch (from Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- GovernanceState token -> Governance Validator script (increment `GovernanceState.rewardsTotal` by `ProvideRewardsAct.value`)
- reward UTXOs -> Governance Validator Script (consolidate to single utxo per epoch)

### ClaimRewardAct / DistributeRewardAct

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
- GovernanceState token  (from Governance Validator Script)
- UserStakeDetail token (from governance Validator Script)
- BalanceRecord Token (from Governance Validator Script)
- (Optional) BalanceRecord Token (from Governance Validator Script )
- LastEpochScriptState (from Governance Validator Script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- Reward UTXO remainder -> Governance Validator Script
- GovernanceState token -> Governance Validator Script
- UserStakeDetail Token -> Governance Validator script (increment `UserStakeDetail.lastEpochClaimed`)
- BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- BalanceRecord Token -> Governance Validator (set `BalanceRecord.originalTxOutRef` to the consumed TxOutRef if it was a `Nothing`)
- LastEpochScriptState Token -> Governance Validator

### TriggerEpochAct

Purpose: the first step in distributing rewards, records the rewards totals for a given reward period/epoch and resets state to track for the new epoch

note: this will rely on a constant `epochLength :: PosixTimeRange`

Validation rules:
- `GovernanceState.lastEpochEnds` must be at least `epochLength` less than the `currentTime`.
- anyone can call this

inputs:
fee/collateral UTXO remainder (from USER)
GovernanceState token (from Governance Validator script)

outputs:
fee/collateral UTXO remainder -> user Wallet
GovernanceState token -> Governance Validator script (set `GovernanceState.rewardsTotal` to `Value.empty`, set `GovernanceState.lastEpochEnds` to current time,  increment `GovernanceState.lastEpoch` )
LastEpochScriptState state token (MINTED)-> Governance Validator Script (set `LastEpochScriptState.stakeTotal` to `GovernanceState.stakeTotal`, and `LastEpochScriptState.rewardsTotal` to `GovernanceState.rewardsTotal`, `LastEpochScriptState.epochNumber` to `GovernanceState.lastEpoch` + 1)

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
  Token Type: Record token witnessing a vote that occured
  
  Purpose: displays a user's vote, which is tabulated after a voting period ends
  
carries datum :
``` 
VoteRecord
  { address :: Address
  , inFavor :: Bool
  , consumed :: Bool
  }
```
initialized to:
```
  VoteRecord
    { address = VoteAct.Address
    , inFavor = VoteAct.inFavor
    , consumed = False
    }
```
minting & Burning:
- must include a ProposalState token
- can only be burned if a ProposalState token is included in the transaction (when vote duplication has occured)

inputs:
- fee/collateral UTXO remainder (from USER)

outputs:
- fee/collateral UTXO remainder -> user Wallet
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
| CountVotes
| ProveDuplicate
| Execute

```

### VoteAct
Purpose: to record a user's vote on a particular proposed value.

Validation Rules
- `UserStakeDetail.address` must validate the transaction
- transaction must occur before `ProposalState.votingEnds`
- `UserStakeDetail.amountStaked` must be greater than 0

inputs:
- fee/collateral UTXO (from USER)
- UserStakeDetail for user (from Governance Validator script)
- ProposalState (from IndividualProposal)
- BalanceRecord (early) - (From Governance Validator script)
- (optional) BalanceRecord (late) - (From Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- UserStakeDetail -> Governance validator Script (set `UserStakeDetail.lockedBy` to IndividualProposal Validator Address and `UserStakeDetail.lockedUntil` to the end of the voting time for the proposal OR the current value of `lockedUntil` whichever is greater/further in the future.
- ProposalState -> IndividualProposal (increment `ProposalState.totalVoteAddresses`)
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

### ProveDuplicate
Purpose: to allow reporting and correction of Vote Duplication before _or after_ a vote has been counted.

note: if necessary we can punish people who vote twice by fining them some $GOV and rewarding the caller.

Validation rules
- anyone can call this
- the early/late BalanceRecords and/or the UserStakeDetail must validate each other using the same scheme in `CountVote`
- the early balance record is the source of truth on the vote amount to be used in correcting the vote count

inputs:
- fee/collateral UTXO (from USER)
- ProposalState (from IndividualProposal)
- first VoteRecord (from IndividualProposal)
- second VoteRecord (from IndividualProposal)
- BalanceRecord (early) - (From Governance Validator script)
- (optional) BalanceRecord (late) - (From Governance Validator script)
- (optional)UserStakeDetail for VOTER (from Governance Validator script)

outputs:
- fee/collateral UTXO remainder -> user Wallet
- VoteRecord -> IndividualProposal
- ProposalState -> IndividualProposal (if the VoteRecords have been consumed, subtract the early `BalanceRecord.balanceAfterTx` from either `ProposalState.totalVotesInFavor` or `ProposalState.totalVotesAgainst` depending on `VoteRecord.inFavor` flag.  if the two votes have opposite `inFavor` values, do nothing)
- first VoteRecord -> IndividualProposal
- second VoteRecord (BURN) (this must the the vote against if the votes have opposing `inFavor` flags)
- 1-2 BalanceRecord UTXOs -> Governance Validators
- UserStakeDetail -> Governance validator Script


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
