# Substrate OpenGov Course

This module shows how to add OpenGov-style governance to a
Substrate-based chain. The goal is not to copy Polkadot's production
parameters verbatim. The goal is to understand the runtime pieces that make
OpenGov work, then assemble a local chain with public referenda, origin-specific
tracks, conviction voting, delegation, deposits, and scheduled enactment.

OpenGov is built around a simple rule: token holders vote on whether a proposal
should be dispatched from a specific origin. The origin selects a track, and the
track decides the deposit, timing, capacity, support curve, approval curve, and
minimum enactment period for that class of proposal.

## Learning Goals

After completing this module, a learner should be able to:

- Explain why OpenGov uses `pallet_referenda` and `pallet_conviction_voting`
  instead of the older `pallet_democracy` flow.
- Add the OpenGov pallet set to a Substrate runtime.
- Define a small set of governance origins and tracks for a local chain.
- Connect referenda to conviction voting so accounts can vote directly or
  delegate by track.
- Store proposal preimages before referenda are submitted.
- Require both submission and decision deposits.
- Schedule accepted proposals for enactment after the track's minimum enactment
  period.
- Write tests for proposal submission, voting, confirmation, enactment, and
  unlocking.

## Mental Model

OpenGov is a pipeline with separate responsibilities:

```text
proposal call
    |
    v
pallet_preimage
stores the call bytes or call hash
    |
    v
pallet_referenda
tracks proposal lifecycle, deposits, origins, tracks, and enactment
    |
    v
pallet_conviction_voting
records direct votes and delegation, then reports a tally to referenda
    |
    v
pallet_scheduler
dispatches approved calls after the enactment period
```

`pallet_referenda` does not contain the vote casting logic. It runs referenda,
tracks deposits and phases, and asks a polling implementation for the vote tally.
`pallet_conviction_voting` is the voting side. It manages votes, vote locks,
delegation, and turnout.

The runtime is where the pieces become one governance system. The runtime
chooses which origins exist, which calls those origins can dispatch, how many
referenda can be deciding at the same time, and how strict each approval and
support curve should be.

## OpenGov Concepts

### Origin

An origin is a privilege level. A `Root` origin can dispatch highly sensitive
calls, while a smaller origin may only be allowed to spend a limited treasury
amount or update a parameter. Learners should treat origins as security
boundaries.

### Track

A track is the voting lane for one origin. Each track has its own parameters:

- decision deposit;
- prepare period;
- decision period;
- confirmation period;
- maximum number of deciding referenda;
- minimum enactment period;
- approval curve;
- support curve.

Small, low-risk calls can use short periods and low deposits. Runtime upgrades
or other privileged calls need slower, stricter tracks.

### Approval and Support

Approval measures aye vote weight against aye plus nay vote weight after
conviction is applied. Support measures aye votes before conviction against the
total possible voting power. A referendum must satisfy both requirements for the
confirmation period before it is approved.

### Conviction

Conviction lets a voter increase vote weight by accepting a longer lock. A vote
with stronger conviction has more weight, but the successful voter keeps funds
locked for longer after the referendum ends.

### Delegation

OpenGov supports track-level delegation. An account can delegate voting power on
one class of referenda while keeping direct control on another class. A learner
should test delegation and undelegation as first-class behavior, not as an
afterthought.

## Repository Starting Point

Start from a recent Polkadot SDK solo-chain or parachain template. File names
vary across SDK releases, but the work usually touches:

```text
runtime/
  Cargo.toml
  src/lib.rs
  src/configs/
  src/weights/
node/
  src/chain_spec.rs
```

This course assumes the runtime already has:

- `frame_system`;
- `pallet_balances`;
- `pallet_timestamp`;
- a block number provider;
- a scheduler-compatible runtime call type.

## Step 1: Add the Runtime Pallets

Add the OpenGov-related pallets to the runtime dependency set. In recent
templates that use the `polkadot-sdk` umbrella dependency, enable the matching
features there. In older templates, add the crates directly. Keep every pallet in
one SDK release family.

Minimum pallet set:

```text
pallet_referenda
pallet_conviction_voting
pallet_preimage
pallet_scheduler
```

Useful optional pallets:

```text
pallet_utility
pallet_whitelist
pallet_treasury
```

`pallet_utility` is useful because governance proposals often need batched
calls. `pallet_whitelist` is useful when a fellowship or technical body can mark
a proposal for a faster track. `pallet_treasury` gives learners a practical
non-root proposal type to test.

