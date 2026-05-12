# Substrate Meta Transaction Course

This module shows how to design a Substrate-based chain where one account can
authorize a runtime call while another account pays the transaction fee and
submits the transaction. This pattern is usually called a meta transaction.

The common user story is simple: Alice wants to use a dApp, but Alice does not
hold the native fee token yet. A relayer account pays the network fee, the
runtime verifies Alice's authorization, and the inner call is dispatched as
Alice. The hard part is making that flow safe against replay, cross-chain reuse,
front-running, invalid signatures, and relayer abuse.

## Learning Goals

After completing this module, a learner should be able to:

- Explain the difference between the signer, relayer, fee payer, and dispatch
  origin.
- Explain why a normal signed transaction does not fully solve gasless onboarding.
- Design a bounded meta transaction payload that includes call data, nonce,
  mortality, genesis hash, spec version, transaction version, and relayer policy.
- Implement a small runtime pallet that verifies a user's signature and dispatches
  a call as that user.
- Add replay protection without assuming that the user account already holds the
  native token.
- Decide when to use a custom pallet, a transaction extension, or a general
  transaction format.
- Write tests for relayer-paid fees, invalid signatures, replay attempts,
  expired payloads, wrong-chain payloads, and blocked calls.

## Mental Model

The simplest meta transaction pipeline has four actors:

```text
user account
  signs the intent
      |
      v
relayer service
  checks business policy and submits an outer transaction
      |
      v
runtime meta transaction module
  verifies the user signature, replay guard, expiry, and call filter
      |
      v
inner runtime call
  dispatches with user origin while the relayer paid the outer fee
```

The outer transaction is signed by the relayer. The inner authorization is signed
by the user. A correct design keeps those two responsibilities separate:

- the relayer pays the fee for including the transaction;
- the user authorizes the state change;
- the runtime verifies the user authorization before dispatching the inner call.

## Scope for This Course

Meta transactions can cover many advanced cases, including multi-origin atomic
actions and cross-chain attestations. This course focuses on the most practical
starting point:

```text
one user signature + one relayer-paid outer transaction + one inner call
```

The finished learning chain should support:

- user signs a bounded payload off-chain;
- relayer submits `MetaTx::submit(payload, signature)`;
- runtime charges the relayer through the normal transaction payment path;
- runtime verifies the user signature and dispatches the inner call as the user;
- payload cannot be replayed on the same chain;
- payload cannot be replayed on another chain;
- payload expires after a bounded lifetime;
- only safe calls can be relayed.

Leave atomic multi-origin transactions and XCM message attestations as advanced
extensions after the core flow is correct.

## Repository Starting Point

Start from a recent Polkadot SDK solo-chain or parachain template. The work
usually touches:

```text
runtime/
  Cargo.toml
  src/lib.rs
pallets/
  meta-tx/
    Cargo.toml
    src/lib.rs
node/
  src/chain_spec.rs
```

This course assumes the runtime already has:

- `frame_system`;
- `pallet_balances`;
- `pallet_transaction_payment`;
- a runtime call type;
- an account ID and signature type, usually derived from `MultiSignature`.

## Design Choice: Pallet First

For a first course implementation, use a custom pallet. A pallet is easier to
teach, test, and audit than a custom transaction extension because learners can
see the full flow in one dispatchable call.

The pallet approach:

```text
outer signed extrinsic:
  relayer signs MetaTx::submit(payload, signature)

runtime:
  validates payload
  verifies user signature
  records replay guard
  dispatches inner call with user origin
```

Advanced production runtimes can move parts of the design into a transaction
extension or the general transaction format. That can reduce overhead and make
authorization part of the transaction pipeline, but it is harder to teach before
learners understand the security model.

## Step 1: Define the Payload

The signed payload must be explicit. Do not sign only the inner call bytes.
Context belongs in the signature so the same bytes cannot be replayed elsewhere.

Example shape:

```rust
#[derive(Clone, Encode, Decode, RuntimeDebug, TypeInfo, MaxEncodedLen)]
pub struct MetaTxPayload<AccountId, RuntimeCall, BlockNumber, Hash> {
    pub signer: AccountId,
    pub relayer: Option<AccountId>,
    pub call: Box<RuntimeCall>,
    pub nonce: u64,
    pub valid_until: BlockNumber,
    pub genesis_hash: Hash,
    pub spec_version: u32,
    pub transaction_version: u32,
}
```

Field purpose:

- `signer`: the account that authorizes the inner call;
- `relayer`: optional restriction to one relayer account;
- `call`: the inner runtime call;
- `nonce`: same-chain replay protection;
- `valid_until`: expiry for signatures that are never submitted;
- `genesis_hash`: cross-chain replay protection;
- `spec_version`: invalidates signatures across incompatible runtime upgrades;
- `transaction_version`: invalidates signatures across transaction format changes.

