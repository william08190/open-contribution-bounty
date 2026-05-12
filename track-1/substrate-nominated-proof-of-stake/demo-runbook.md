# Local NPoS Demo Runbook

This runbook gives maintainers and learners a concrete acceptance path for a
local Nominated Proof of Stake course chain. It assumes a Polkadot SDK node
template with staking, session, balances, timestamp, election, and consensus
pallets wired into the runtime.

The exact binary name and account model can change by SDK release. Replace
`node-template` and the example account names with the names used by the course
implementation.

## Acceptance Goal

The local demo should prove these behaviors:

- Genesis contains funded validator and nominator accounts.
- Validators have session keys before they can be healthy authorities.
- Validators can bond funds and declare validator intent.
- Nominators can bond funds and nominate validator candidates.
- The election provider selects a bounded active validator set.
- Session rotation moves the elected set into the active authorities.
- Rewards are observable after an era.
- Slashes are observable through a controlled test or mock offence.
- Chilling, unbonding, and withdrawal work with the configured delay.
- Locked stake cannot be transferred freely.

## Suggested Local Parameters

Use short tutorial constants so the lifecycle is observable in one local run:

| Parameter | Suggested value |
| --- | --- |
| Desired validators | 2 |
| Validator candidates | 4 |
| Nominators | 2 to 4 |
| Sessions per era | 2 |
| Bonding duration | 1 to 2 eras |
| Max nominations | 2 to 4 |
| Slash fraction | Small but visible |

Document that these values are tutorial settings, not production economics.

## 1. Build and Start From a Clean Chain

Build the node:

```bash
cargo build --release
```

Purge old local state before each acceptance run:

```bash
./target/release/node-template purge-chain --dev -y
```

Start the tutorial chain:

```bash
./target/release/node-template --dev --tmp
```

For a multi-validator demo, generate a local chain spec, insert session keys for
each validator node, and start each validator with the same chain spec. The
course should include the exact command names used by the chosen template.

## 2. Verify Genesis Staking State

Before sending any extrinsic, record the initial state:

- `staking.validatorCount`
- `staking.validators`
- `staking.bonded`
- `staking.ledger`
- `staking.nominators`
- `session.validators`
- `session.nextKeys`
- `balances.account`

Expected evidence:

| Check | Expected result |
| --- | --- |
| Validator accounts are funded | Free balance is above the validator bond |
| Nominator accounts are funded | Free balance is above the nominator bond |
| Initial validators are bonded | `staking.ledger` exists |
| Session keys exist | `session.nextKeys` exists for validators |
| Node produces blocks | New blocks appear without manual intervention |

## 3. Register Validator Candidates

For each validator candidate:

1. Bond stake with the configured reward destination.
2. Set session keys if the template does not insert them at genesis.
3. Call `staking.validate` with a visible commission value.
4. Query `staking.validators` and confirm the validator preferences.

Record these events:

- `staking.Bonded`
- `session.NewSession` or the template equivalent
- `staking.ValidatorPrefsSet`, when emitted by the SDK version

Negative check:

- Try to validate with insufficient bond and confirm the dispatch fails.

## 4. Add Nominations

For each nominator:

1. Bond nominator stake.
2. Nominate one or more validator candidates.
3. Query `staking.nominators`.
4. Confirm the nomination target list is bounded by `MaxNominations`.

Record these events:

- `staking.Bonded`
- `staking.Nominated`

Negative checks:

- Nominate more targets than `MaxNominations`.
- Nominate an account that is not an eligible validator candidate.
- Try to transfer bonded stake and confirm the locked transfer fails.

## 5. Advance Sessions and Eras

Wait until the configured number of sessions has passed, or use a test harness
that advances blocks to the next era.

Record these values before and after advancement:

- `session.currentIndex`
- `session.validators`
- `staking.activeEra`
- `staking.currentEra`
- `staking.erasStakers` or the SDK release equivalent
- `staking.erasValidatorPrefs`

Expected evidence:

- The active validator set matches the election output.
- Exposure includes validator self stake and nominator backing.
- A nomination change affects a later election, not the already-active era.

## 6. Prove Reward Payout

After at least one rewardable era:

1. Capture validator and nominator balances.
2. Call `staking.payoutStakers` for a validator and era when the runtime uses
   explicit payout.
3. Query balances and staking ledgers again.
4. Capture reward-related events.

Expected evidence:

- Reward destination behavior matches the runtime configuration.
- Validator commission is visible when commission is nonzero.
- Nominator reward share changes with exposure.

## 7. Prove Slashing Behavior

For a local course, prefer a controlled runtime test or mock offence over trying
to force a real consensus equivocation.

The evidence packet should include:

- the offence or slash trigger used by the test;
- the validator exposure before the slash;
- slash-related events;
- validator and nominator balances or ledgers after the slash;
- whether the slash is immediate or deferred;
- any governance or admin cancellation path, if configured.

Expected evidence:

- Validator stake is reduced.
- Nominator-backed exposure is affected according to the runtime's staking
  rules.
- The slash amount is large enough to see in balances or events.

## 8. Chill, Unbond, and Withdraw

For one validator:

1. Call `staking.chill`.
2. Advance to a later election.
3. Confirm the validator is no longer selected when enough candidates remain.

For one bonded account:

1. Call `staking.unbond`.
2. Query the unlocking chunk in `staking.ledger`.
3. Try `withdrawUnbonded` before the bonding duration ends and confirm it does
   not release funds early.
4. Advance past the bonding duration.
5. Call `withdrawUnbonded`.
6. Confirm the free balance and ledger state.

Expected evidence:

- Chilling removes validator intent.
- Unbonding creates an unlocking chunk.
- Withdrawal respects the configured delay.

## 9. Required Evidence Packet

A submission is easy to review when it includes:

- the local constants used for the demo;
- the chain spec or genesis snippet for staking and session keys;
- commands used to start the node;
- storage snapshots before and after nomination;
- storage snapshots before and after session or era rotation;
- reward payout events and balance changes;
- slash test output;
- chill, unbond, and withdrawal output;
- failing negative checks for insufficient bond, too many nominations, and
  locked transfer.

## 10. Maintainer Review Checklist

Use this checklist to accept or reject the local demo:

- [ ] The runtime compiles.
- [ ] The local node produces blocks.
- [ ] Session keys are present for validators.
- [ ] Validators can bond and validate.
- [ ] Nominators can bond and nominate.
- [ ] Election output is visible.
- [ ] Session rotation uses the elected validators.
- [ ] Reward payout is visible.
- [ ] Slash behavior is visible.
- [ ] Chilling changes later validator selection.
- [ ] Unbonding and withdrawal respect bonding duration.
- [ ] Locked stake transfer fails.
- [ ] Tutorial constants are clearly marked as non-production values.
