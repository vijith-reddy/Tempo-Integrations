# TIP20 Migration FAQ

## Does Tempo support Safe multisig accounts and hardware-key signers?

Yes. Safe support is live on Tempo through the [Safe app](https://app.safe.global/) and supports multisig hardware-key signers.

To create a new Safe, select **Create new Safe**, choose **Tempo**, add the signer addresses and confirmation threshold required by your security policy, then deploy it using pathUSD for gas. To use an existing Safe address, select **Add Safe accounts to the workspace** and add it on Tempo.

## Can a TIP20 token be deployed by extending `ITIP20` like an ERC20 contract?

No. `ITIP20` is an ABI/interface for Tempo's native TIP20 implementation. TIP20 tokens are created through `TIP20Factory` at deterministic native addresses, and their callable surface is the protocol/precompile surface.

Unlike a Solidity ERC20 implementation, a TIP20 token is not deployed as a per-token Solidity contract that an issuer can inherit from or override.

## Can custom methods be added directly to a TIP20 token?

No. Custom per-token methods cannot be added to the TIP20 token address itself.

If a method must appear on the token contract, that requires a protocol-level TIP20 extension. A separate Solidity contract can expose custom methods, but those methods live on the separate contract, not on the TIP20 token address.

## How should a rebasing ERC20 migrate to TIP20?

Do not use TIP20 rewards to recreate rebasing behavior. TIP20 rewards have been deprecated, and even before deprecation they accrued through opt-in reward accounting rather than making every holder's ERC20 `balanceOf` increase automatically.

For rebasing UX, use an rToken-style wrapper: a wrapper token around the yield-bearing TIP20 whose `balanceOf` is derived from internal shares multiplied by an exchange-rate or multiplier. Yield, donations, or issuer-side accrual update the multiplier, while transfers move shares. This preserves the user-facing pattern where balances can grow at the ERC20/TIP20 wrapper level without requiring a separate `claimRewards()` flow.

Migration shape:

- Snapshot the old rebasing ERC20 balances at a cutoff.
- Deploy or use a yield-bearing TIP20 as the underlying settlement asset.
- Deploy an rToken wrapper for that underlying asset.
- Mint wrapper shares so each holder's initial wrapper `balanceOf` matches their snapshotted rebasing balance at the current index.
- Route future yield/accrual into the wrapper multiplier rather than TIP20 rewards.

Reference implementation: [`tempoxyz/rtokens`](https://github.com/tempoxyz/rtokens), especially `src/RTokenWrapper.sol`.

## What are the TIP20 alternatives to `increaseAllowance` and `decreaseAllowance`?

TIP20/USDY does not expose OpenZeppelin-style atomic allowance delta helpers such as `increaseAllowance(spender, addedValue)` or `decreaseAllowance(spender, subtractedValue)`.

Use exact allowance-setting methods instead:

- To increase an allowance, read `allowance(owner, spender)` off-chain and call `approve(spender, currentAllowance + delta)`.
- To decrease an allowance, read `allowance(owner, spender)` off-chain and call `approve(spender, currentAllowance - delta)`, or call `approve(spender, 0)` to revoke it.
- For a safer reset flow, call `approve(spender, 0)` and then call `approve(spender, newAmount)`.
- For signature-based approval, use `permit(owner, spender, value, ...)` to set an exact allowance without requiring the owner to send the approval transaction.

These flows are client-side workarounds and do not provide the same atomic delta semantics as `increaseAllowance` or `decreaseAllowance`. If a token integration requires true atomic allowance increases or decreases on the TIP20 token address, that would require a protocol-level TIP20 extension.

## How does a TIP-403 whitelist apply to a delegated `transferFrom` spender?

For `transferFrom(from, to, amount)`, TIP-403 evaluates `from` as the sender and `to` as the recipient. It does not evaluate the delegated spender (`msg.sender`) that invokes `transferFrom`. The spender is separately authorized by the TIP20 allowance check.

Therefore, TIP-403 cannot block a sanctioned spender when both `from` and `to` are authorized by the token transfer policy. Token holders must avoid granting or must revoke allowances to such spenders, and integrations can enforce additional spender checks in their own contracts. Enforcing spender authorization globally for direct TIP20 `transferFrom` calls would require a protocol-level extension.

## Does TIP20 support roles?

Yes. TIP20 has native role-based access control through:

- `grantRole(bytes32 role, address account)`
- `revokeRole(bytes32 role, address account)`
- `renounceRole(bytes32 role)`
- `setRoleAdmin(bytes32 role, bytes32 adminRole)`

TIP20 also exposes built-in role constants such as:

- `ISSUER_ROLE`
- `PAUSE_ROLE`
- `UNPAUSE_ROLE`
- `BURN_BLOCKED_ROLE`

## What is the TIP20 alternative to OpenZeppelin role read helpers?

TIP20 does not expose OpenZeppelin-style role read helpers such as `hasRole(role, account)`, `getRoleAdmin(role)`, or enumerable role-member reads in the public interface.

For off-chain reads, index TIP20 role events:

- `RoleMembershipUpdated(role, account, sender, hasRole)` for current role membership.
- `RoleAdminUpdated(role, newAdminRole, sender)` for role admin changes.

For contract logic, do not read TIP20 role state directly. Call the permissioned TIP20 function and let the precompile enforce authorization. If the caller lacks the required role, the call reverts with `Unauthorized`.

## Can arbitrary custom roles be created on TIP20?

Custom `bytes32` roles can be granted through TIP20's RBAC functions and tracked from role events, but only protocol-defined roles affect native TIP20 behavior.

For example, a custom role like `TRANSFER_AGENT_ROLE` can exist as role data, but TIP20 will not automatically attach new native behavior to it unless the protocol implementation is extended or another contract explicitly checks that role.

## Can a manager or control-plane contract be the entrypoint for TIP20 workflows?

Yes. A separate Solidity manager contract can be the entrypoint for flows such as `subscribe`, `redeem`, issuance, redemption, pausing, or other admin workflows.

The manager contract can call TIP20 methods. For example, the TIP20 token can grant `ISSUER_ROLE` or `PAUSE_ROLE` to the manager contract, and the manager can expose business-specific methods that call `mint`, `burn`, `pause`, or other TIP20 functions.

## How should RBAC work for a generic manager contract?

Use two RBAC layers:

1. Manager-contract RBAC for business roles, such as `TRANSFER_AGENT_ROLE`, `SUBSCRIPTION_ADMIN_ROLE`, `REDEMPTION_ADMIN_ROLE`, or `COMPLIANCE_ADMIN_ROLE`.
2. TIP20 native RBAC for protocol roles granted to the manager contract, such as `ISSUER_ROLE`, `PAUSE_ROLE`, `UNPAUSE_ROLE`, or `BURN_BLOCKED_ROLE`.

In this model, users and operators interact with the manager contract. The TIP20 token grants native authority to the manager, not to every individual operator.

## Does a manager contract enforce custom logic on direct TIP20 transfers?

No. A manager contract only controls flows routed through the manager.

Current TIP20/precompile execution cannot call out to arbitrary EVM contracts for enforcement. That means a manager contract cannot be used as a callback hook that TIP20 invokes during every direct token transfer.

If transfer restrictions must apply to direct `transfer` or `transferFrom` calls, they must be expressible through native TIP20 or TIP-403 policy mechanisms, or require a protocol-level extension.

## What is the recommended migration pattern for ERC20 contracts with custom helper methods?

Use a manager/control-plane contract for workflow and business logic, and use TIP20 for the native token.

For example:

- Deploy the TIP20 through `TIP20Factory`.
- Deploy a Solidity manager contract.
- Grant the manager the needed TIP20 native roles.
- Define business roles on the manager contract.
- Route `subscribe`, `redeem`, issuance, redemption, and admin workflows through the manager.
- Use native TIP20/TIP-403 mechanisms for behavior that must be enforced on direct token transfers.

## When is a protocol-level TIP20 extension needed?

A protocol-level extension is needed when the desired behavior must:

- appear as a method on the TIP20 token address itself;
- be enforced inside every TIP20 transfer;
- change native mint, burn, pause, role, or transfer-policy semantics; or
- require TIP20/precompile code to call out to arbitrary EVM contract logic.

Manager contracts are useful for orchestration and issuer workflows, but they do not change TIP20's native callable surface.

## How does `BURN_BLOCKED_ROLE` work, and why can `burnBlocked` fail?

An authorized operator calls `burnBlocked(address from, uint256 amount)` on the TIP20 token. This is distinct from the ordinary `burn` method: the caller must hold `BURN_BLOCKED_ROLE`, and `from` must be unauthorized as a **sender** under the token's active TIP-403 transfer policy.

The target qualifies when it is included in a TIP-403 blocklist or excluded from a TIP-403 whitelist. With `ALLOW_ALL` or no restrictive policy, the target remains authorized as a sender and the call reverts with `PolicyForbids()`.

The token must also be unpaused, `from` must hold at least `amount`, and protected system custody addresses cannot be targeted. Depending on which prerequisite fails, the call can revert with `Unauthorized`, `ContractPaused`, `PolicyForbids`, `InsufficientBalance`, or `ProtectedAddress`.

## How should a TIP20 asset integrate with Chainlink CCIP?

Use the standard TIP20 implementation with Chainlink's existing CCIP token pool. The integration does not require changes to the TIP20 contract:

1. Confirm the asset, initial lane, and canonical source chain, such as Solana to Tempo or Ethereum to Tempo.
2. Confirm the bridge model. For a lock-and-mint lane, the pool locks the canonical asset on the source chain and mints its TIP20 representation on Tempo.
3. For a TIP20 token deployment, the Tempo team must confirm the TIP20 `currency` field with the Chainlink team before deployment.
4. Deploy the token through `TIP20Factory`.
5. Configure the audited CCIP token pool and grant it `ISSUER_ROLE` on the TIP20. This role lets the pool mint and burn tokens for cross-chain transfers.
6. Review audit coverage. Tempo can provide chain and TIP20 audits, and the CCIP implementer can provide token-pool audits. The issuer decides whether its policies require an additional integration-specific review.
7. Choose a devnet deployment or approve a direct mainnet deployment.
8. Configure transfer capacity and its refill period. A starting point proposed for PRIME was capacity equal to 1–2% of supply with a one-hour refill for a high-volume asset. A four-hour refill is also common. These are configurable operating parameters, not protocol requirements.
9. Provide issuer-controlled multisig addresses for each relevant chain so token-pool and CCT ownership can be transferred after deployment.
10. Deploy the lane, test transfers in both directions, verify roles and rate limits, and complete the ownership handoff.

The Figure PRIME integration selected Solana to Tempo with lock-and-mint because PRIME had deeper liquidity on Solana. Figure approved a direct mainnet deployment. At the cited supply of 156,221,832 PRIME, the proposed 1–2% transfer capacity was approximately 1.56 million–3.12 million tokens.

## What does the TIP20 `currency` field mean?

`currency` is an immutable protocol classification and eligibility tag. It is not a price feed and does not select a trading pair.

The current protocol gives the exact value `USD` special behavior: USD-classified TIP20s can be selected as user and validator fee tokens, participate in the Fee AMM, and form pairs in the native Stablecoin DEX. Other currency strings currently have no special protocol behavior.

Although integrations should use meaningful denomination identifiers, the current implementation accepts arbitrary currency strings rather than enforcing ISO 4217 codes.

## What is the difference between `currency` and `quoteToken`?

`currency` classifies the unit of account that one token unit is intended to track approximately 1:1. It is immutable after deployment.

`quoteToken` is the mutable liquidity and routing edge for the token. It identifies the token's direct pair in the native DEX and links it into the quote-token tree rooted at `pathUSD`. The DEX uses that tree to find direct and multi-hop routes, while the Fee AMM can use `quoteToken` as an intermediate route when direct liquidity is unavailable.

For USD-classified tokens, the selected quote token must also be USD-classified.

## How should an integration choose `currency`?

Choose the asset or unit of account that one token unit remains approximately 1:1 with:

- A USD stablecoin that stays near $1: `USD`.
- A rebasing USD yield token whose balance grows while each unit stays near $1: `USD`.
- A 1:1 wrapped BTC token such as WBTC or cbBTC: `BTC`.
- An accumulating or share-price token whose unit redemption value rises over time: use its own asset identifier, such as `PRIME` or `cbETH`, rather than `USD` or `ETH`.
- A non-rebasing wrapper around a rebasing asset generally becomes an accumulating token: its balance stays fixed while its redemption value rises, so it should use its own asset identifier unless the wrapper itself remains near 1:1 with the underlying denomination.

The deciding question is unit-price behavior, not whether the asset generates yield.

## Why should an accumulating USD-backed token not use `currency = USD`?

Classifying a token as `USD` opts it into infrastructure designed for near-par USD assets. In particular, the Fee AMM converts between a user's fee token and a validator's preferred fee token at a fixed near-par rate. If one unit of an accumulating token is worth more than $1, treating it as USD would economically misprice that conversion.

The native Stablecoin DEX is price-aware, but it is restricted to USD-classified pairs and a bounded near-par price range. An accumulating asset can eventually move outside those assumptions.

## Can transaction fees be paid with non-USD TIP20s?

No. The current Fee Manager requires both user-selected and validator-selected fee tokens to have `currency == "USD"`.

A user may pay fees in one USD-classified TIP20 while the validator receives another, such as USDC. The Fee AMM handles that conversion directly or, when necessary, through the user's `quoteToken`. This is why the `USD` classification must be reserved for assets that remain near dollar parity.
