# Review Checklist

Use this checklist to review a learner's OpenGov implementation before accepting
the module.

## Conceptual Accuracy

- [ ] The submission explains OpenGov as public referenda plus origin-specific
      tracks.
- [ ] It distinguishes `pallet_referenda` from `pallet_conviction_voting`.
- [ ] It does not present `pallet_democracy` as the OpenGov path.
- [ ] It explains approval and support separately.
- [ ] It explains conviction locks and their consequences.
- [ ] It explains track-level delegation.
- [ ] It explains why preimages are needed.
- [ ] It explains why approved referenda are scheduled for later enactment.

## Runtime Safety

- [ ] High-privilege calls are only available through high-privilege origins.
- [ ] Low-risk tracks cannot dispatch root-level calls.
- [ ] Track capacity is limited.
- [ ] Decision deposits are not trivially cheap.
- [ ] Root-like tracks have stricter thresholds than routine tracks.
- [ ] Vote locks are not shorter than the relevant enactment period.
- [ ] Scheduler weight limits are configured.
- [ ] Pallet indices are unique.
- [ ] The implementation does not copy production network parameters without
      explaining why they fit the new chain.

## Functional Coverage

- [ ] A learner can submit a preimage.
- [ ] A learner can submit a referendum on the intended track.
- [ ] A learner can place the decision deposit.
- [ ] A learner can vote aye with conviction.
- [ ] A learner can vote nay with conviction.
- [ ] A learner can delegate by track.
- [ ] A learner can undelegate.
- [ ] An approved referendum schedules a call.
- [ ] A scheduled call executes after the enactment period.
- [ ] A rejected referendum does not execute.
- [ ] A voter can remove votes and unlock when allowed.

## Tests

- [ ] Tests cover valid origin-to-track mapping.
- [ ] Tests cover invalid origin-to-track mapping.
- [ ] Tests cover preimage submission.
- [ ] Tests cover missing preimage failure.
- [ ] Tests cover decision deposit requirement.
- [ ] Tests cover support and approval behavior.
- [ ] Tests cover conviction-weighted voting.
- [ ] Tests cover delegation isolation by track.
- [ ] Tests cover lock enforcement.
- [ ] Tests cover scheduled enactment.
- [ ] Tests cover rejection.
- [ ] Tests cover cancel or kill behavior if those origins are exposed.

## Documentation Quality

- [ ] The course can be followed from a clean template.
- [ ] The proposal flow is written step by step.
- [ ] Code snippets are clearly marked as release-dependent shapes when they are
      not exact drop-in code.
- [ ] The document links to official Polkadot or Polkadot SDK references.
- [ ] The document names common mistakes and how to avoid them.
- [ ] The document includes learner deliverables.
- [ ] The final runbook produces visible local evidence.

## Acceptance Criteria

- [ ] The runtime compiles.
- [ ] The tests pass.
- [ ] A local node can run.
- [ ] At least one referendum is submitted, voted on, approved, enacted, and
      verified.
- [ ] The submitted documentation is accurate enough that a learner can reproduce
      the same flow.
