# Review Checklist

Use this checklist to review a learner's meta transaction implementation before
accepting the module.

## Conceptual Accuracy

- [ ] The submission separates signer, relayer, fee payer, and dispatch origin.
- [ ] It explains why relayer-paid transactions are useful for onboarding.
- [ ] It explains why meta transactions are hard to implement securely.
- [ ] It distinguishes a pallet-based prototype from a transaction extension
      design.
- [ ] It mentions the general transaction path as an advanced option, not as a
      required first step.
- [ ] It explains replay protection and nonce storage economics.
- [ ] It explains same-chain and cross-chain replay risks.

## Payload Review

- [ ] The signed payload includes the signer.
- [ ] The signed payload includes the inner call.
- [ ] The signed payload includes a nonce or equivalent replay guard.
- [ ] The signed payload includes expiry.
- [ ] The signed payload includes genesis hash or equivalent chain context.
- [ ] The signed payload includes runtime version context.
- [ ] The signed payload optionally restricts the relayer.
- [ ] The payload encoding is deterministic.

## Runtime Safety

- [ ] The runtime verifies the user's signature on-chain.
- [ ] The runtime rejects invalid signatures.
- [ ] The runtime rejects wrong relayers.
- [ ] The runtime rejects replay attempts.
- [ ] The runtime rejects expired payloads.
- [ ] The runtime rejects wrong-chain payloads.
- [ ] The runtime filters relayable calls.
- [ ] Sensitive calls are blocked by default.
- [ ] The relayer pays the outer fee through normal transaction payment.
- [ ] Storage growth from nonce tracking is economically bounded or clearly
      documented as a tutorial limitation.

## Dispatch Behavior

- [ ] The inner call dispatches as the user, not as the relayer.
- [ ] The implementation documents when the nonce increments.
- [ ] A failing inner call cannot be replayed accidentally unless retries are
      deliberately supported.
- [ ] Events include enough information to audit the relayed action.
- [ ] Large payload data is not emitted unnecessarily.

## Tests

- [ ] Valid meta transaction succeeds.
- [ ] The relayer-paid fee path is verified.
- [ ] The inner call origin is verified.
- [ ] Invalid signature is tested.
- [ ] Wrong relayer is tested.
- [ ] Wrong nonce is tested.
- [ ] Replay is tested.
- [ ] Expiry is tested.
- [ ] Wrong genesis hash is tested.
- [ ] Wrong runtime version is tested.
- [ ] Blocked call is tested.
- [ ] Inner call failure behavior is tested.

## Documentation Quality

- [ ] The implementation can be followed from a clean template.
- [ ] Code snippets are clearly marked as release-dependent shapes when they are
      not exact drop-in code.
- [ ] The relayer flow is written step by step.
- [ ] The local runbook proves both fee payer and dispatch origin.
- [ ] The module links to Polkadot SDK or Polkadot developer references.
- [ ] Common mistakes are documented.
- [ ] Production caveats are not hidden.

## Acceptance Criteria

- [ ] The runtime compiles.
- [ ] The tests pass.
- [ ] A local node can run.
- [ ] A user can sign a payload.
- [ ] A relayer can submit the outer transaction.
- [ ] The relayer pays the outer fee.
- [ ] The inner call dispatches as the user.
- [ ] Replaying the same payload fails.
