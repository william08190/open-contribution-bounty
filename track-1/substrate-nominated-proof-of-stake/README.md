# Substrate Nominated Proof of Stake Course

This module shows how to design a Substrate-based chain that uses Nominated
Proof of Stake. In an NPoS system, validators produce or finalize blocks, while
nominators back validators with stake. The runtime uses that backing to choose an
active validator set, distribute rewards, and apply slashing when validators
misbehave.

This course is a runtime design and implementation guide. It does not ask
learners to copy Polkadot's production economics. Instead, learners build a
small local chain that demonstrates bonding, nominating, validator selection,
session rotation, rewards, and slashing with parameters that are easy to test.

## Learning Goals

After completing this module, a learner should be able to:

- Explain the roles of validator, nominator, stash, controller, exposure, era,
  and session.
- Add the staking-related pallet set to a Substrate runtime.
- Configure a small validator election for a local chain.
- Wire staking to sessions so elected validators become consensus authorities.
- Add local session keys for validators.
- Bond funds, declare validator intent, nominate validators, and rotate the
  active set.
- Test rewards, slashes, chilling, unbonding, and withdrawal.
- Explain which parameters are tutorial shortcuts and which require production
  economic analysis.

## Mental Model

NPoS is easier to understand as a pipeline:

```text
accounts bond funds
    |
    v
validators declare intent
nominators choose validators
    |
    v
staking pallet calculates exposures
    |
    v
election provider selects active validators
    |
    v
session pallet rotates validator set and session keys
    |
    v
consensus pallets use the active authorities
    |
    v
staking pays rewards or applies slashes by era
```

`pallet_staking` manages stake, roles, eras, exposures, rewards, slashes, and
unbonding. `pallet_session` turns the elected validator set into the session
authorities used by the consensus pallets. The election provider chooses the
active validator set from validator candidates and nominator backing.

## Core Terms

### Validator

A validator is a candidate to author or finalize blocks. Validators bond stake
and must set session keys. If selected, they become part of the active set.

### Nominator

A nominator backs one or more validators with bonded stake. Nominators share in
rewards and slash risk according to the runtime's staking rules.

### Stash and Controller

Many Substrate staking designs separate the stash account, which holds bonded
funds, from the controller account, which sends staking operations. Recent
runtime designs may simplify this pattern, so learners should check the SDK
version they are using.

### Era

An era is the accounting period for staking rewards and slashes. Active exposure
is usually calculated per era.

### Session

A session is the period over which a validator set and its session keys are
active for consensus. Several sessions can fit inside one era.

### Exposure

Exposure is the stake behind an active validator, including the validator's own
stake and nominator backing.

## Repository Starting Point

Start from a recent Polkadot SDK node template. Exact file names vary, but the
work usually touches:

```text
runtime/
  Cargo.toml
  src/lib.rs
  src/configs/
  src/weights/
node/
  src/chain_spec.rs
```

This course assumes the chain already has:

- `frame_system`;
- `pallet_balances`;
- a block authoring pallet such as BABE or Aura;
- a finality pallet such as GRANDPA, when the template includes finality;
- a local node service that can inject session keys.

## Step 1: Choose the Local NPoS Scope

Keep the first implementation small:

```text
desired validators: 2 or 3
validator candidates: 4 or 5
nominators: 4 to 8
sessions per era: small enough for tests
bonding duration: short enough for local verification
slash: small but visible
```

This lets learners observe election, reward, and unbonding behavior without
waiting for production-scale eras.

## Step 2: Add Runtime Dependencies

Minimum pallet set:

```text
pallet_staking
pallet_session
pallet_balances
pallet_timestamp
```

Common supporting pallets:

```text
pallet_babe or pallet_aura
pallet_grandpa
pallet_authorship
pallet_offences
pallet_election_provider_multi_phase
pallet_bags_list
pallet_session_historical
```

For a learning chain, it is acceptable to start with a simpler election provider
or a small on-chain election. For production-like NPoS, learners should study
the current Polkadot SDK election provider and benchmarking requirements.

Keep all dependencies in the same Polkadot SDK release family. Do not mix crate
versions from different SDK releases.

## Step 3: Configure Balances and Currency

Staking needs a currency pallet because it reserves or locks stake. The course
chain should define:

- the native `Balance` type;
- existential deposit;
- staking currency type;
- reward destination behavior;
- slash destination behavior.

The first test should prove that bonded funds cannot be freely transferred while
they are locked for staking.

## Step 4: Configure Session Keys

Validators need session keys for the consensus pallets. A typical runtime has a
session key type that combines authoring and finality keys:

```rust
impl_opaque_keys! {
    pub struct SessionKeys {
        pub babe: Babe,
        pub grandpa: Grandpa,
    }
}
```

Templates that use Aura instead of BABE will use the Aura key instead. The
course should teach the concept, not force a specific consensus engine.

Local chain spec requirements:

- each initial validator has a stash or validator account;
- each validator has session keys;
- genesis state contains the initial validator set;
- the node can author blocks with the configured keys.

## Step 5: Configure `pallet_session`

The session pallet controls validator set rotation and session key assignment.
Example shape:

```rust
impl pallet_session::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type ValidatorId = AccountId;
    type ValidatorIdOf = StashOf<Self>;
    type ShouldEndSession = Babe;
    type NextSessionRotation = Babe;
    type SessionManager = Staking;
    type SessionHandler = <SessionKeys as OpaqueKeys>::KeyTypeIdProviders;
    type Keys = SessionKeys;
    type WeightInfo = pallet_session::weights::SubstrateWeight<Runtime>;
}
```

The important invariant is that staking supplies the next validator set, while
the consensus pallet decides session boundaries. Exact associated types depend on
the chosen template and SDK version.

## Step 6: Configure `pallet_staking`

`pallet_staking` is the center of NPoS. It needs types for currency, rewards,
sessions, election, offence handling, and operational limits.

Example shape:

```rust
impl pallet_staking::Config for Runtime {
    type Currency = Balances;
    type CurrencyBalance = Balance;
    type UnixTime = Timestamp;
    type CurrencyToVote = CurrencyToVoteHandler;
    type RewardRemainder = Treasury;
    type RuntimeEvent = RuntimeEvent;
    type Slash = Treasury;
    type Reward = ();
    type SessionsPerEra = SessionsPerEra;
    type BondingDuration = BondingDuration;
    type SlashDeferDuration = SlashDeferDuration;
    type AdminOrigin = EnsureRoot<AccountId>;
    type SessionInterface = Self;
    type EraPayout = EraPayout;
    type NextNewSession = Session;
    type MaxNominations = MaxNominations;
    type VoterList = BagsList;
    type TargetList = Staking;
    type ElectionProvider = ElectionProviderMultiPhase;
    type GenesisElectionProvider = OnChainElectionProvider;
    type MaxUnlockingChunks = MaxUnlockingChunks;
    type HistoryDepth = HistoryDepth;
    type BenchmarkingConfig = StakingBenchmarkingConfig;
    type WeightInfo = pallet_staking::weights::SubstrateWeight<Runtime>;
}
```

Do not treat this as copy-paste code. Staking configuration changes across SDK
versions. The learner should use this as a map for what needs to be wired, then
check the Rust docs for the exact trait in the selected release.

## Step 7: Configure the Election Provider

The election provider chooses active validators from candidates and nominations.
A small learning chain can start with simple limits:

```text
MaxWinners: 3
MaxBackersPerWinner: small value
MaxNominations: small value
ElectionLookahead: short
```

Production-like NPoS usually needs:

- off-chain election workers;
- signed or unsigned election submissions;
- snapshot limits;
- fallback behavior;
- benchmarking for election weights;
- monitoring when elections fail.

Learners should add tests for:

- too few validators;
- more candidates than winners;
- nominators backing multiple validators;
- election result changes after stake changes.

## Step 8: Add Bags List When Needed

`pallet_bags_list` helps order voters by stake so staking can operate within
bounded weight. In a tiny local chain, learners may not feel the need for it, but
it is important when voter counts grow.

Course guidance:

- include it when using staking configs that expect `VoterList`;
- configure bag thresholds for the local balance scale;
- test that nominators can be inserted and repositioned;
- document that production thresholds need chain-specific tuning.

## Step 9: Add Offences and Slashing

A chain should define what happens when validators equivocate or otherwise
misbehave. The full offence pipeline depends on consensus choice, but the course
should cover the staking-side result:

- slash exposure;
- defer slash when configured;
- allow governance or admin cancellation during the defer period if the runtime
  supports it;
- reward reporters when applicable;
- emit events that make the slash auditable.

For local tests, use a controlled mock offence or staking test helper rather than
trying to force a real consensus equivocation.

## Step 10: Add Genesis Configuration

A local chain should include:

```text
Alice validator candidate
Bob validator candidate
Charlie validator candidate
Dave validator candidate
Eve nominator
Ferdie nominator
```

Genesis setup:

- give each account enough balance;
- bond validator stake;
- set validator intent;
- set session keys;
- bond nominator stake;
- add nominations;
- set desired validator count.

The first local run should produce blocks and rotate sessions without manual
intervention.