## Step 2: Decide the Local Governance Scope

Do not start by copying all Polkadot origins. A learning chain can start with
three tracks:

```text
Track 0: Root
  Purpose: runtime upgrades and privileged changes.
  Capacity: one deciding referendum.
  Deposit: high for the local token.
  Timing: long prepare, decision, and enactment periods.

Track 1: Treasurer
  Purpose: limited treasury spends.
  Capacity: several simultaneous decisions.
  Deposit: medium.
  Timing: shorter than root.

Track 2: Whitelisted Caller
  Purpose: fast execution for calls approved by a trusted technical origin.
  Capacity: many simultaneous decisions.
  Deposit: medium or high.
  Timing: short prepare, confirm, and enactment periods.
```

This structure gives learners meaningful contrast:

- a high-risk track with strict parameters;
- a routine spending track;
- an emergency or pre-reviewed track.

## Step 3: Define Governance Origins

A runtime needs concrete origins that map to track IDs. Exact code differs by SDK
version, but the shape is stable:

```rust
#[derive(
    Clone,
    Eq,
    PartialEq,
    Encode,
    Decode,
    RuntimeDebug,
    TypeInfo,
    MaxEncodedLen,
)]
pub enum OpenGovOrigin {
    Root,
    Treasurer,
    WhitelistedCaller,
}
```

Production runtimes normally use generated origin types and origin filters. The
important part for this course is that every track has a deliberate dispatch
origin. A referendum on the treasury track must not be able to dispatch the same
calls as the root track unless the runtime intentionally permits it.

## Step 4: Configure Tracks

The referenda pallet asks the runtime for `TracksInfo`. Tracks describe how each
origin should be processed. A local course can use simple constants:

```rust
pub const ROOT_TRACK: u16 = 0;
pub const TREASURER_TRACK: u16 = 1;
pub const WHITELIST_TRACK: u16 = 2;
```

Example track table:

```text
id  name                max deciding  decision deposit  prepare  decision
0   root                1             1_000 units       1 hour   7 days
1   treasurer           10            100 units         10 min   2 days
2   whitelisted-caller  50            500 units         1 min    1 day
```

Keep numbers small in local tests. Use blocks or constants that make tests run
quickly, then explain that production chains should set these values using real
economic and safety analysis.

The track also needs threshold curves. For a course, define:

- root requires high support and high approval for most of the decision period;
- treasurer gradually becomes easier as voters have more time to participate;
- whitelisted caller can use lower support but still requires strong approval.

The exact curve constructors move across SDK versions. If the current SDK exposes
`pallet_referenda::Curve`, use the SDK examples for linear or reciprocal curves
instead of hard-coding a private format.

## Step 5: Configure `pallet_preimage`

Governance proposals usually refer to a preimage. The preimage stores the call
bytes separately from the referendum record. This avoids stuffing large calls
directly into every submission and lets voters inspect exactly what will be
dispatched.

Configuration goals:

- charge a deposit based on the stored bytes;
- use the runtime's currency;
- make preimage storage available to `pallet_referenda`;
- test note, request, and unnote behavior.

Example shape:

```rust
impl pallet_preimage::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type WeightInfo = pallet_preimage::weights::SubstrateWeight<Runtime>;
    type Currency = Balances;
    type ManagerOrigin = EnsureRoot<AccountId>;
    type BaseDeposit = PreimageBaseDeposit;
    type ByteDeposit = PreimageByteDeposit;
}
```

## Step 6: Configure `pallet_scheduler`

Approved referenda are not dispatched immediately. The track's enactment period
gives the network time to prepare for the change. `pallet_scheduler` performs the
later dispatch.

Configuration goals:

- accept the runtime call type;
- allow scheduling from the origin used by referenda;
- set a realistic maximum number of scheduled calls per block;
- ensure scheduled calls use bounded weight.

Example shape:

```rust
impl pallet_scheduler::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type RuntimeCall = RuntimeCall;
    type RuntimeOrigin = RuntimeOrigin;
    type PalletsOrigin = OriginCaller;
    type MaximumWeight = MaximumSchedulerWeight;
    type ScheduleOrigin = EnsureRoot<AccountId>;
    type MaxScheduledPerBlock = ConstU32<50>;
    type WeightInfo = pallet_scheduler::weights::SubstrateWeight<Runtime>;
    type OriginPrivilegeCmp = EqualPrivilegeOnly;
    type Preimages = Preimage;
}
```

