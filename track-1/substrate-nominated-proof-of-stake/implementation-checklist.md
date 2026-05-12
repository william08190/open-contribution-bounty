# Implementation Checklist

Use this checklist while building the Nominated Proof of Stake course chain.

## Design

- [ ] Define the local validator count.
- [ ] Define the candidate count.
- [ ] Define the nominator count.
- [ ] Define session length.
- [ ] Define sessions per era.
- [ ] Define bonding duration.
- [ ] Define slash defer duration.
- [ ] Define maximum nominations per nominator.
- [ ] Define election limits.
- [ ] Document which parameters are tutorial shortcuts.

## Dependencies

- [ ] Add `pallet_staking`.
- [ ] Add `pallet_session`.
- [ ] Add `pallet_balances`.
- [ ] Add `pallet_timestamp`.
- [ ] Add a block authoring pallet such as BABE or Aura.
- [ ] Add GRANDPA if the template uses finality.
- [ ] Add `pallet_authorship` when rewards or authorship tracking need it.
- [ ] Add `pallet_offences` for offence reporting.
- [ ] Add `pallet_election_provider_multi_phase` for production-like elections.
- [ ] Add `pallet_bags_list` when the staking config expects a voter list.
- [ ] Keep all pallets in the same Polkadot SDK release family.

## Balances

- [ ] Define `Balance`.
- [ ] Define existential deposit.
- [ ] Configure balances as staking currency.
- [ ] Test that bonded funds cannot be transferred freely.
- [ ] Test reward destination behavior.
- [ ] Test slash destination behavior.

## Session Keys

- [ ] Define the runtime `SessionKeys` type.
- [ ] Include the authoring key type.
- [ ] Include the finality key type if applicable.
- [ ] Add keys for every genesis validator.
- [ ] Verify the node can author blocks with those keys.
- [ ] Test that missing or wrong keys are visible in local verification.

## Session Pallet

- [ ] Configure validator ID.
- [ ] Configure validator ID lookup.
- [ ] Configure session ending logic.
- [ ] Configure next session rotation.
- [ ] Configure `SessionManager = Staking`.
- [ ] Configure session handler.
- [ ] Configure session keys.
- [ ] Add the pallet to the runtime construct with a unique pallet index.
- [ ] Test session rotation.

## Staking Pallet

- [ ] Configure currency.
- [ ] Configure vote conversion.
- [ ] Configure rewards.
- [ ] Configure slashing.
- [ ] Configure sessions per era.
- [ ] Configure bonding duration.
- [ ] Configure slash defer duration.
- [ ] Configure admin or governance origin.
- [ ] Configure session interface.
- [ ] Configure era payout.
- [ ] Configure election provider.
- [ ] Configure genesis election provider.
- [ ] Configure maximum nominations.
- [ ] Configure voter list.
- [ ] Configure target list.
- [ ] Configure history depth.
- [ ] Add the pallet to the runtime construct with a unique pallet index.

## Election Provider

- [ ] Configure maximum winners.
- [ ] Configure maximum backers per winner.
- [ ] Configure election lookahead.
- [ ] Configure snapshot limits.
- [ ] Configure fallback behavior.
- [ ] Benchmark or document weight limits.
- [ ] Test deterministic election results in a small setup.
- [ ] Test behavior when there are too few candidates.
- [ ] Test behavior when nominator backing changes.

## Genesis

- [ ] Fund validator accounts.
- [ ] Fund nominator accounts.
- [ ] Bond validator funds.
- [ ] Mark validator accounts as candidates.
- [ ] Add session keys.
- [ ] Bond nominator funds.
- [ ] Add nominations.
- [ ] Set desired validator count.
- [ ] Start the node and verify initial authorities.

## Tests

- [ ] Bonding works.
- [ ] Validator intent works.
- [ ] Nominations work.
- [ ] Too many nominations fail.
- [ ] Election selects expected winners.
- [ ] Session rotates to elected validators.
- [ ] Rewards are paid.
- [ ] Slashes are applied.
- [ ] Chilling works.
- [ ] Unbonding creates unlocking chunks.
- [ ] Withdrawal respects bonding duration.
- [ ] Locked stake cannot be transferred.

## Local Runbook

- [ ] Start local chain.
- [ ] Inspect initial validators.
- [ ] Submit validator intent.
- [ ] Submit nominations.
- [ ] Advance sessions.
- [ ] Advance eras.
- [ ] Inspect exposures.
- [ ] Inspect rewards.
- [ ] Simulate or test a slash.
- [ ] Chill a validator.
- [ ] Unbond and withdraw.
- [ ] Capture logs or screenshots for each lifecycle stage.