Learners can add `tip`, `max_fee`, `dapp_id`, or `domain_separator` later, but the
first implementation should stay small.

## Step 2: Add a Call Filter

Not every runtime call should be relayable. For example, calls that change keys,
proxy configuration, multisig state, or governance authority may need stronger
UX and review.

Define a runtime-configured filter:

```rust
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RuntimeCall: Parameter
        + Dispatchable<RuntimeOrigin = Self::RuntimeOrigin>
        + GetDispatchInfo
        + From<frame_system::Call<Self>>;
    type Signature: Verify<Signer = Self::Signer> + Parameter;
    type Signer: IdentifyAccount<AccountId = Self::AccountId>;
    type RelayableCall: Contains<Self::RuntimeCall>;
    type WeightInfo: WeightInfo;
}
```

Then reject blocked calls before dispatch:

```rust
ensure!(T::RelayableCall::contains(&payload.call), Error::<T>::CallNotRelayable);
```

The exact trait bounds depend on the SDK release and the runtime's call type.
Keep the course focused on the invariant: the runtime must choose the relayable
surface deliberately.

## Step 3: Verify the Signature

Build the signed bytes from SCALE-encoded payload data. A learner should use the
runtime's configured signature type and account conversion, not a hard-coded
cryptography library.

Example shape:

```rust
let encoded = payload.encode();
let signer = payload.signer.clone();

let verified = signature.verify(encoded.as_slice(), &signer);
ensure!(verified, Error::<T>::BadSignature);
```

In runtimes that use `MultiSignature`, verification should match the chain's
normal account ID derivation. Tests should cover at least one supported signature
scheme.

## Step 4: Add Replay Protection

Replay protection is the core difficulty. If Alice does not hold the native
token, Alice may not have an account provider or account nonce on-chain. A course
implementation can use pallet-local replay storage:

```rust
#[pallet::storage]
pub type NextMetaNonce<T: Config> =
    StorageMap<_, Blake2_128Concat, T::AccountId, u64, ValueQuery>;
```

Validation:

```text
payload.nonce == NextMetaNonce[payload.signer]
```

After a successful dispatch attempt:

```text
NextMetaNonce[payload.signer] += 1
```

There are tradeoffs:

- If nonce storage is paid by nobody, an attacker may create state for many
  accounts.
- If the relayer pays a storage deposit, the relayer needs a way to recover it
  or price it into service fees.
- If the user must pay existential deposit first, the design no longer gives
  fully gasless onboarding.

For a learning chain, keep the nonce map bounded and document who economically
pays for the storage. For production, combine deposits, rate limits, expiry, and
possibly dApp-specific relayer allowlists.

## Step 5: Add Expiry and Chain Context

Reject payloads that are too old:

```rust
ensure!(frame_system::Pallet::<T>::block_number() <= payload.valid_until, Error::<T>::Expired);
```

Reject payloads signed for another chain:

```rust
ensure!(
    payload.genesis_hash == frame_system::Pallet::<T>::block_hash(Zero::zero()),
    Error::<T>::WrongGenesis,
);
```

Also check the runtime versions against constants from the runtime version. The
exact access pattern varies by template, but the important rule is stable:
include version context in the signed message so old signatures do not survive
incompatible runtime changes.

## Step 6: Restrict the Relayer When Needed

If `payload.relayer` is `Some(account)`, require the outer transaction signer to
match that account. This prevents any relayer from taking a signed payload and
submitting it first.

```rust
let relayer = ensure_signed(origin)?;
if let Some(allowed) = &payload.relayer {
    ensure!(&relayer == allowed, Error::<T>::WrongRelayer);
}
```

If `payload.relayer` is `None`, any relayer may submit the payload. That can be
useful when several relayers compete, but the dApp should understand the
front-running and pricing consequences.

## Step 7: Dispatch the Inner Call

Once all checks pass, dispatch the inner call with the user as the origin:

```rust
let origin = frame_system::RawOrigin::Signed(payload.signer.clone()).into();
let result = payload.call.dispatch(origin);
```

Nonce handling must be deliberate. A common course policy is:

```text
increment nonce before dispatch
return the inner dispatch result
do not allow a failing inner call to preserve a reusable signature
```

This prevents the same signature from being replayed until the inner call happens
to succeed. If a dApp wants retryable payloads, it needs a different signed
policy that explicitly allows retries and bounds the risk.

## Step 8: Fees and Incentives

With the pallet-first approach, the outer transaction is a normal signed
transaction by the relayer. The runtime's normal transaction payment extension
charges the relayer.

The dApp can reimburse the relayer in several ways:

- off-chain business agreement;
- user pays the relayer in another token inside the inner call;
- dApp maintains a relayer treasury;
- relayer only submits payloads that pass its own profitability checks.

