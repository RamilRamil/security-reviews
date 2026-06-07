# Missing trusted executor validation in `lz_receive_alert` and `lz_compose_alert`

**Likelihood**: High
**Impact**: Medium

## Submission
- [Link](https://code4rena.com/audits/2026-04-layerzero-stellar-endpoint/submissions?uid=Nt29hTqw5Mg)
- Verdict: Invalid 
- Judging Rationate: High -> Low
"The endpoint is designed to ensure that fees for an OApp's send() call are never undercharged, and any remaining balance is fully refunded. In theory, the endpoint should never hold any balance. Even if the endpoint does have a pre-existing balance, it does not harm the send() caller. The endpoint is designed not to hold any assets by nature, and there is no security impact to the endpoint itself."

## Description

* `lz_receive_alert` and `lz_compose_alert` are intended for executors to signal failed `lz_receive` / `lz_compose` delivery. The implementation enforces only that the `executor` parameter authorizes the invocation (`executor.require_auth()`), then validates amounts (and compose index bounds) and publishes `LzReceiveAlert` / `LzComposeAlert`.

* There is no check that `executor` is a protocol-approved executor (for example aligned with ULN default executor or another on-chain registry). Any Stellar account that signs the call with `executor` set to itself can emit a plausible alert. Identity is verified, not the executor role.

* This is not impersonation of a specific third-party executor address without its keys: the event’s `executor` field is always the signer. The issue is that **permissionless emission** breaks the assumption that on-chain alerts come only from LayerZero’s executor infrastructure, which weakens integrity for monitoring and incident response if those systems trust the event without off-chain filtering.

`lz_receive_alert`:

```rust
fn lz_receive_alert(
    env: &Env,
    executor: &Address,
    // ...
) {
    executor.require_auth();
    assert_with_error!(env, gas >= 0 && value >= 0, EndpointError::InvalidAmount);
@>  // no trusted-executor check
    LzReceiveAlert { /* ... */ }.publish(env);
}
```

`lz_compose_alert`:

```rust
fn lz_compose_alert(
    env: &Env,
    executor: &Address,
    // ...
) {
    executor.require_auth();
    assert_with_error!(env, gas >= 0 && value >= 0, EndpointError::InvalidAmount);
    assert_compose_index(env, index);
@>  // no trusted-executor check
    LzComposeAlert { /* ... */ }.publish(env);
}
```

**Affected files**: `contracts/protocol/stellar/contracts/endpoint-v2/src/endpoint_v2.rs` (`lz_receive_alert`), `contracts/protocol/stellar/contracts/endpoint-v2/src/messaging_composer.rs` (`lz_compose_alert`).

## Risk

**Likelihood**:

* Any account that can sign a Soroban transaction can call these entrypoints with `executor` equal to its own address and pass `require_auth()`. No governance action, stake, or allowlist is required on-chain.

* Filling `guid`, `origin`, `reason`, and other fields with realistic values is unrestricted beyond basic validation.

**Impact**:

* No direct fund loss and no corruption of core messaging state (verify / clear / nonce) from these functions alone.

* Event integrity for operational pipelines is degraded: false or noisy alerts, polluted monitoring, wasted incident response, and weaker forensic signal if consumers assume “executor in the event” implies a LayerZero-trusted worker without cross-checking.

* Severity of harm depends on how much off-chain infrastructure naively trusts these events as authoritative executor signals.

## Proof of Concept

**Behavior.** Pick any new Stellar account address `A` that is not registered anywhere as a LayerZero executor. Build a transaction that calls `lz_receive_alert` (or `lz_compose_alert`) on `EndpointV2` with the `executor` argument set to `A`, and have `A` authorize that call. The contract runs `executor.require_auth()` (passes), optional numeric checks (pass), then emits `LzReceiveAlert` / `LzComposeAlert` with `executor == A`. No storage is read to decide whether `A` is an approved executor.

**Minimal Soroban test harness (illustrative).** The same flow is exercised in the SDK test environment: mock authorization for a generated address, invoke the endpoint client, observe the alert event with that address in the `executor` field.

```rust
// Soroban Env::default() + deployed EndpointV2; no executor allowlist configured anywhere.
let spoof_executor = Address::generate(env);
let receiver = Address::generate(env);
let origin = Origin {
    src_eid: 2,
    sender: BytesN::from_array(env, &[1u8; 32]),
    nonce: 1,
};
let guid = BytesN::from_array(env, &[9u8; 32]);
let message = Bytes::from_array(env, b"payload");
let extra_data = Bytes::new(env);
let reason = Bytes::from_array(env, b"out of gas");

env.mock_auths(&[MockAuth {
    address: &spoof_executor,
    invoke: &MockAuthInvoke {
        contract: &endpoint.address,
        fn_name: "lz_receive_alert",
        args: (
            &spoof_executor,
            &origin,
            &receiver,
            &guid,
            &1_000_000i128,
            &0i128,
            &message,
            &extra_data,
            &reason,
        )
        .into_val(env),
        sub_invokes: &[],
    },
}]);

endpoint_client.lz_receive_alert(
    &spoof_executor,
    &origin,
    &receiver,
    &guid,
    &1_000_000,
    &0,
    &message,
    &extra_data,
    &reason,
);
// Transaction succeeds; LzReceiveAlert is published with executor == spoof_executor.
```

The compose path is analogous: same `spoof_executor`, call `lz_compose_alert` with matching `mock_auths` args shape, event `LzComposeAlert` includes `executor == spoof_executor`.

**What this proves.** The only access control is “`executor` signed this call.” There is no on-chain step that binds `executor` to protocol-approved worker addresses, so monitoring stacks that treat these events as coming from “the” executor without an off-chain allowlist can be fed arbitrary alerts.

## Recommended Mitigation

Introduce governance-controlled trusted executor flags and reject unauthorized callers after `require_auth()`.

```diff
  fn lz_receive_alert(
      env: &Env,
      executor: &Address,
      ...
  ) {
      executor.require_auth();
+     assert_with_error!(
+         env,
+         EndpointStorage::is_trusted_executor(env, executor),
+         EndpointError::UnauthorizedExecutor
+     );
      assert_with_error!(env, gas >= 0 && value >= 0, EndpointError::InvalidAmount);
      ...
  }
```

Same pattern for `lz_compose_alert`. Add storage (for example `TrustedExecutor { executor: Address } -> bool`), owner-only `set_trusted_executor(executor, enabled)`, a dedicated error such as `UnauthorizedExecutor`, and an admin event when the set changes. After deployment, verify that only allowlisted executors can emit alerts after `require_auth()`.


# `pay_messaging_fees` refunds the endpoint’s global balance, not the current sender’s contribution

**Likelihood**: Medium
**Impact**: High -> Low 

## Submission
- [Link](https://code4rena.com/audits/2026-04-layerzero-stellar-endpoint/submissions?uid=VhtzyveeJRQ)
- Verdict: Invalid 
- Judging Rationate: "this report and its duplicates do not show a valid path were funds would be sitting in the endpoint without being caused by user error, lz_send() is atomic and the excess fees are always refunded for each tx, so any stranded funds/fees is user error,"

## Description

* `send` quotes fees from the configured send library, then calls `pay_messaging_fees` with the returned recipient lists and the caller-chosen `refund_address`. Fee “supply” is not taken from an argument scoped to `sender`; it is derived from the **entire** native and (when `pay_in_zro` is true) ZRO balance of the endpoint contract.

* After paying fee recipients, **all remaining** native and (in the ZRO branch) ZRO balance is transferred to `refund_address`. Nothing ties that remainder to the address that authorized `send` or that transferred tokens in a prior step.

* If tokens belonging to another party are already sitting on the endpoint (pre-fund in an earlier transaction, mistaken transfer, operational residual, and so on), a later `send` by user `B` can route the leftover after recipients are paid to `B`’s `refund_address`, **including value sourced from user `A`’s balance**. The `sender` field only controls authorization for `send`, not which token units are treated as “theirs” for refund purposes.

Core pattern in `pay_messaging_fees`:

```rust
fn pay_messaging_fees(
    env: &Env,
    pay_in_zro: bool,
    native_fee_recipients: &Vec<FeeRecipient>,
    zro_fee_recipients: &Vec<FeeRecipient>,
    refund_address: &Address,
) -> MessagingFee {
    let this_contract = env.current_contract_address();
    // ...
    let native_token_client = TokenClient::new(env, &Self::native_token(env));
    let mut native_fee_supplied = native_token_client.balance(&this_contract);
@>  // full contract balance treated as "supplied" native
    // ... pay recipients from native_fee_supplied ...
    if native_fee_supplied > 0 {
        native_token_client.transfer(&this_contract, refund_address, &native_fee_supplied);
    }

    if pay_in_zro {
        let zro_client = TokenClient::new(env, &zro_addr);
        let mut zro_fee_supplied = zro_client.balance(&this_contract);
@>      // full contract ZRO balance treated as "supplied" ZRO
        // ... pay ZRO recipients ...
        if zro_fee_supplied > 0 {
            zro_client.transfer(&this_contract, refund_address, &zro_fee_supplied);
        }
    }
    fee_paid
}
```

**Affected**: `contracts/protocol/stellar/contracts/endpoint-v2/src/endpoint_v2.rs` — `pay_messaging_fees`, invoked from `send`.

## Risk

**Likelihood**:

* Exploitation requires a **non-trivial balance on the endpoint that the current `sender` did not just supply in an atomic “fund + send”** flow from the same user. Examples: two-step pre-funding, accidental transfer to the endpoint address, or leftover balance between transactions.

* Where OApps always move fees and call `send` in one transaction and no unrelated liquidity rests on the contract, practical exposure is lower. Where long-lived or third-party balances are realistic, the issue becomes easy to trigger.

**Impact**:

* When the precondition holds, the effect is **misappropriation of third-party tokens**: the refund leg sends someone else’s residual to an attacker-chosen `refund_address`.

* The intended invariant “refunds only reflect surplus **from this sender’s** payment” is broken on-chain.

## Proof of Concept

**Behavior (native).** Address `V` transfers 100 units of the native fee token to the endpoint (so the endpoint’s native balance is 100). Address `A` invokes `send` **without** adding any native tokens beforehand. The send library requests **10** native for fee recipients. The implementation uses balance(endpoint) = 100: it pays 10 to recipients and sends **90** to `A`’s `refund_address`. Those 90 units originated from `V`, not from `A`.

**Behavior (ZRO, `pay_in_zro = true`).** Same structure: `V` funds 100 ZRO on the endpoint; `A` calls `send` with `pay_in_zro = true` and does not deposit ZRO; if the library charges 10 ZRO, **90** ZRO is refunded to `A`’s `refund_address` from the pool that included `V`’s deposit.

**Illustrative sequence (native).**

```text
1. V: transfer 100 native -> endpoint (endpoint balance 100).
2. A: send(..., refund_address = R); library asks 10 native for recipients.
3. pay_messaging_fees: native_fee_supplied = balance(endpoint) = 100.
4. Pay 10 to fee recipients; native_fee_supplied = 90.
5. Refund 90 native to R. V’s intended use of 100 is partly captured by A.
```

**What this proves.** Refund is computed from **global** contract balance, not from an amount attributed to `sender`. No mempool-specific assumption is required: ordering matters only insofar as someone else’s tokens are on the contract before `A`’s `send`.

## Recommended Mitigation

**Option A (minimal):** Stop using total contract balance as implied “supply.” Inside the same `send` transaction, pull the quoted fee from `sender` (or from an explicit `max_fee` / transfer-in), track `supplied_native` and `supplied_zro` for **this call**, pay recipients up to that, refund only `supplied - paid` to `refund_address`.

```diff
  fn pay_messaging_fees(
      env: &Env,
+     supplied_native: i128,
+     supplied_zro: i128,
      pay_in_zro: bool,
      native_fee_recipients: &Vec<FeeRecipient>,
      zro_fee_recipients: &Vec<FeeRecipient>,
      refund_address: &Address,
  ) -> MessagingFee {
      let this_contract = env.current_contract_address();
-     let mut native_fee_supplied = native_token_client.balance(&this_contract);
+     let mut native_fee_supplied = supplied_native;
      // assert transferred-in amount matches supplied_native if needed
      // ... pay from native_fee_supplied, refund remainder ...
-     let mut zro_fee_supplied = zro_client.balance(&this_contract);
+     let mut zro_fee_supplied = supplied_zro;
```

Wire `supplied_*` from `send` after pulling tokens from `sender` (exact shape depends on API design).

**Option B (architectural):** Per-sender credit / escrow storage for native and ZRO; spend and refund only against the caller’s credited balance if two-step funding must remain supported.

Either approach restores: **refund to `refund_address` <= surplus attributed to the current messaging fee payment path**, not the entire vault balance.
