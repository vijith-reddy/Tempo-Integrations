# TIP20 Migration FAQ

## Can a TIP20 token be deployed by extending `ITIP20` like an ERC20 contract?

No. `ITIP20` is an ABI/interface for Tempo's native TIP20 implementation. TIP20 tokens are created through `TIP20Factory` at deterministic native addresses, and their callable surface is the protocol/precompile surface.

Unlike a Solidity ERC20 implementation, a TIP20 token is not deployed as a per-token Solidity contract that an issuer can inherit from or override.

## Can custom methods be added directly to a TIP20 token?

No. Custom per-token methods cannot be added to the TIP20 token address itself.

If a method must appear on the token contract, that requires a protocol-level TIP20 extension. A separate Solidity contract can expose custom methods, but those methods live on the separate contract, not on the TIP20 token address.

## What are the TIP20 alternatives to `increaseAllowance` and `decreaseAllowance`?

TIP20/USDY does not expose OpenZeppelin-style atomic allowance delta helpers such as `increaseAllowance(spender, addedValue)` or `decreaseAllowance(spender, subtractedValue)`.

Use exact allowance-setting methods instead:

- To increase an allowance, read `allowance(owner, spender)` off-chain and call `approve(spender, currentAllowance + delta)`.
- To decrease an allowance, read `allowance(owner, spender)` off-chain and call `approve(spender, currentAllowance - delta)`, or call `approve(spender, 0)` to revoke it.
- For a safer reset flow, call `approve(spender, 0)` and then call `approve(spender, newAmount)`.
- For signature-based approval, use `permit(owner, spender, value, ...)` to set an exact allowance without requiring the owner to send the approval transaction.

These flows are client-side workarounds and do not provide the same atomic delta semantics as `increaseAllowance` or `decreaseAllowance`. If a token integration requires true atomic allowance increases or decreases on the TIP20 token address, that would require a protocol-level TIP20 extension.

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
