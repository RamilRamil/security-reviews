# Submission: atomWarden=0 DoS for Unclaimed AtomWallets

## Severity: Medium

## What to Emphasize in the Report

### 1. Impact on In-Scope Code (AtomWallet)

- The vulnerability **manifests in AtomWallet** (in-scope): `owner()` returns `address(0)` for unclaimed wallets when `atomWarden` is set to zero.
- `execute()` checks `msg.sender == owner() || msg.sender == entryPoint()` — with `owner() == address(0)`, no one can execute.
- `address(0)` cannot send transactions; funds become **permanently stuck**.
- AtomWallet is explicitly in scope; the finding demonstrates direct impact on its behavior.

### 2. Root Cause (Context, Not Focus)

- `setWalletConfig` in MultiVault does not validate `atomWarden != address(0)`.
- MultiVault is out of scope, but the **effect** is in AtomWallet — frame the report around AtomWallet impact.

### 3. Misconfiguration, Not Just Admin Malice

- `address(0)` is a technically valid value but logically invalid for `atomWarden`.
- A single mistaken config (e.g. typo, wrong variable, migration error) leads to **irreversible loss** of funds.
- Adding `require(atomWarden != address(0))` is a standard safeguard against misconfiguration.

### 4. Trusted Roles Context

- MultiVault.DEFAULT_ADMIN_ROLE is not listed in the protocol's trusted roles overview.
- This highlights that the config path is under-documented and lacks basic validation.

### 5. Mitigation

- In `setWalletConfig`: `require(_walletConfig.atomWarden != address(0), "atomWarden cannot be zero");`
- Consider similar checks for other critical addresses (entryPoint, beacon, factory).

---

## Risk of Downgrade

- Judge may treat this as **admin/centralization risk** and downgrade to Low/QA.
- To reduce this risk:
  - Stress that **one misconfiguration** causes **permanent loss** with no recovery.
  - Emphasize that validation is a **defense-in-depth** measure, not a restriction on admin power.
  - Keep the narrative focused on AtomWallet (in-scope) impact.

---

## PoC Checklist

- [x] PoC uses the provided test suite (BaseTest, PoCCore.t.sol)
- [x] Test name: `test_submissionValidity`
- [x] Run: `forge test --match-contract PoCCore --match-test submissionValidity`
- [x] PoC must run successfully for submission to be valid