In a complete runtime, referenda must be wired to a scheduler type that can
schedule the approved call with the proposal's dispatch origin.

## Step 7: Configure `pallet_referenda`

`pallet_referenda` owns the referendum lifecycle:

- submitted;
- preparing;
- deciding;
- confirming;
- approved or rejected;
- scheduled for enactment;
- deposit refunded or slashed where applicable.

Example shape:

```rust
impl pallet_referenda::Config for Runtime {
    type RuntimeCall = RuntimeCall;
    type RuntimeEvent = RuntimeEvent;
    type Scheduler = Scheduler;
    type Currency = Balances;
    type SubmitOrigin = EnsureSigned<AccountId>;
    type CancelOrigin = EnsureRoot<AccountId>;
    type KillOrigin = EnsureRoot<AccountId>;
    type Slash = Treasury;
    type Votes = Balance;
    type Tally = pallet_conviction_voting::Tally<Balance>;
    type SubmissionDeposit = ReferendaSubmissionDeposit;
    type MaxQueued = ConstU32<100>;
    type UndecidingTimeout = UndecidingTimeout;
    type AlarmInterval = AlarmInterval;
    type Tracks = OpenGovTracks;
    type Preimages = Preimage;
    type BlockNumberProvider = System;
    type WeightInfo = pallet_referenda::weights::SubstrateWeight<Runtime>;
}
```

Review the SDK docs for the exact associated types in the release you use. The
important learning outcome is that `Tracks`, `Preimages`, `Scheduler`, `Currency`,
and `Tally` are explicit runtime decisions.

## Step 8: Configure `pallet_conviction_voting`

`pallet_conviction_voting` connects accounts and balances to referenda polls. It
tracks direct votes, delegated votes, turnout, and locks.

Example shape:

```rust
impl pallet_conviction_voting::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type WeightInfo = pallet_conviction_voting::weights::SubstrateWeight<Runtime>;
    type Currency = Balances;
    type Polls = Referenda;
    type MaxTurnout = TotalIssuanceOf<Balances, Runtime>;
    type MaxVotes = ConstU32<128>;
    type VoteLockingPeriod = VoteLockingPeriod;
    type BlockNumberProvider = System;
    type VotingHooks = ();
}
```

Key invariant: vote locks must cover the consequences of a successful vote. Do
not set the lock period shorter than the enactment period for the same class of
proposal.

## Step 9: Add Pallets to the Runtime Construct

Register each pallet with a unique pallet index. Example layout:

```rust
#[frame_support::runtime]
mod runtime {
    #[runtime::runtime]
    pub struct Runtime;

    #[runtime::pallet_index(0)]
    pub type System = frame_system;

    #[runtime::pallet_index(10)]
    pub type Balances = pallet_balances;

    #[runtime::pallet_index(20)]
    pub type Scheduler = pallet_scheduler;

    #[runtime::pallet_index(21)]
    pub type Preimage = pallet_preimage;

    #[runtime::pallet_index(30)]
    pub type Referenda = pallet_referenda;

    #[runtime::pallet_index(31)]
    pub type ConvictionVoting = pallet_conviction_voting;
}
```

Never reuse a pallet index. Changing a pallet index after launch is a storage
migration issue, so choose stable values before a public network starts.

## Step 10: Add Genesis State

For local tests, fund a few accounts and set balances high enough to exercise
conviction locks:

```text
Alice: proposer and aye voter
Bob: nay voter
Charlie: delegate
Dave: treasury or technical actor
```

If the course includes treasury proposals, also fund the treasury account. If it
includes a whitelisted track, add the membership or origin rules needed to call
the whitelist flow.

## Step 11: Submit a Referendum

A basic happy path is:

```text
1. Build a runtime call, such as setting a harmless storage value in a demo
   pallet or spending a small treasury amount.
2. Note the preimage.
3. Submit the referendum with the intended origin.
4. Place the decision deposit.
5. Cast direct aye and nay votes through conviction voting.
6. Advance blocks until the prepare period ends.
7. Continue until approval and support satisfy the track's curves for the
   confirmation period.
8. Advance to the enactment block.
9. Assert that the scheduled call executed.
10. Remove votes and unlock balances when allowed.
```

The same flow should fail when:

- the preimage hash does not match;
- the selected origin is not allowed to dispatch the call;
- no decision deposit is placed;
- the track is already at capacity;
- support or approval stays below the threshold;
- a voter tries to transfer locked funds before unlock.

## Step 12: Add Delegation

Track-level delegation should be part of the course, because it is one of the
features that makes OpenGov different from a single global voting queue.

Test scenarios:

- Alice delegates only the treasurer track to Charlie.
- Alice still votes directly on the root track.
- Charlie votes on a treasurer referendum and Alice's delegated balance counts.
- Alice undelegates and can vote directly on later treasurer referenda.
- Removing votes and unlocking works after the referendum lifecycle ends.

## Step 13: Add a Practical Proposal Type

A course is easier to understand when learners can observe a visible result.
Good local proposal types include:

- update a value in a demo configuration pallet;
- transfer a small amount from a treasury account;
- update a registered metadata value;
- schedule a harmless batch call through `pallet_utility`.

Avoid making the first exercise a runtime upgrade. Runtime upgrades are important
but bring extra complexity around Wasm building, preimage size, and operational
safety. Teach the OpenGov flow first, then add runtime upgrades as an advanced
exercise.

## Security Notes

- Do not let a low-privilege origin dispatch high-privilege calls.
- Do not make root proposals easy to pass in production.
- Do not set a decision deposit so low that the queue can be spammed.
- Do not allow unlimited simultaneous referenda on sensitive tracks.
- Do not shorten vote locks below the consequences of the vote.
- Do not ignore failed scheduled dispatches; surface the event and status.
- Do not use local tutorial constants as production economics.
- Do not use OpenGov pallets without benchmarking weights before a real launch.

## Suggested Tests

Unit tests:

- track lookup returns the expected track for each origin;
- invalid origin-to-track mapping fails;
- preimage deposit is reserved and released correctly;
- decision deposit is required before deciding;
- aye and nay conviction weights affect approval;
- support is based on turnout before conviction;
- delegation is isolated by track;
- locked funds cannot be transferred;
- approved calls are scheduled after the enactment period;
- rejected calls are not scheduled;
- cancel refunds deposits;
- kill slashes deposits according to the configured handler.

End-to-end local node checks:

- start the chain;
- submit a preimage through the UI or script;
- submit a referendum on the treasurer track;
- vote with two accounts using different conviction levels;
- observe status moving from preparing to deciding to confirming;
- wait for enactment;
- verify the target state changed;
- remove votes and unlock balances.

## Common Mistakes

### Using `pallet_democracy`

OpenGov referenda should use `pallet_referenda` with
`pallet_conviction_voting`. The older democracy pallet is not the OpenGov voting
path.

### Treating Tracks as Labels Only

Tracks are security and economics configuration. They control timing, deposits,
capacity, and thresholds. A track choice should be reviewed like a permission.

### Skipping Preimages

If voters cannot inspect the exact call, they cannot responsibly vote. Store and
reference preimages for real proposals.

### Forgetting the Scheduler

An approved referendum normally schedules a call for later enactment. If nothing
dispatches the call after approval, the governance pipeline is incomplete.

### Copying Production Parameters Blindly

Polkadot parameters are tuned for Polkadot's risk model and economics. A new
chain should start from its own token supply, security assumptions, block time,
and governance expectations.

## Learner Deliverables

At the end of the module, the learner should submit:

- a short design note listing origins, tracks, deposits, periods, and thresholds;
- runtime configuration for `Preimage`, `Scheduler`, `Referenda`, and
  `ConvictionVoting`;
- at least one visible proposal type;
- unit tests for voting, deposits, delegation, and enactment;
- a local runbook showing how to submit and vote on a referendum;
- screenshots or logs proving that an approved proposal executed.

## References

- Polkadot OpenGov overview:
  <https://wiki.polkadot.com/learn/learn-polkadot-opengov/>
- Polkadot OpenGov origins and tracks:
  <https://wiki.polkadot.com/learn/learn-polkadot-opengov-origins/>
- `pallet_referenda` Rust docs:
  <https://paritytech.github.io/polkadot-sdk/master/pallet_referenda/>
- `pallet_conviction_voting` Rust docs:
  <https://paritytech.github.io/polkadot-sdk/master/pallet_conviction_voting/>
- Add an existing pallet to a runtime:
  <https://docs.polkadot.com/parachains/customize-runtime/add-existing-pallets/>
