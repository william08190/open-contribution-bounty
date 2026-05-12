# Implementation Checklist

Use this checklist while building the meta transaction course chain.

## Design

- [ ] Define the user story for gasless or relayer-paid transactions.
- [ ] Identify the signer, relayer, fee payer, and dispatch origin.
- [ ] Choose the initial scope: one user signature, one relayer, one inner call.
- [ ] Decide whether the first implementation is pallet-based or transaction
      extension-based.
- [ ] Define which calls are relayable.
- [ ] Define who economically pays for replay-guard storage.
- [ ] Define whether failing inner calls consume the meta nonce.
- [ ] Define whether payloads can be submitted by any relayer or only one named
      relayer.

## Dependencies

- [ ] Add a custom `pallet_meta_tx` or equivalent module.
- [ ] Confirm `frame_system` is available.
- [ ] Confirm `pallet_balances` is available.
- [ ] Confirm `pallet_transaction_payment` is available.
- [ ] Confirm the runtime has a signature type.
- [ ] Confirm the runtime has a runtime call type.
- [ ] Keep every dependency in the same Polkadot SDK release family.

## Payload

- [ ] Include signer account.
- [ ] Include optional relayer account.
- [ ] Include inner runtime call.
- [ ] Include meta nonce.
- [ ] Include expiry block.
- [ ] Include genesis hash.
- [ ] Include spec version.
- [ ] Include transaction version.
- [ ] Keep payload encoding deterministic.
- [ ] Document every payload field.

## Signature Verification

- [ ] Build signed bytes from the full payload.
- [ ] Verify with the runtime signature type.
- [ ] Convert the signer consistently with the chain account ID type.
- [ ] Reject invalid signatures.
- [ ] Test every supported signature scheme, or document which scheme is covered
      by the course.

## Replay Protection

- [ ] Add pallet-local nonce storage or an equivalent replay guard.
- [ ] Check `payload.nonce` against the expected nonce.
- [ ] Increment nonce after an accepted payload according to the documented
      policy.
- [ ] Reject repeated payloads.
- [ ] Reject stale payloads.
- [ ] Document the storage deposit or anti-spam policy.

## Context Checks

- [ ] Reject wrong genesis hash.
- [ ] Reject wrong spec version.
- [ ] Reject wrong transaction version.
- [ ] Reject expired payloads.
- [ ] Test that signatures from another chain context fail.

## Relayer Policy

- [ ] Charge the outer fee to the relayer through the normal signed transaction
      path.
- [ ] Reject submissions from a relayer that does not match
      `payload.relayer = Some(account)`.
- [ ] Allow any relayer only when `payload.relayer = None`.
- [ ] Document off-chain reimbursement options.
- [ ] Document atomic reimbursement options.

## Call Filtering

- [ ] Add a runtime-configured relayable call filter.
- [ ] Permit a harmless demo call.
- [ ] Permit a balances or application call if needed.
- [ ] Block sensitive calls by default.
- [ ] Test that blocked calls fail before dispatch.

## Dispatch

- [ ] Dispatch the inner call with user origin.
- [ ] Return or emit the inner dispatch result.
- [ ] Emit signer, relayer, nonce, and result.
- [ ] Avoid emitting large or private payload data.
- [ ] Benchmark or bound dispatch weight before production use.

## Tests

- [ ] Valid payload succeeds.
- [ ] Relayer pays the outer transaction fee.
- [ ] Inner call executes as the user.
- [ ] Invalid signature fails.
- [ ] Wrong relayer fails.
- [ ] Wrong nonce fails.
- [ ] Replay fails.
- [ ] Expired payload fails.
- [ ] Wrong genesis hash fails.
- [ ] Wrong spec version fails.
- [ ] Wrong transaction version fails.
- [ ] Blocked call fails.
- [ ] Failing inner call follows the documented nonce policy.
- [ ] Event fields are correct.

## Local Runbook

- [ ] Start a local chain.
- [ ] Build an inner call.
- [ ] Build the payload.
- [ ] Sign payload with Alice.
- [ ] Submit with Bob as relayer.
- [ ] Verify Bob paid the fee.
- [ ] Verify the inner call executed as Alice.
- [ ] Submit the same payload again and verify replay rejection.
- [ ] Capture terminal logs, events, or screenshots.
