# Review Checklist

Use this checklist to review a learner's Nominated Proof of Stake implementation
before accepting the module.

## Conceptual Accuracy

- [ ] The submission distinguishes PoS from NPoS.
- [ ] It defines validators.
- [ ] It defines nominators.
- [ ] It defines stash and controller accounts, or explains the SDK version's
      current account model.
- [ ] It defines sessions.
- [ ] It defines eras.
- [ ] It defines exposure.
- [ ] It explains how nominations affect validator selection.
- [ ] It explains reward and slash sharing.

## Runtime Wiring

- [ ] Staking is configured.
- [ ] Session is configured.
- [ ] Balances are configured as staking currency.
- [ ] Timestamp is configured when required.
- [ ] Consensus pallets are compatible with session keys.
- [ ] Election provider is configured.
- [ ] Bags list is configured when required.
- [ ] Offences or slashing path is configured.
- [ ] Runtime pallet indices are unique.
- [ ] The configuration matches one Polkadot SDK release family.

## Session and Consensus

- [ ] Validators have session keys.
- [ ] Genesis validators can author blocks.
- [ ] Staking provides the session manager.
- [ ] Session rotation changes the active authority set.
- [ ] Missing or invalid keys are handled visibly.

## Election Behavior

- [ ] The desired validator count is documented.
- [ ] Candidate limits are documented.
- [ ] Nominator limits are documented.
- [ ] A deterministic small election is tested.
- [ ] Too few validators is tested.
- [ ] Too many nominations is tested.
- [ ] Changing stake or nominations changes later exposure where expected.
- [ ] Election fallback behavior is documented.

## Economics and Safety

- [ ] Bonding duration is documented.
- [ ] Slash defer duration is documented.
- [ ] Reward destination behavior is documented.
- [ ] Slash destination behavior is documented.
- [ ] Bonded funds cannot be transferred freely.
- [ ] Slashing affects validator and nominator exposure.
- [ ] Tutorial constants are not presented as production economics.
- [ ] Weight and benchmarking requirements are mentioned.

## Tests

- [ ] Bonding is tested.
- [ ] Validator intent is tested.
- [ ] Nominations are tested.
- [ ] Election is tested.
- [ ] Session rotation is tested.
- [ ] Reward payout is tested.
- [ ] Slashing is tested.
- [ ] Chilling is tested.
- [ ] Unbonding is tested.
- [ ] Withdrawal after bonding duration is tested.
- [ ] Locked transfer failure is tested.

## Documentation Quality

- [ ] The course can be followed from a clean template.
- [ ] Code snippets are clearly marked as release-dependent shapes when they are
      not exact drop-in code.
- [ ] The local runbook demonstrates the full staking lifecycle.
- [ ] The module links to Polkadot or Polkadot SDK references.
- [ ] Common mistakes are documented.
- [ ] Production caveats are visible.

## Acceptance Criteria

- [ ] The runtime compiles.
- [ ] The tests pass.
- [ ] A local node can produce blocks.
- [ ] Validators and nominators can bond funds.
- [ ] Nominations affect election results.
- [ ] Sessions rotate to the elected validator set.
- [ ] Rewards and slashes are observable.
- [ ] Unbonding and withdrawal work after the configured delay.
