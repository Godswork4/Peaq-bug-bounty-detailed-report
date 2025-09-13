
# Peaq Network — Critical Bug Report

This report contains two critical vulnerabilities discovered in the Peaq repository.

---

## **Bug 1 — Dev / Sudo Keys Present in Deployment Config / Dev Chain-spec**

**Description**  
The repository contains development-chain configurations embedding a well-known Substrate dev account (`5GrwvaEF5z...` / `//Alice`) as Sudo/root in `deploy/peaq.yaml` and `dev_chain_spec.rs`. If these configurations are ever used in production, anyone knowing the dev key could perform privileged operations such as runtime upgrades (`set_code`), minting tokens, altering balances, or freezing the chain. Critical impact: unauthorized token minting, transaction/consensus manipulation, and permanent freezing.

**Affected Files**  
- `deploy/peaq.yaml`  
- `node/src/parachain/dev_chain_spec.rs`  
- `node/src/parachain/peaq_chain_spec.rs`  

**Exact Code Snippets**  
```yaml
# deploy/peaq.yaml
Sudo: 
  Key: 5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
````

```rust
// dev_chain_spec.rs
SudoConfig { key: Some(root_key) },
```

**Runnable PoC (local devnet only)**

1. Build and launch a local dev node:

```bash
cargo build --release -p node
./target/release/node --chain ./node/src/parachain/dev_chain_spec.json --dev
```

2. Confirm dev key exists:

```bash
grep -A3 "Sudo" deploy/peaq.yaml
```

3. Use `//Alice` to submit a sudo call (local devnet only):

```bash
sudo.sudo(system.remark("PoC - dev key works"))
```

4. Observe execution succeeds — the dev key can perform privileged actions.

**Impact**

* Arbitrary runtime upgrades
* Unauthorized token minting
* Chain freezing or consensus manipulation

**Suggested Fixes**

* Remove dev keys from production chain-specs
* Maintain separate dev vs production configs
* Use multisig/governance-controlled Sudo for production
* Add CI checks to detect dev keys in PRs

---

## **Bug 2 — EVM Precompile `assets-erc20` Dispatch May Allow Unauthorized Minting**

**Description**
The EVM precompile converts EVM callers into Substrate runtime `AccountId`s and dispatches `pallet_assets::mint`. If the asset admin is misconfigured or address mapping allows an EVM caller to act as admin, arbitrary token minting is possible. Critical impact: unauthorized token minting, economic damage, and potential theft.

**Affected Files**

* `precompiles/assets-erc20/src/lib.rs`
* `runtime/peaq/src/lib.rs` (pallet\_assets config)
* `pallets/address-unification/*`

**Runnable PoC (local devnet only)**

1. Launch local dev node with EVM + precompiles enabled:

```bash
cargo build --release -p node
./target/release/node --dev
```

2. Create a test asset with controlled admin:

```bash
# via runtime extrinsic (local devnet only)
create_asset(asset_id=100, admin=<runtime-mapped EVM account>)
```

3. Call precompile mint from an EVM account (local devnet only):

```js
// Using ethers.js or chopsticks connected to local dev node
const tx = await precompileContract.mint(assetId, beneficiary, amount);
await tx.wait();
```

4. Observe that the token balance increases for the test asset.

**Impact**

* Unauthorized token minting
* Economic damage / potential theft

**Suggested Fixes**

* Harden `pallet_assets` permission checks
* Secure address mapping for EVM → runtime accounts
* Add automated tests preventing unauthorized minting

---

**Note:** All PoC steps must only be executed on **local/private devnets**. **Never run on mainnet.**