## Step 11: Run the Staking Flow

Happy path:

```text
1. Alice and Bob bond funds.
2. Alice and Bob call validate.
3. Eve and Ferdie bond funds.
4. Eve and Ferdie nominate Alice and Bob.
5. The chain advances to the next election.
6. Staking selects the active validator set.
7. Session rotates to that validator set.
8. Blocks continue to be produced.
9. Era reward is paid.
10. A nominator changes nominations.
11. The next election reflects the changed backing.
12. A validator chills and stops being selected.
13. A bonded account unbonds and later withdraws.
```

Failure path:

```text
wrong session keys
insufficient bond
too many nominations
too many unlocking chunks
transfer of locked funds
validator set below minimum
election provider fallback
slash event
```

## Step 12: Add Observability

Learners should know which events and storage items prove the system works:

- staking ledgers;
- bonded account map;
- validators;
- nominators;
- eras and sessions;
- current elected exposures;
- active session validators;
- rewards and slashes;
- unlocking chunks.

The runbook should include commands or UI steps that inspect these values before
and after the election.

## Local Acceptance Runbook

Use [demo-runbook.md](demo-runbook.md) as the maintainer acceptance path. It
shows the storage items, events, extrinsics, and negative checks that prove the
course chain is more than explanatory text. A complete submission should attach
the runbook output, logs, or screenshots to show that nominations change later
validator exposure and that rewards, slashes, chilling, unbonding, and locked
transfer failures are observable.

## Security and Economics Notes

- Do not copy Polkadot's staking parameters without analysis.
- Do not launch production NPoS without benchmarking election and staking
  weights.
- Do not make the minimum validator bond trivially cheap.
- Do not set `MaxNominations` higher than the election provider can handle.
- Do not ignore what happens when elections fail.
- Do not forget session keys; a selected validator without usable keys hurts
  liveness.
- Do not make the slash fraction invisible in tests.
- Do not shorten bonding duration so much that the economic signal is meaningless
  in production.

## Common Mistakes

### Treating PoS and NPoS as the Same

Pure PoS selects validators from their own stake. NPoS adds nominator backing and
spreads stake across selected validators. That extra nomination layer is the main
topic of this module.

### Wiring Staking Without Sessions

If staking elects validators but sessions never rotate to them, the elected set
does not become the consensus authority set.

### Ignoring Election Limits

Validator and nominator counts must stay within configured limits. A tutorial can
use small numbers, but production chains need realistic stress tests.

### Skipping Key Management

Validators need session keys. A candidate with bonded stake but missing session
keys should not be treated as a healthy validator.

### Hiding Slashing

NPoS is not only rewards. Learners must see what risk validators and nominators
take when backing a validator.

## Suggested Tests

Unit tests:

- bonding reserves or locks funds;
- validator intent is recorded;
- nomination list is recorded;
- too many nominations fail;
- election selects the expected winners in a small deterministic setup;
- session rotates to the elected validators;
- reward payout updates balances or reward destinations;
- slash reduces exposure;
- chilling removes validator intent;
- unbonding creates an unlocking chunk;
- withdrawal works only after the bonding duration;
- locked funds cannot be transferred.

End-to-end local node checks:

- start a local chain with at least two validators;
- inspect current session validators;
- submit nominations from two accounts;
- advance through at least one era;
- verify the active set and exposures;
- trigger or simulate a slash in a controlled test environment;
- chill one validator and verify the later active set changes;
- unbond and withdraw after the configured delay.

## Learner Deliverables

At the end of the module, the learner should submit:

- a design note with validator count, session length, era length, bonding
  duration, slashing policy, and election limits;
- runtime configuration for staking, session, election, and supporting pallets;
- genesis configuration for validators, nominators, balances, and session keys;
- tests for bonding, nominating, election, session rotation, reward, slash, and
  unbonding;
- a local runbook that demonstrates the full staking lifecycle;
- logs or screenshots proving that nominations affect the active validator set.

## References

- Polkadot staking overview:
  <https://wiki.polkadot.com/learn/learn-staking/>
- `pallet_staking` Rust docs:
  <https://paritytech.github.io/polkadot-sdk/master/pallet_staking/>
- `pallet_session` Rust docs:
  <https://paritytech.github.io/polkadot-sdk/master/pallet_session/>
- `pallet_election_provider_multi_phase` Rust docs:
  <https://paritytech.github.io/polkadot-sdk/master/pallet_election_provider_multi_phase/>
- Add an existing pallet to a runtime:
  <https://docs.polkadot.com/parachains/customize-runtime/add-existing-pallets/>
