# Implementation Checklist

Use this checklist while building the OpenGov course chain. It is intentionally
ordered from runtime design to local verification.

## Design

- [ ] Write the purpose of the chain's OpenGov module.
- [ ] Choose the proposal types that learners will test.
- [ ] Define the account roles used in tests.
- [ ] Choose a block time assumption for converting minutes or days to blocks.
- [ ] Define at least three tracks: root, treasurer, and whitelisted caller.
- [ ] Document each track's origin, capacity, deposit, periods, and threshold
      curves.
- [ ] Mark which calls each origin is allowed to dispatch.
- [ ] Explain why low-risk and high-risk calls use different tracks.

## Dependencies

- [ ] Add `pallet_referenda`.
- [ ] Add `pallet_conviction_voting`.
- [ ] Add `pallet_preimage`.
- [ ] Add `pallet_scheduler`.
- [ ] Add `pallet_utility` if batch calls are part of the tutorial.
- [ ] Add `pallet_whitelist` if the whitelisted caller track is implemented.
- [ ] Keep every dependency in the same Polkadot SDK release family.
- [ ] Confirm that `std` features are enabled for native tests.

## Origins and Tracks

- [ ] Define the runtime origin type or generated origins used by OpenGov.
- [ ] Assign a stable track ID to every origin.
- [ ] Implement or configure `TracksInfo`.
- [ ] Set low local-test periods so tests finish quickly.
- [ ] Add comments warning that tutorial economics are not production settings.
- [ ] Test every origin-to-track mapping.
- [ ] Test that a wrong origin cannot use a privileged track.

## Preimage

- [ ] Configure preimage deposits.
- [ ] Wire `pallet_preimage` into `pallet_referenda`.
- [ ] Test noting a preimage.
- [ ] Test submitting a referendum that references the preimage.
- [ ] Test failure when the preimage hash is missing or wrong.
- [ ] Test deposit release or cleanup after the lifecycle ends.

## Scheduler

- [ ] Configure the scheduler with the runtime call type.
- [ ] Set maximum scheduled calls per block.
- [ ] Set bounded scheduler weight.
- [ ] Connect referenda to the scheduler.
- [ ] Test that an approved referendum schedules the call.
- [ ] Test that the call executes after the enactment period.
- [ ] Test that rejected referenda do not schedule calls.

## Referenda

- [ ] Configure `RuntimeCall`.
- [ ] Configure `RuntimeEvent`.
- [ ] Configure `Scheduler`.
- [ ] Configure `Currency`.
- [ ] Configure `SubmitOrigin`.
- [ ] Configure `CancelOrigin`.
- [ ] Configure `KillOrigin`.
- [ ] Configure `Slash`.
- [ ] Configure `Votes`.
- [ ] Configure `Tally`.
- [ ] Configure `SubmissionDeposit`.
- [ ] Configure `MaxQueued`.
- [ ] Configure `UndecidingTimeout`.
- [ ] Configure `AlarmInterval`.
- [ ] Configure `Tracks`.
- [ ] Configure `Preimages`.
- [ ] Configure `BlockNumberProvider`.
- [ ] Add the pallet to the runtime construct with a unique pallet index.

## Conviction Voting

- [ ] Configure `Currency`.
- [ ] Configure `Polls = Referenda`.
- [ ] Configure `MaxTurnout`.
- [ ] Configure `MaxVotes`.
- [ ] Configure `VoteLockingPeriod`.
- [ ] Configure `BlockNumberProvider`.
- [ ] Configure `VotingHooks`.
- [ ] Add the pallet to the runtime construct with a unique pallet index.
- [ ] Test direct aye votes.
- [ ] Test direct nay votes.
- [ ] Test vote removal.
- [ ] Test balance locks.
- [ ] Test track-level delegation.
- [ ] Test undelegation.

## Runtime Construct

- [ ] Register `System`.
- [ ] Register `Balances`.
- [ ] Register `Scheduler`.
- [ ] Register `Preimage`.
- [ ] Register `Referenda`.
- [ ] Register `ConvictionVoting`.
- [ ] Register optional `Utility`, `Whitelist`, or `Treasury` pallets.
- [ ] Confirm every pallet index is unique.
- [ ] Avoid changing pallet indices after any public launch.

## Genesis and Local Node

- [ ] Fund proposer accounts.
- [ ] Fund voter accounts.
- [ ] Fund delegate accounts.
- [ ] Fund the treasury account if treasury proposals are tested.
- [ ] Add any required technical or whitelist actors.
- [ ] Build the node.
- [ ] Start a local chain.
- [ ] Submit one full referendum through CLI, UI, or script.
- [ ] Record the events that prove each lifecycle stage occurred.

## Validation

- [ ] Run `cargo fmt`.
- [ ] Run `cargo check` or `cargo build`.
- [ ] Run unit tests.
- [ ] Run any runtime benchmark or weight check required by the template.
- [ ] Run a local node smoke test.
- [ ] Capture logs showing preimage submission, referendum submission, voting,
      approval, scheduling, dispatch, and unlock.