Do not make reimbursement implicit unless the runtime can enforce it atomically.
For example, if the inner call is a batch that first transfers a dApp token to
the relayer and then performs the user action, the relayer still needs to verify
that the batch cannot be reordered or partially executed against its interest.

## Step 9: Add Events and Errors

Events:

```rust
MetaTransactionDispatched {
    signer,
    relayer,
    nonce,
    dispatch_result,
}
```

Errors:

```text
BadSignature
WrongRelayer
WrongNonce
Expired
WrongGenesis
WrongSpecVersion
WrongTransactionVersion
CallNotRelayable
InnerCallFailed
```

Keep enough event data to debug relayer behavior, but do not emit private or
large payload data unnecessarily.

## Step 10: Add Tests

Minimum unit tests:

- relayer submits a valid payload and pays the outer fee;
- inner call dispatches as the user;
- invalid signature fails;
- wrong relayer fails;
- wrong nonce fails;
- repeated payload fails;
- expired payload fails;
- wrong genesis hash fails;
- wrong spec version fails;
- blocked call fails;
- failing inner call consumes the nonce if that is the chosen policy;
- nonce increments after accepted payloads;
- event includes signer, relayer, nonce, and result.

End-to-end local node checks:

- start a local chain;
- build a payload for a simple balances transfer or demo pallet call;
- sign the payload with Alice;
- submit the outer transaction with Bob as relayer;
- verify Bob paid the transaction fee;
- verify the call executed as Alice;
- resubmit the same payload and confirm replay rejection.

## Advanced Path: Transaction Extensions

Recent Polkadot SDK documentation describes transaction extensions and general
transactions. A transaction extension can participate in transaction validation
and authorization before the call is dispatched. This is closer to a production
meta transaction design because authorization can happen in the transaction
pipeline rather than inside a pallet call.

Use a transaction extension when:

- the authorization should be part of the transaction format;
- the chain needs custom transaction priority or validity tags;
- replay protection should be enforced before the transaction enters the pool;
- the runtime team is comfortable maintaining the extension across SDK upgrades.

Use a pallet first when:

- teaching the flow;
- prototyping dApp-specific relaying;
- keeping the call surface small;
- avoiding custom extrinsic format work.

## Security Notes

- Do not sign only call bytes. Sign domain and chain context.
- Do not omit replay protection.
- Do not let gasless users create unlimited on-chain nonce state for free.
- Do not allow every runtime call to be relayed by default.
- Do not preserve a reusable signature after a failing inner dispatch unless the
  signed payload explicitly allows retries.
- Do not let an arbitrary relayer steal a payload intended for a specific relayer.
- Do not ignore runtime upgrades; include spec and transaction versions.
- Do not rely only on off-chain relayer checks. Critical checks belong on-chain.
- Do not expose meta transactions without monitoring relayer failure and abuse
  rates.

## Common Mistakes

### Confusing Fee Payer With Dispatch Origin

The relayer pays the outer transaction fee. The inner call should still dispatch
as the user who signed the payload.

### Treating Unsigned Transactions as Free Meta Transactions

Unsigned transactions need strong custom validation because they have no normal
fee deterrent. A relayer-signed outer transaction is easier to reason about for
the first course implementation.

### Skipping Chain Context

If the payload does not include genesis hash and version context, a signature can
survive in places the signer did not intend.

### Reusing Account Nonce Without Understanding ED

The issue that motivated Polkadot SDK meta transaction discussion is that a user
with no tokens may not be able to maintain normal nonce storage safely. A course
must explain the storage economics instead of hiding them.

### Making the Relayer Trust the User

The relayer should simulate or inspect the inner call before submitting. If the
relayer expects reimbursement, it must verify that the reimbursement is enforced
by the transaction itself or by a trusted off-chain agreement.

## Learner Deliverables

At the end of the module, the learner should submit:

- a design note describing the signer, relayer, payload fields, replay guard, and
  call filter;
- a `pallet_meta_tx` or equivalent runtime module;
- runtime configuration that wires the pallet into the chain;
- tests for valid submission and each rejection path;
- a local runbook for signing a payload and relaying it;
- logs or screenshots proving that the relayer paid the outer fee and the inner
  call dispatched as the user.

## References

- Polkadot SDK meta transaction discussion:
  <https://github.com/paritytech/polkadot-sdk/issues/266>
- Polkadot transactions reference:
  <https://docs.polkadot.com/reference/parachains/blocks-transactions-fees/transactions/>
- Polkadot SDK extrinsic encoding reference:
  <https://paritytech.github.io/polkadot-sdk/master/polkadot_sdk_docs/reference_docs/extrinsic_encoding/>
- Transaction extension trait docs:
  <https://paritytech.github.io/polkadot-sdk/master/polkadot_sdk_frame/traits/transaction_extension/trait.TransactionExtension.html>
