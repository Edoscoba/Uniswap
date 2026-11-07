# Audit Engine Protocol Docs: Uniswap v4 Margin Trading

Scope: `src/MarginRouter.sol`, `src/MarginAccount.sol`, `src/MarginAccountFactory.sol`,
`src/MorphoLendingAdapter.sol`, `src/AaveLendingAdapter.sol`, `src/AaveV4LendingAdapter.sol`,
`src/CompoundV3LendingAdapter.sol`, `src/base/PositionAmountResolver.sol`,
`src/libraries/MarginActions.sol`, `src/libraries/MarginCalldataDecoder.sol`,
`src/libraries/TransientPlanState.sol`, `src/libraries/Units.sol`,
`src/types/Market.sol`, `src/types/Ltv.sol`, `src/types/Call.sol`, `src/types/PositionData.sol`,
`src/interfaces/ILendingAdapter.sol`, `src/interfaces/IMarginRouter.sol`,
`src/interfaces/IMarginAccount.sol`, `src/interfaces/IAmountResolver.sol`.

Branch `margin-trading`, HEAD `1c86348d` at the time of writing. This is the Audit Engine protocol
doc; the companion integration guide at `docs/margin-trading.md` covers the same system for
front-end and smart-contract integrators and is the source for most facts restated here. Where the
two disagree, the current source is authoritative and this document says so.

Referenced but out of scope: the vendored venue interfaces under `src/interfaces/external/` (Morpho
Blue, Aave v3, the Aave v4 Spoke, the Compound Comet, WETH9, the Universal Router), the shared
v4-periphery bases the router inherits from `src/base/` (`V4Router`, `BaseActionsRouter`,
`DeltaResolver`, `ReentrancyLock`, `Permit2Forwarder`, `Multicall_v4`, `NativeWrapper`), and `lib/`.

**Audit target vs. deployed bytecode.** The live mainnet suite (`docs/margin-trading.md` Section 10)
was deployed from commit `1e818e62`. HEAD differs from that commit only in `src/MarginRouter.sol` and
`src/interfaces/IMarginRouter.sol` (commits `89fddef7`, `508cc9c4`, `a8a60bd0`); the one behavioral
difference is that a zero `maxLtvAfter` on a partial decrease now reverts `LtvBoundRequired` where the
deployed router reverts `SlippageBoundRequired`. Every other statement in this document applies to
both.

## 1. Protocol Overview

### 1.1 Summary

The margin-trading suite is a Uniswap v4 periphery that lets a caller open a leveraged spot
position in one transaction by composing a v4 swap with a borrow/supply against an external
lending protocol (Morpho Blue, Aave v3, an Aave v4 Spoke, or a Compound v3 Comet). The caller
contributes equity, a `MarginRouter` flash-takes the debt token from the `PoolManager` inside one
`unlock`, a caller-supplied Universal Router swaps it into collateral, the caller's own
`MarginAccount` supplies that collateral and borrows the debt back to settle the pool, and the
position then lives entirely at the external venue under the `MarginAccount`'s own address. There
is no shared pool, no shares, no protocol-owned balance, and no index: the only state this suite
keeps is the venue's own position, read back through a stateless adapter, plus live balance deltas
measured inside one transaction. The venue (not this suite) prices, accrues interest on, and
liquidates every position.

This suite composes three existing primitives: v4's flash-accounting `unlock`/`take`/`settle` pattern
(from `v4-core`), Permit2 for equity pulls, and the Universal Router for the actual swap leg. The
"leverage" primitive itself is a straightforward borrow-swap-supply loop; the contribution of this
suite is wiring that loop safely through v4 flash accounting while keeping zero protocol-owned
balances and zero governance.

### 1.2 Architectural evolution relevant to auditors

The current permissionless, market-key ABI is a replacement for an earlier, governed design. This
matters directly for Section 5: the OpenZeppelin audit in that section was performed against the
earlier architecture, before refactor commits `f329e532` / `d3c915cf` / `c8146949`
("permissionless market keys decoded by the adapters").

- **Old model (audited by OpenZeppelin, commit** `e9b50f5`**, superseded routers still live onchain):**
`Market` carried no `data` field. Each adapter held an owned `MarketRegistry` with a `setMarket`
/ `allowed` surface, the router had a `transferGovernance`-style admin and an adapter allowlist,
and the audited adapters inherited an `OwnableAdapter`.
- **Current model:** `Market` carries `collateral`, `debt`, and an adapter-decoded `data` field
(Section 2.5). There is no registry, no `setMarket`, no router governance, and no adapter
allowlist anywhere in `src/`: the caller names the adapter and the market on every call, and the
adapter validates the key against the live venue on every call instead of against a stored
registration. Nothing in the suite is configurable after deployment (Section 4).

Consequence for this document: every finding in the OpenZeppelin report that concerns
`setMarket`/`MarketRegistry`/`transferGovernance`/`OwnableAdapter` describes code that no longer
exists. Findings about swap, router, or account mechanics unrelated to that registry (for example
the absolute-balance fill assertion) were re-checked against current HEAD for this document and
are called out in Section 6 as still open where that is the case. Four earlier router deployments predate the
permissionless model and remain live onchain but unintegrated-against: see `docs/margin-trading.md`
Section 10 for their addresses.

### 1.3 Actors

- **Position owner (**`MarginAccount.owner()`**)** - any EOA or contract that calls `increasePosition`,
`decreasePosition`, `addCollateral`, or `execute` on the router, or deploys an account directly
with `createAccount`. Untrusted by the protocol; the protocol's only guarantee is that the router
derives their account from their own authenticated address and `subId`, never from calldata. Goal:
leveraged exposure to a `(collateral, debt)` pair, or a hedged pair of exposures across sub-accounts
(`docs/margin-trading.md` Section 8.3).
- **Front end / integrating contract** - builds the Universal Router route (`routeCommands`,
`routeInputs`), the per-call `universalRouter` address, and the `Market.data` key on the owner's
behalf. Fully trusted by the owner who uses it, untrusted by the protocol: a malicious or buggy
integrator can only harm its own caller's account (Section 4.1).
- **Lending venue (Morpho Blue, Aave v3, an Aave v4 Spoke, a Compound v3 Comet)** - holds the actual
collateral and debt, prices the position, and liquidates it out of band. Trusted for correctness by
the adapter that targets it; the margin suite adds no independent oracle or liquidation logic
(Section 4.1, Section 2.4).
- **Liquidator / keeper (venue-side)** - not part of this suite. Liquidation happens entirely inside
the lending venue; the margin suite observes the result the next time it reads the position and
never triggers or blocks it.
- **Deployer EOA** - broadcasts `DeployMargin.s.sol`. Not a privileged role after deployment: every
contract is deployed through the canonical CREATE2 deployer at a deterministic address, no address
is baked in as an owner, and nothing is configurable afterwards (`script/DeployMargin.s.sol`
NatSpec). See Section 4.1.

### 1.4 Core User Flows

- **Open / increase a leveraged position** - entry point `MarginRouter.increasePosition`
(`IncreaseParams`). Touches: `MarginRouter` -> `PoolManager.unlock` -> Universal Router (caller-
supplied, routes the exact-output swap, which may itself touch v2/v3/v4 pools) -> `MarginAccount`
-> the chosen `ILendingAdapter` -> the lending venue -> `PoolManager.settle`. Synchronous, one
transaction, one `unlock`.
- **Add collateral (delever / improve health, no swap)** - entry point
`MarginRouter.addCollateral` (`AddCollateralParams`). Touches: `MarginRouter` -> `MarginAccount` ->
adapter -> venue. No `PoolManager` unlock.
- **Partial decrease (delever by repaying some debt)** - entry point
`MarginRouter.decreasePosition` with `debtToRepay < type(uint256).max`. Touches: `MarginRouter` ->
`PoolManager.unlock` -> Universal Router -> `MarginAccount` -> adapter -> venue -> `PoolManager.settle`.
Enforces the caller's mandatory `maxLtvAfter` bound.
- **Full close** - entry point `MarginRouter.decreasePosition` with `debtToRepay == type(uint256).max`. Same touch list as a partial decrease when debt is open; when the position is
already debt-free it skips the swap and the unlock entirely and withdraws collateral directly.
- **Arbitrary plan** - entry point `MarginRouter.execute(bytes unlockData, uint256 deadline)`.
Touches whatever the caller's packed opcode plan touches: any mix of v4 routing actions and the
margin opcodes (`SET_ACCOUNT`, `PULL_TO_ACCOUNT`, `ACCOUNT_SUPPLY_COLLATERAL`, `ACCOUNT_BORROW`,
`ACCOUNT_WITHDRAW_COLLATERAL`, `ACCOUNT_REPAY`, `ACCOUNT_SWEEP`, `ROUTE_SWAP`, `ASSERT_HEALTH`,
`ASSERT_ACCOUNT_BALANCE`) against the caller's own accounts only. See Section 2.3 and
`docs/margin-trading.md` Section 4.1 for the plan-composition contract; this is the path with the
fewest built-in guardrails.
- **Owner escape hatch** - entry point `MarginAccount.execute(adapter, callData)`, owner-only,
bypasses the router entirely and calls the adapter's `lendingProtocol()` directly as the account.
The guaranteed exit when a curated flow or the router is unavailable for any reason.

## 2. Detailed Breakdown by Component

Ordered from the component that directly custodies the account's authority (`MarginAccount`) through
orchestration (`MarginRouter`) to the venue-specific periphery (lending adapters) and finally the
shared value types.

### 2.1 MarginAccount

**Purpose.** `src/MarginAccount.sol` (214 lines). A minimal, soulbound per-`(owner, subId)` clone
(Solady clone-with-immutable-args) that is itself the lending counterparty: every position is held
under the account's own address (`onBehalf == account`), so the account needs no delegated
authorization from the owner at the venue. `owner()` and `manager()` are baked into the clone's
bytecode at deployment; there is no initializer and no transfer path (soulbound). The account's
fund-moving primitives, `supplyCollateral`, `withdrawCollateral`, `borrow`, `repay`, and `sweep`, are
callable only by the manager (always the deploying `MarginRouter`) or the owner, and every recipient
argument on those calls is constrained to the manager or the owner, so funds can never be redirected
to a third party no matter which adapter or calldata is supplied. `execute(adapter, adapterCall)` is the
owner-only escape hatch: it forwards an arbitrary call (with `msg.value`) to the named adapter's
`lendingProtocol()` as the account, bypassing the router and the router's `PositionUpdated` event
entirely; the account itself emits `Executed(caller, adapter, target)` (`src/MarginAccount.sol:142-152`).

**Mechanics.** Every fund-moving call takes an `ILendingAdapter` argument and an adapter-specific
amount/market; the account never interprets venue semantics itself. It calls the adapter's `encode*`
view to get a `Call[]` sequence (Section 2.5) and executes each entry, in order, against
`adapter.lendingProtocol()` with a regular `call` (never a `delegatecall`). `repay`'s max-sentinel
path approves the account's entire debt-token balance to the venue target before the call and resets
the approval afterward (`src/MarginAccount.sol:125-128`; Pashov lead, Section 6.2). `supplyCollateral`
approves exactly `amount` around its sequence and resets it likewise. A repay encoder that returns an
empty sequence (no debt at the venue) is skipped and reported as `repaid == 0` (`:119-122`), which is
what lets a generic repay-then-withdraw plan run against a debt-free position.

**Expected inputs and caller responsibilities.** The manager (router) is expected to derive the
account's address only from the authenticated caller and `subId`, never from calldata, and the
router upholds this on every curated and `execute` path. The owner is expected to use the escape
hatch only with calldata it has reviewed for the adapter's `lendingProtocol()`; the account performs
no validation of `adapterCall` beyond the owner-only gate on `execute` itself (`:147-148`).

**Edge cases and intentional behavior.**

- Mutations through the escape hatch bypass the router and emit no `PositionUpdated` snapshot or
delta event; an indexer built only on router logs will not see them (`docs/margin-trading.md`
Section 4.1 item 6).
- Any address, deployed or not, can receive a supply or repay "on behalf of" the deterministic
account from a venue that allows on-behalf actions (Morpho, Aave v3, Comet). This is intentional
venue behavior the account cannot prevent; see Section 6 for the concrete consequences on specific
venues.
- The account has an unguarded `receive()` (`:213`) and forwards each `Call.value` the adapter
encodes (`:208-210`), so it can hold native currency for a future native-venue adapter. Every in-tree
adapter encodes `value: 0`; a hostile adapter named by the owner could spend any native balance the
account holds, which stays inside the caller-scoped trust model of Section 4.1.

**Areas of concern.** Whether every recipient-constrained primitive remains constrained under a
hostile adapter's `Call[]` sequence (the account trusts the adapter's encoding to target only
`lendingProtocol()`, since `execCall`'s target comes from the adapter, not the caller); whether the
account's state (none beyond `owner`/`manager` immutables) can desync from the venue's in any way
that the account itself would need to reconcile (currently it holds no cached venue state, so this
class of bug is structurally absent here, but adapter-side reads can still be stale, see Section 2.4).

### 2.2 MarginAccountFactory

**Purpose.** `src/MarginAccountFactory.sol` (85 lines, abstract, inherited by `MarginRouter`). Derives
and deploys the deterministic `MarginAccount` clone for a given `(owner, subId)`. `accountOf(owner, subId)` is a view CREATE2 address computation and never deploys; `createAccount(owner, subId)`
deploys idempotently (a second call on an already-deployed account is a no-op that returns the
existing address, `src/MarginAccountFactory.sol:60-66`). Deployment is permissionless: anyone may
deploy anyone's account, but the owner is bound into both the salt and the immutable args, so a
third-party deploy cannot change who owns it. `increasePosition`, `addCollateral`, and the
`SET_ACCOUNT` opcode call `createAccount` before acting on the account; `decreasePosition` only
derives the address with `accountOf` (`src/MarginRouter.sol:137`) and never deploys, so a position
owner never needs to call it directly except to predict an address before funding it
(`docs/margin-trading.md` Section 6.2).

**Mechanics.** CREATE2 salt is `keccak256(abi.encode(owner, manager, subId))` (`:82-84`); the
clone's immutable args bake in the owner address and the manager address (always the `MarginRouter`
that deployed it, by construction, since the factory is inherited rather than a separate deployed
contract). Because the salt depends only on `(owner, manager, subId)` and not on `msg.sender`, the
predicted address is public and precomputable by anyone, including before deployment (Section 6).

**Expected inputs.** `subId` is caller-chosen and unconstrained (any `uint256`); it exists purely to
let one owner hold independent positions. The router does not enforce any relationship between
`subId` and the market opened under it; `docs/margin-trading.md` Section 3.2 documents that mixing
two Aave or Compound markets under one `subId` is a caller footgun, not a protocol-enforced
invariant.

**Edge cases and intentional behavior.** An account's address is deterministic and discoverable
before it is deployed; this is required for the "predict, then fund, then open" integration pattern
(Section 6.2 of the integration guide) but is also what makes the predictable-account griefing class
in Section 6 possible.

### 2.3 MarginRouter

**Purpose.** `src/MarginRouter.sol` (815 lines). The entry point and orchestrator. Inherits
`IMarginRouter`, `V4Router` (the v4 swap-routing base), `ReentrancyLock`, `Permit2Forwarder`,
`Multicall_v4`, `NativeWrapper`, and `MarginAccountFactory` (declaration at
`src/MarginRouter.sol:59-67`). It builds each curated flow as a fixed packed-opcode plan and runs it
inside one `PoolManager.unlock`; `execute` runs an arbitrary caller-supplied plan of the same opcode
vocabulary. The router is the `manager` of every account it deploys, so it alone (besides the owner)
can drive an account's lending primitives.

**Lineage and changes.** Not a fork; `V4Router`, `Permit2Forwarder`, `Multicall_v4`, and
`NativeWrapper` are shared v4-periphery bases also used by `PositionManager` and other v4 routers in
this repository. See Section 1.2 for this contract's own prior ABI generation (governed, now
removed).

**Mechanics.**

- *Increase* (`_increase`, `src/MarginRouter.sol:417`): snapshot `debtBefore` via
`adapter.positionOf` -> pull equity (native wrap or Permit2) -> `ROUTE_SWAP` (flash-take up to
`maxDebtIn` of debt, run the caller's route, which must deliver `collateralToBuy` to the account) ->
`ASSERT_ACCOUNT_BALANCE` -> `ACCOUNT_SUPPLY_COLLATERAL` (the account's full balance, via
`OPEN_DELTA`) -> `ACCOUNT_BORROW` (exactly the swap cost, via `OPEN_DELTA`) -> `SETTLE` -> optional
`ASSERT_HEALTH` (skipped when `maxLtvAfter == 0`).
- *Partial decrease* (`_partialDecrease`, `:186`): requires `maxLtvAfter` strictly between 0 and
`1e18` (`LtvBoundRequired` / `IneffectiveLtvBound` otherwise) -> swap collateral for exactly
`debtToRepay` -> repay -> withdraw exactly what the swap consumed (`OPEN_DELTA`) -> settle ->
enforce `maxLtvAfter`. A transient slot (`TransientPlanState`) records the measured repay so the
function can tell a repay that landed inside the unlock from one a third party made beforehand
(`:205`, `:220`).
- *Full close* (`_closePosition`, `:148`): if the position is already debt-free, withdraws collateral
directly with no unlock (`:151-159`); otherwise swaps collateral for exactly the current debt,
repays all (a venue-resolved full repay), withdraws all collateral to the router, settles, then
sweeps the account's entire debt-token balance (the route's over-buy plus anything the account
already held) back to the caller (`:174-175`, full-close branch only, see Section 6) and returns the
router's collateral gain to the caller as realized PnL. Both close branches emit a `PositionDecreased`
whose resulting-state fields are hardcoded to an empty position (`_emitClosed`, `:739-742`).
- *Route swap* (`_routeSwap`, `:573-612`): rejects a zero `universalRouter` (`UniversalRouterNotSet`),
flash-takes `maxIn` of the input currency from the `PoolManager`'s own balance, grants the
caller-supplied `universalRouter` a Permit2 allowance of exactly `maxIn` expiring at
`block.timestamp` (the ERC-20 approval to Permit2 itself is unlimited and set lazily once, `:591-593`),
runs the caller's `commands`/`inputs` unchanged, settles only the unspent portion of its own take by
snapshotting the router's own balance before and after (`:584`, `:603`), and finally zeroes the
Permit2 allowance to that `universalRouter` (`:611`, the OpenZeppelin L-08 fix). This router-balance
guard is distinct from the account-side `ASSERT_ACCOUNT_BALANCE` guard below, whose opcode handler is
absolute by itself. The route is an external call to a caller-chosen address made while the caller is
still the `ReentrancyLock` locker; re-entering any `isNotLocked` entry point reverts, and the account
primitives are gated to the manager and owner, so the route gains no authority beyond what its caller
already had (Section 6.1).
- *Account-balance assertion* (`_assertAccountBalance`, `:758`): the opcode handler itself checks
only that the account's **absolute** balance of a currency is at least `minAmount`; by itself that
is a floor, not a delta. Each curated flow converts it into a true delta guarantee by reading the
account's **live** balance immediately before encoding the plan and adding the required delivery
on top: `_increase` at `:469-470` (`collateralHeldBefore + params.collateralToBuy`) and
`_decreasePlan` at `:292` (`params.market.debt.balanceOf(account) + requiredDelivery`), both with
an explicit comment that this is so "a pre-existing or donated balance cannot mask a short fill."
This live pre-unlock read is what makes the curated flows immune to the account being pre-funded
or donated to before the call; a hand-built `execute` plan has no equivalent automatic snapshot and
must replicate the technique itself (Section 6).

**Expected inputs and caller responsibilities.** Callers MUST derive `maxDebtIn`/`maxCollateralIn`
from a real quote plus a slippage buffer; these are the only slippage bounds and they also size the
flash-take and the Permit2 allowance granted to the Universal Router, so an oversized value is a
real (if self-inflicted) risk, not just an inefficiency. `deadline` is enforced
(`DeadlinePassed`) and callers are expected to set it. The router holds no balance between
transactions by design for the curated flows; anything an `execute` plan leaves on the router is
claimable by the next caller (`docs/margin-trading.md` Section 4.1 item 3), so a plan MUST terminate
with `SWEEP` for every currency it may touch. The `universalRouter` argument must be a Universal
Router that already has `V4_SWAP` unlocked (a UR built after community PR #491); it is supplied per
call, not fixed at deployment, and is caller-chosen with no validation beyond a non-zero check
(`UniversalRouterNotSet`).

**Edge cases and intentional behavior.**

- A curated flow's exact-output swap is all-or-nothing: `IncompleteFill` reverts rather than opening
or closing a smaller position on a thin pool.
- `maxLtvAfter == 0` means "skip the check" on an increase (optional) but is rejected outright on a
partial decrease (`LtvBoundRequired`, mandatory); a non-zero bound at or above `1e18` is rejected
everywhere it is accepted (`IneffectiveLtvBound`), since it could never bind.
- A debt-free full close takes a different, unlock-free code path than a swap-based close and the
event emitted is governed by that branch, not the swap-based branch's sweep logic (Section 6).
- `execute`'s guarantees are strictly weaker than the curated flows': it does no entry validation,
`ASSERT_ACCOUNT_BALANCE`'s delta-equivalence trick (sizing the floor from a same-transaction
pre-unlock balance read) is not atomically reproducible by a hand-built plan, since `minAmount` is
static calldata and no opcode snapshots a balance mid-plan (the guide's workarounds are to run
against an account holding none of the output currency, or to accept that an off-chain snapshot can
be relaxed by a balance arriving before inclusion), and
signing an `execute` plan is "equivalent to handing over the sub-account": a malicious plan can
borrow to the market maximum, withdraw all collateral, and send it anywhere, with no token
approval required (`docs/margin-trading.md` Section 4.1, bolded warning).

**Areas of concern.** Whether every curated call site that encodes `ASSERT_ACCOUNT_BALANCE` keeps
reading a live pre-unlock balance (the property the delta guarantee depends on) if the plan-building
code is refactored; a future change that hoists the balance read earlier, or replaces it with a
caller-supplied value, would silently reintroduce the absolute-floor masking that a hand-built
`execute` plan already has by design (Section 6). The router's flash-take being bound by
`PoolManager`'s own ERC-20 inventory for
the input currency regardless of where the Universal Router actually sources liquidity, which can
make a position's close revert even though its open succeeded, for a large enough position in a
thin-inventory token (Section 6); the asymmetry between `_increase` (no router-balance guard after
its unlock) and `_partialDecrease`/`_closePosition` (both guard the router's own collateral balance)
(Section 6).

### 2.4 Lending Adapters (`ILendingAdapter` implementations)

**Purpose (shared across all four).** Each adapter is a stateless, unowned singleton that translates
the protocol-agnostic `(adapter, Market)` pair the router and account pass around into the concrete
calls a `MarginAccount` executes against one venue. The encode surface is
`encodeSupplyCollateral`/`encodeWithdrawCollateral`/`encodeBorrow`/`encodeRepay`, each returning a
`Call[]` (Section 2.5); the read surface is `lendingProtocol()`, `isSupportedMarket(Market)`,
`positionOf(account, Market)`, `maxLtvWad(Market)`, `currentLtvWad(account, Market)`, and
`describePosition(account, Market)` (`ILendingAdapter`, implemented by all four; all four also
implement `IAmountResolver` via `src/base/PositionAmountResolver.sol`, which supplies the amount-
resolution logic the router's `OPEN_DELTA`/`CONTRACT_BALANCE` sentinels need). There is no routing
table, no owner, and no `setMarket`: every encode and read re-validates the market key against the
live venue.

**Lineage and changes.** All four bind their venue singleton at construction
(`MorphoLendingAdapter(IMorpho)`, `AaveLendingAdapter(IPoolAddressesProvider)`,
`AaveV4LendingAdapter(ISpoke)`, `CompoundV3LendingAdapter(IComet)`) and hold it immutable. None
inherits an ownable base; the prior `OwnableAdapter` lineage described in the OpenZeppelin report
(Section 5) was removed.

**Mechanics and math (shared shape, venue-specific inputs).** Every adapter reports
`currentLtv = debtValue * WAD / collateralValue` as an `Ltv`, returning `type(uint256).max` for debt
against zero collateral value and `0` for no debt, and `healthFactorWad = maxLtv / currentLtv`
(`type(uint256).max` with no debt). What differs is where the values come from: Morpho prices
collateral in loan-token units through the market's own `IOracle.price()` (1e36 scale,
`src/MorphoLendingAdapter.sol:211-232`) and uses `lltv` as `maxLtv`; Aave v3 takes Aave's own
account-level `getUserAccountData` base-currency totals, liquidation threshold, and health factor
verbatim (`src/AaveLendingAdapter.sol:236-264`), so its numbers are account-wide, not per-market; Aave
v4 takes the Spoke's `getUserAccountData` and normalizes the RAY-scaled debt value
(`src/AaveV4LendingAdapter.sol:226-278`) with the reserve's current `collateralFactor` as `maxLtv`;
Compound values both legs in USD through Comet's price feeds (1e8 scale,
`src/CompoundV3LendingAdapter.sol:268-304`) with the liquidate collateral factor as `maxLtv`. No
adapter reimplements interest accrual, liquidation, or a close factor; those are read from, or left
entirely to, the venue (Section 6.4). `maxLtvWad` is the venue's liquidation boundary in every case,
never the borrow limit, which on Compound (borrow collateral factor) and Aave v3 (the `ltv` field) is
tighter.

#### 2.4.1 MorphoLendingAdapter

`Market.data` is `abi.encode(address oracle, address irm, uint256 lltv)` (96 bytes); the adapter
reconstructs Morpho's `MarketParams` and validates it is a created Morpho Blue market on every call.
Supply is a single `Call` (Morpho treats supplied collateral as collateral automatically). Morpho
market creation is itself permissionless, so the oracle, IRM, and LLTV are whatever the market's
creator chose; **the caller vets the market it names**, not the adapter (`docs/margin-trading.md`
Section 3.1, Section 9). A partial repay encoded at or within one wei of the live debt can switch
Morpho's share-rounding path and either underflow or clear the whole debt, depending on direction
(Section 6).

#### 2.4.2 AaveLendingAdapter (Aave v3)

Constructed from an `IPoolAddressesProvider`; resolves and stores the Pool immutably (a stable-
address proxy) but re-resolves `dataProvider()` from the provider **on every use rather than
caching it** (`src/AaveLendingAdapter.sol:85-92`, `:104-106`), specifically to stay correct across an
Aave data-provider redeployment. `Market.data` must be empty; Aave keys a market by the asset pair,
so any pair whose two assets are live Aave reserves is routable. The supply sequence ends with an
unconditional `setUserUseReserveAsCollateral(collateral, true)` call. Reads are account-level
(`getUserAccountData`), so co-locating two Aave v3 markets under one `(owner, subId)` blends their
health; keep one Aave position per account. `maxLtvWad` reads the reserve's base liquidation
threshold and does not account for the account's eMode category. `isSupportedMarket`/`_resolve`
accept any pair of live reserves, including the many Aave v3 reserves whose base LTV is zero, for
which the unconditional enable call reverts at Aave for an account outside eMode (Section 6.3).

#### 2.4.3 AaveV4LendingAdapter

Constructed against a single Spoke (`ISpoke`); a second Spoke needs a second adapter instance.
`Market.data` is `abi.encode(uint256 collateralReserveId, uint256 debtReserveId)` (64 bytes); the
adapter validates on every call that each reserve id exists, that its `underlying` matches the named
currency (`ReserveMismatch` otherwise), and that both reserves share a Hub (`HubMismatch`
otherwise). The supply sequence likewise ends with an explicit `setUsingAsCollateral` call, so,
unlike the v3 adapter, the v4 adapter is immune to Aave's first-supply-only auto-enable gap by
construction. Debt is read via `getUserTotalDebt` (drawn debt plus accrued premium, Spoke-native).
`currentLtvWad` normalizes a RAY-scale factor present in the Spoke's debt-value field; this
normalization was independently verified against the live Main Spoke (Section 5, severity annex
coverage notes).

#### 2.4.4 CompoundV3LendingAdapter

Constructed against a single Comet (`IComet`); a different base token needs a second adapter
instance. A Comet is single-base: every routable pair must have `debt == comet.baseToken()` and a
`collateral` that is a listed Comet collateral asset (`DebtNotBaseToken`, `MarketNotSupported`
otherwise); `Market.data` must be empty. Borrowing the base is `withdraw`ing it (driving the base
balance negative); repaying is `supply`ing it; there is no share-based "repay all," so a full close
supplies exactly `borrowBalanceOf(account)`, and any over-sized repay is clamped to that figure
(`src/CompoundV3LendingAdapter.sol:189-191`) so an overshoot is never converted into a base supply
position. `maxLtvWad` surfaces only the venue's **liquidate**
collateral factor; Comet separately enforces a tighter **borrow** factor on the withdraw leg of a
decrease, so a position that has drifted above the borrow factor (but is still under the liquidate
factor) can reject a small partial decrease while a full close still succeeds
(`docs/margin-trading.md` Section 8.1). The adapter's `positionOf`/`describePosition` read only
`comet.borrowBalanceOf(account)`; they never read `comet.balanceOf(account)`, the Comet-native
signed base-asset balance that becomes a positive supply after a liquidation absorbs the account
(Section 6, L-1).

### 2.5 Shared Types, Libraries & Errors

- `Market` (`src/types/Market.sol`) - `{Currency collateral, Currency debt, bytes data}`. `data`
is opaque outside the adapter that defines it; see Section 2.4 for each adapter's encoding and
Section 1.2 for why this field exists (it replaced a stored registry entry).
- `Ltv` (`src/types/Ltv.sol`) - `type Ltv is uint256`, WAD-scaled (`1e18 == 100%`), with an
attached `gt` comparison (`using {gt} for Ltv global`) used for health checks. A distinct type
specifically so health math cannot be accidentally added to a raw token amount.
- `PositionData` (`src/types/PositionData.sol`) - the consolidated snapshot
`{collateralAmount, debtAmount, Ltv maxLtv, Ltv currentLtv, uint256 healthFactorWad}` returned by
`describePosition`. No price fields by design; this keeps the struct oracle-unit-agnostic across
every adapter.
- `Call` (`src/types/Call.sol`) - `{uint256 value, bytes data}`, one step of an adapter-encoded
sequence the account executes against `lendingProtocol()`. `value` is always zero for every in-
tree venue.
- `MarginActions` (`src/libraries/MarginActions.sol`, 94 lines) - the packed-opcode constants for
the margin-specific actions (`SET_ACCOUNT`, `PULL_TO_ACCOUNT`, `ACCOUNT_SUPPLY_COLLATERAL`,
`ACCOUNT_BORROW`, `ACCOUNT_WITHDRAW_COLLATERAL`, `ACCOUNT_REPAY`, `ACCOUNT_SWEEP`, `ROUTE_SWAP`,
`ASSERT_HEALTH`, `ASSERT_ACCOUNT_BALANCE`), pinned to a disjoint range from the inherited v4
routing opcodes so one packed `actions` byte string can mix both vocabularies (see the suite's own
`test_opcodes_arePinnedAndDisjointFromActions` unit test). Has no unit test file of its own; it is
exercised only transitively through every integration, invariant, and fork test that dispatches an
action (Section 7).
- `MarginCalldataDecoder` (`src/libraries/MarginCalldataDecoder.sol`, 110 lines) - decodes each
margin opcode's packed parameters out of `execute`'s `bytes[] params`. 12 unit tests (3 of them
fuzz) in `test/libraries/MarginCalldataDecoder.t.sol`.
- `TransientPlanState` (`src/libraries/TransientPlanState.sol`, 51 lines) - the two EIP-1153
transient slots a plan carries across one unlock: the active account bound by `SET_ACCOUNT` or the
curated entry points, and the amount the last `ACCOUNT_REPAY` measured. Written only by the router,
never from calldata; the only inline assembly in the scoped files (`memory-safe` `tstore`/`tload`).
The active-account slot is cleared when an entry point returns; the last-repaid slot is reset before
a partial decrease's unlock but never cleared afterward (Section 6.1).
- `Units` (`src/libraries/Units.sol`) - the `WAD`, `RAY`, and `BPS` scale constants the adapters and
router share.
- `PositionAmountResolver` / `IAmountResolver` (`src/base/PositionAmountResolver.sol`,
`src/interfaces/IAmountResolver.sol`) - the shared logic all four adapters use to resolve an
`OPEN_DELTA`/`CONTRACT_BALANCE` sentinel amount against the venue's live position, so the router's
generic amount-resolution opcodes work uniformly across venues.

Key errors (see `docs/margin-trading.md` Section 11 for the full table with every adapter error):
`DeadlinePassed`, `SlippageBoundRequired`, `LtvBoundRequired`, `IneffectiveLtvBound`, `ZeroAmount`,
`PositionUnhealthy`, `NoDebtToRepay`, `NativeCollateralMismatch`, `NativeNotSupported`,
`IncompleteFill(requested, received)`, `NoActiveAccount`, `UniversalRouterNotSet` (router);
`UnsupportedAction(uint256)` (inherited from `BaseActionsRouter`, the revert for an undefined opcode
in an `execute` plan), `InvalidEthSender` (inherited from `NativeWrapper`, the router's `receive()`
rejecting native value from anyone but WETH9 or the `PoolManager`); `NotAuthorized`,
`ReceiverNotAllowed(address)` (account); `MarketNotSupported(Currency, Currency)`,
`InvalidMarketData(uint256)` (declared on `ILendingAdapter`, raised by all four adapters);
`ZeroAddress` (all four adapters and the factory); `AccountMismatch(address, address)` (Aave v3,
Aave v4, and Compound only); `DebtNotBaseToken(Currency, address)` (Compound);
`ReserveMismatch(uint256, address, address)`, `HubMismatch(address, address)` (Aave v4).

Key events: `PositionUpdated` (resulting-state snapshot, emitted after every supply/withdraw/
borrow/repay on every router path including `execute` plans and the unlock-free paths),
`PositionIncreased`, `PositionDecreased`, `CollateralAdded` (router, curated flows only, carry per-
operation deltas); `CollateralSupplied`, `CollateralWithdrawn`, `Borrowed`, `Repaid`, `Swept`, `Executed`
(account; `Executed` is the only trace of an escape-hatch call, and `CollateralWithdrawn`'s amount is
the account's own balance delta, which is zero on three of the four venues, Section 6.2);
`AccountCreated` (factory). None of the four adapters emit events.

## 3. Deployment Environment

### 3.1 Supported Chains

The audited deployment is Ethereum mainnet only (`script/DeployMargin.s.sol`, chain 1; addresses in
`docs/margin-trading.md` Section 10). The contracts are chain-agnostic by construction and
`DeployMargin` accepts the venue addresses as parameters, so a future deployment on another chain is
expected to pass that chain's own `PoolManager`, Permit2 (same address everywhere), and venue
addresses; no code change is required for a standard EVM L1 or L2. Mempool, `block.number` semantics,
and gas-limit considerations therefore depend on wherever a given deployment lands and are not fixed
by this source; they are out of scope for a mainnet-only review unless the engagement is scoped
wider.

**Native-wrapper-less chains.** Where a chain has no canonical 1:1 wrapped native (the stated
motivating case is Arc, where native USDC at 18 decimals and the ERC-20 USDC predeploy at 6 decimals
are two views of one balance with no wrapper between them), the router is deployed with
`weth9 = address(0)`, and every native-value code path (`msg.value` equity, `WRAP`, `UNWRAP`) reverts
`NativeNotSupported`. This is enforced in the router itself (`NativeWrapper` base plus the explicit
checks in `_increase`/`addCollateral`), not by convention, so a misconfigured deployment fails closed
rather than crediting a native amount at the wrong decimal scale.

### 3.2 Tokens

Standard ERC-20 only. The router's curated flows are designed to fail closed on a non-standard
token (a fee-on-transfer or rebasing asset trips the fill and settle assertions) rather than silently
mis-accounting it, but this has not been independently fuzzed against a non-standard token for this
document; treat it as a design intent to verify, not a proven guarantee. `Currency.wrap(address(0))`
(native ETH) is valid at the v4-core type level but margin markets are ERC-20 only in practice: the
only native-value path is wrapping equity to WETH before it reaches any lending venue. Decimals are
whatever the token uses: the worked examples in `docs/margin-trading.md` use WETH (18), USDC (6), and
UNI (18), and all sizing math in the integration guide is written to use each token's actual
`decimals()` rather than assume a layout.

Market *selection* is permissionless (any market the chosen venue's adapter can resolve is routable),
so there is no protocol-level token allowlist to describe beyond "whatever the venue lists as a
reserve/market/collateral asset." The caller is expected to do their own due diligence on the
specific market they name, most acutely on Morpho Blue, where market creation is itself
permissionless (Section 2.4.1, Section 4).

### 3.3 External Integrations & Dependencies

- **Uniswap v4** `PoolManager` - the flash-accounting substrate every curated flow and `execute`
plan unlocks. Trusted to behave as the canonical v4-core deployment.
- **Permit2** - pulls caller equity and funds the Universal Router's spend for a routed swap via a
scoped, per-call allowance. Canonical address, same on every chain.
- **A Universal Router** - supplied per call (`universalRouter` field), not fixed at deployment. Must
already have `V4_SWAP` unlocked (built after community PR #491). Because it is a per-call argument,
a hostile or buggy UR address only harms the caller who named it (Section 2.3, Section 4.1); it is
also how a deployment adopts a newer UR without redeploying the router.
- **Morpho Blue, Aave v3 Pool, an Aave v4 Spoke/Hub, a Compound v3 Comet** - the four lending venues.
Each is trusted to price, accrue interest on, and liquidate positions correctly; the margin suite
adds no independent oracle and performs no local liquidation logic (Section 4.1).
- **Each venue's own oracle** (Morpho's per-market `IOracle`, Aave's price oracle, the Spoke's
oracle, Comet's price feeds) - read only through the adapter's `currentLtvWad`/`describePosition`/
health calls. An oracle outage blocks any curated flow whose plan runs `ASSERT_HEALTH` with a
non-zero bound (every partial decrease; an increase only when `maxLtvAfter != 0`); a full close and
`addCollateral` never consult the oracle on their mutation path, and every event-only read is
wrapped in try/catch, so an outage degrades those to skipped events rather than reverts. The owner
escape hatch is never affected (Section 6).

## 4. Roles, Permissions & Trust Model

### 4.1 Privileged Roles

There is no admin surface anywhere in this suite. Specifically:

- `MarginRouter` **as account manager** - Fully trusted by every account it deploys (it is baked
into the clone's bytecode as `manager()`), but this trust is structural, not a key someone holds:
the router's logic is fixed at deployment, it has no owner, and it cannot be redirected to act on
an account other than the one its own code derives from the authenticated caller.
- **Position owner** - Untrusted by the protocol toward anyone except themselves; trusted absolutely
by their own `MarginAccount` (manager-or-owner gate on every fund-moving call, Section 2.1). The
owner can always act through `execute` directly on the lending protocol, bypassing the router.
- **Adapter named on a call** - Caller-chosen on every call, with no allowlist and no router-side
vetting. An adapter is fully trusted by the account that executes its encoded `Call[]` sequence,
but that trust is scoped to the caller: the router always derives the account from the
authenticated caller, so a hostile adapter reaches only the account of whoever named it. Front
ends should hardcode the adapters they have reviewed and never take an adapter address from
untrusted input (`docs/margin-trading.md` Section 9).
- **Market named on a call** - Likewise caller-chosen with no allowlist. On Aave and Compound the
venue's own governance lists markets, so the universe the adapter can resolve is venue-vetted; on
Morpho Blue, market creation is itself permissionless, so the caller (or the integrator building
the call) is the only party vetting the oracle, IRM, and LLTV a given key encodes.
- **Deployer EOA** - Only relevant at deployment time. Every contract deploys through the canonical
CREATE2 deployer at a deterministic, idempotent address; no address is written into any contract
as an owner (`script/DeployMargin.s.sol` NatSpec). There is nothing for this key to do after
deployment.
- **Upgrade authority** - None. No contract in this suite is a proxy, inherits an initializer
pattern, or has an owner-gated upgrade function (verified: no `Ownable`, `Initializable`, or
`onlyOwner` pattern anywhere in the scoped `src/` files). Everything is immutable from the moment
it is deployed. A permanent lock of funds, if one existed, could not be remediated by an upgrade;
there is none known as of this document (Section 6).

### 4.2 Configurable Parameters & Valid Ranges

This suite has no privileged, post-deployment-configurable parameters: nothing here maps to the
classic "an admin role sets X within a documented range" pattern, because there is no admin role.
The closest analog is the set of bounds every caller supplies on their own call, which the protocol
enforces but does not itself choose a value for:

- `maxDebtIn` (increase) / `maxCollateralIn` (decrease/close) - mandatory, caller-supplied. Valid
range: any non-zero value; `SlippageBoundRequired` on zero. This is also the flash-take amount and
the Permit2 allowance granted to the Universal Router, so it additionally must not exceed the
`PoolManager`'s own ERC-20 inventory of the input currency, or the swap reverts regardless of the
venue's own liquidity (Section 2.3, Section 6).
- `maxLtvAfter` - optional on an increase (zero skips the check), mandatory on a partial decrease
(zero reverts `LtvBoundRequired`), ignored on a full close. Valid range when supplied: strictly
below `1e18` (100%); `IneffectiveLtvBound` at or above it, since such a bound could never bind.
Expected production usage: sized to the caller's own risk tolerance, well below the venue's
`maxLtvWad` for the market.
- `deadline` - mandatory on every entry point that accepts it, a Unix timestamp;
`DeadlinePassed` once `block.timestamp` exceeds it. No upper or lower bound is enforced beyond
"in the future relative to inclusion."
- `subId` - caller-chosen, unconstrained `uint256`, used only to derive the account address
(Section 2.2). No validation that a given venue's one-position-per-account requirement is
respected across `subId`s; this is a caller responsibility, not an enforced range.

### 4.3 Off-Chain Components & Operational Assumptions

- **Front ends / integrating contracts** - expected to: build the Universal Router route so it
delivers the exact requested output to the active account and draws input from the router as
payer via Permit2; derive `maxDebtIn`/`maxCollateralIn` from a real quote, not spot price; only
offer adapters and markets they have reviewed; and never construct or forward an `execute` plan
built by an untrusted party. None of this is enforced on-chain; it is entirely an off-chain
operational assumption, and Section 6 documents what happens when it is violated.
- **Indexers** - expected to key resulting state off `PositionUpdated` uniformly across curated and
`execute` paths, and to be aware that the delta-shaped events (`PositionIncreased`,
`PositionDecreased`, `CollateralAdded`) report **requested** amounts in several fields, not
measured ones, and that every `describePosition`-derived emission is best-effort (skipped, not
reverted, if the venue read fails). See `docs/margin-trading.md` Section 4.1 "Event-field caveats"
for the complete list; Section 6 of this document lists the specific cases that are currently
known to diverge.
- **No keepers, relayers, or governance procedures exist for this suite.** Liquidation, interest
accrual, and oracle updates are entirely the lending venue's own off-chain/on-chain machinery; this
suite has no off-chain component of its own beyond the front end that constructs calldata.

## 5. Previous Audits

- **OpenZeppelin, "Uniswap 10 Retainer 26: Margin Trading"** (`audits/OpenZeppelin_audit_margin_trading.pdf`).
Scoped against commit `e9b50f5` (2026-08-05, the pre-market-key architecture described in
Section 1.2). 33 findings reported, 0 Critical/High, 3 Medium, 14 Low, 16 Notes. Every finding was
independently re-validated against that commit by this team (tracked internally; not part of the
protocol source) and all 33 were confirmed as real against the audited snapshot, with no fund-
theft or insolvency path in any of them. **Caveat for this engagement:** the registry, governance,
`setMarket`, and `OwnableAdapter` findings (L-02, L-03, L-06, L-11, N-01, N-03, N-08, N-12, N-16)
concern code that was removed by the permissionless-market-key refactor and do not apply to current
`src/`, although the substance of L-06 (a venue delisting a market makes the curated exits revert)
and N-01 (a nonsensical but existing Morpho market is routable) survives in permissionless form and
is listed in Section 6. Every other OpenZeppelin finding was re-checked against current HEAD for this
document; Section 6.5 gives the per-finding status. Of the three Mediums (none of which concern the
registry): M-01 (the Aave protocol data provider cached immutably instead of re-resolved) has been
fixed, `AaveLendingAdapter.dataProvider()` now re-resolves from the addresses provider on every use
(`src/AaveLendingAdapter.sol:104`); M-02 (the Aave v3 supply sequence relying on Aave's auto-enable
instead of an explicit collateral-enable call) also appears fixed, the current adapter's supply
sequence ends with an explicit `setUserUseReserveAsCollateral` call matching the fix OZ's own
tracker recommended (`audit-validation/oz-margin-trading.md` M-02), though this was not
independently re-run as a fork PoC for this document; M-03 (the routed-swap fill assertion
measuring an absolute balance instead of the delivered amount) is fixed for the curated flows,
`_increase` and `_decreasePlan` both size their `ASSERT_ACCOUNT_BALANCE` floor from a live
pre-unlock balance read specifically to close this gap (Section 2.3, Section 6.1).
- **Pashov AI audit** (`v4-periphery-pashov-ai-audit-report-20260929-105028.md`,
`v4-periphery-pashov-ai-audit-severity-20260929-105028.md`). Commit `f87a6fc0` (2026-09-23), scan
run 2026-09-29, already on the current permissionless-market-key architecture; scope matches this
document's scope line almost exactly. 5 scored findings, all Low, 0 Critical/High/Medium, plus 17
unscored "leads" (concrete code smells the methodology could not fully chain into a complete exploit in one
pass). Re-checked against current HEAD for this document: findings 1 through 5 are all still open
(Section 6); none has been fixed since this audit. The 17 leads are each dispositioned in Section 6
(the Aave v4 RAY-scale lead was refuted by the annex's own live-Spoke read and is treated as closed).
- **Simao accounting-first audit** (`v4-periphery-simao-audit-report-20260929-110019.md`). Same
commit `f87a6fc0`, same architecture, independent methodology (money-model-first). 4 Low findings
(L-1 through L-4), 6 Info findings (I-1 through I-6), 0 Critical/High/Medium, plus a short list of
unverified leads. Re-checked against current HEAD for this document: L-1 (Compound base-supply
invisibility), L-2 (multicall native double-wrap, the same defect as Pashov #3/#5), L-3 (over-
repaid partial decrease, the same defect as Pashov #4), and L-4 (Aave dust-collateral griefing) are
all still open (Section 6). I-1 (stale `_readPosition` NatSpec), I-2 (last-repaid transient slot
never cleared after the unlock), I-3 (flash-take inventory cap), I-4 (zero-LTV Aave reserves
accepted; `maxLtvWad` ignores eMode), I-5's first half (Aave v4 `maxLtv` read from the reserve's
current config key rather than the position's snapshotted one), and I-6 (two inherited Comet quirks)
are all still open and listed in Section 6; I-5's second half (stale NatSpec at
`AaveV4LendingAdapter.sol` claiming a router allowlist) has been fixed: no such sentence remains in
current source.

All issues in the three reports above are treated as known for this engagement, subject to the
corrections and fix-status notes in Section 6: a submission that reproduces one of them, with no
additional novel element, should not be treated as a new finding.

## 6. Known Issues, Accepted Risks & Design Decisions

Every item below was checked against current HEAD (`1c86348d`) while writing this document; the
citation after each item says what was checked and where. Items not re-verified line-by-line are
marked explicitly.

### 6.1 MarginRouter

- **Position events omit the adapter, so two venues that share an empty-data market key are
indistinguishable in logs.** `PositionIncreased`, `PositionDecreased`, `CollateralAdded`, and
`PositionUpdated` carry `(collateral, debt, marketData)` but no adapter address; Aave v3 and
Compound both key with empty `data`, so their positions for the same pair collide in an indexer
keyed only on the event fields. Confirmed still open: `grep "address indexed adapter" src/interfaces/IMarginRouter.sol` matches nothing. Source: Pashov finding #1 (confidence 85).
Severity Low.
- `_increase` **does not guard the router's own debt-token balance, unlike the decrease paths.**
`_partialDecrease` and `_closePosition` both snapshot the router's relevant balance before their
unlock and net against it after (`src/MarginRouter.sol:167`, `:203`, `:315`); `_increase`
(`:417-486`) takes no equivalent snapshot of the debt token before its unlock. In principle a
short borrow could settle from tokens the router already holds (the only realistic source being
undocumented `execute`-plan residue, which is already claimable by the next caller) instead of
reverting. Confirmed open by direct inspection of `_increase`. Source: Pashov finding #2
(confidence 75). Severity Low; the reachable value is the already-documented router residue.
- **A second native leg inside one** `multicall` **wraps ETH the router already holds, instead of
reverting.** `Multicall_v4.multicall` delegatecalls every leg with the same `msg.value`, and both
`addCollateral` and `_increase` wrap and credit `msg.value` on every leg without checking whether
that specific leg funded it. `docs/margin-trading.md`'s current text ("do not batch two native-ETH
position calls... the second wrap would revert") is incorrect when the router holds any ETH; the
wrap only reverts when the router's balance is exactly zero. Confirmed open:
`NativeValueUsed`/`NativeValueAlreadyUsed` do not exist anywhere in `src/`. Source: Pashov findings
#3 and #5 (confidence 75 each); Simao L-2. Severity Low (bounded by the router's `receive()`
accepting only WETH9/PoolManager, so ETH reaches the router only via undocumented `execute`
residue).
- **An over-sized or front-run partial decrease strands bought debt tokens in the account.**
`_partialDecrease`'s only gate is that the measured repay is non-zero
(`TransientPlanState.lastRepaid() == 0` check at `src/MarginRouter.sol:220`); every venue caps the
actual repay at the live debt, so a `debtToRepay` above the live debt (or a debt reduced by a
third-party repay landing before inclusion) leaves the difference, already bought by the swap,
sitting in the account, while `PositionDecreased.debtRepaid` still reports the requested amount.
The full-close branch has the equivalent sweep (`:174-175`); the partial-decrease branch does not.
Confirmed open by direct inspection: no `debtOver`/sweep logic exists in `_partialDecrease`.
Source: Pashov finding #4 (confidence 75); Simao L-3. Severity Low; recoverable via
`MarginAccount.sweep` or an `execute` plan's `ACCOUNT_SWEEP`, no attacker profit.
- **(Fixed, re-stated here only to correct the record.)** OpenZeppelin M-03 reported that
`_assertAccountBalance`'s absolute-balance check let a pre-existing or donated account balance mask
a short-filled swap. Re-checked against current HEAD for this document: it does not hold. Both
`_increase` (`src/MarginRouter.sol:469-470`) and `_decreasePlan` (`:292`) read the account's **live**
balance immediately before encoding `minAmount` and add the required delivery on top, with an
explicit comment that this is "so a pre-existing or donated balance cannot mask a short fill"
(Section 2.3). This converts the opcode's absolute floor into a true delta guarantee for every
curated flow. The only place the absolute-floor semantics remain caller-visible is a hand-built
`execute` plan, which has no automatic snapshot and must replicate the live-balance-read technique
itself; `docs/margin-trading.md` Section 4.1 already documents this explicitly as a limitation of
`execute`, not of the curated flows.
- **The flash-take for a position swap is bound by the** `PoolManager`**'s own ERC-20 inventory of the
input currency, not by where the Universal Router actually sources liquidity**, and that inventory
can be smaller than the WETH-equivalent native ETH balance the same pool holds. A position sized
against ETH liquidity can open successfully (equity in ETH, debt borrowed and swapped) and then
fail to close in one call if its required input exceeds the pool's ERC-20 balance of that token,
even though the pool holds ample native ETH. Not independently re-verified against current mainnet
inventory for this document. Source: Pashov lead (seen 1/3 runs); mitigated by chunking a
partial decrease across a `multicall`, or by the owner escape hatch.
- **A plan passed to** `execute` **is "equivalent to handing over the sub-account."** Documented
design decision, not a defect: `execute` does no entry validation, so a malicious or buggy plan
can borrow to the market maximum, withdraw all collateral, and send it anywhere, with no token
approval required. `docs/margin-trading.md` Section 4.1 states this explicitly; front ends are
expected to construct `execute` calldata themselves and never forward one from an untrusted
party.
- **Anything an** `execute` **plan leaves on the router is claimable by the next caller.** Documented
design decision: the router holds no governance-protected balance, so a plan that does not
terminate with `SWEEP` for every currency it touched creates a race any subsequent caller can win.
- **Both full-close branches emit a** `PositionDecreased` **whose resulting-state fields are hardcoded
to an empty position** (`_emitClosed`, `src/MarginRouter.sol:739-742`: zero collateral, zero debt,
zero LTV, unbounded health). On Aave v3/v4, whose reads are account-level, an account that co-locates
a second position under the same `subId` is not actually empty after the close. Mitigation: the
swap-free branch co-emits a measured `PositionUpdated` (`:156-157`) and the swap-based branch emits
one after each action inside the unlock; indexers should key on `PositionUpdated`. Source:
OpenZeppelin L-07. Severity Low, observability only.
- **A full close of a debt-free, collateral-free account is a silent success** (`:151-159`): it
withdraws nothing and still emits the canonical close event. Source: OpenZeppelin L-10. Severity Info.
- **An increase supplies the account's entire collateral balance, not just equity plus the bought
amount** (`OPEN_DELTA`, `:472-473`), so any idle collateral the account held is absorbed and the
`PositionIncreased.equity` field understates what was supplied. Accepted design: the account is a
position container, and a sweep-first `execute` plan is the opt-out. Source: OpenZeppelin L-09.
- **A full close of an account holding debt against zero collateral cannot run through the curated
path.** `_closePosition` passes `collateralBefore` as the withdraw amount; zero is also the
`OPEN_DELTA` sentinel (`ActionConstants.OPEN_DELTA == 0`), so `_withdrawCollateral` resolves it to
the swap cost (`:657`) and the venue withdraw reverts. Reachable only after a venue-side seizure
leaves residual bad debt (Aave v3); documented as unclosable in `IMarginRouter.sol:314`; the escape
hatch repays it. Source: Pashov lead (seen 2/3 runs); Simao unverified lead. Not reproduced on a fork.
- **An underwater position (LTV at or above 100%) has no curated partial delever**, because a partial
decrease's `maxLtvAfter` must be strictly below `1e18` even when each step would improve the
position. A full close or the escape hatch still works. Accepted design. Source: Simao unverified lead.
- **The caller-supplied Universal Router is called with the fixed** `execute(bytes,bytes[],uint256)`
**selector while the caller is still the** `ReentrancyLock` **locker** (`:598`), with no validation of
the address beyond non-zero. Re-entering any `isNotLocked` entry point reverts; the account
primitives are gated to the manager and owner, so a hostile route gains no authority its caller did
not already have, and the Permit2 allowance it can spend is scoped to `maxIn` and zeroed afterward.
Accepted design (the UR is a per-call parameter precisely so a deployment can adopt a newer one).
Source: OpenZeppelin L-04; Pashov leads (seen 1/3 and 3/3 runs). Severity Low, caller-scoped.
- **A** `decreasePosition` **against an undeployed account can emit a close event for an address with
no code.** `decreasePosition` derives the account with `accountOf` and never deploys (`:137`), so a
caller-chosen adapter that reports `(0, 0)` for the empty address makes the swap-free branch emit a
canonical `PositionDecreased` and `PositionUpdated` for an account that does not exist. Event
spoofing only, scoped to the caller's own derived address. Source: Pashov lead (seen 1/3 runs).
- **The transaction sender can suppress the rich position events with a tight gas limit.** The
event-only `describePosition` reads are wrapped in try/catch (`:108`, `:229`, `:350`, `:688`), so a
sender who meters gas so that the read runs out (the 63/64 rule) completes the mutation but skips
the snapshot and delta events. Accepted consequence of making those reads best-effort (OpenZeppelin
L-05's fix). Source: Pashov lead (seen 1/3 runs). Severity Info.
- **Zero is overloaded as a control value in several fields.** `maxLtvAfter == 0` means "skip" on an
increase but reverts `LtvBoundRequired` on a partial decrease; `collateralToBuy == 0`,
`debtToRepay == 0`, and a zero `PULL_TO_ACCOUNT` amount revert `ZeroAmount`; `OPEN_DELTA` is itself
zero. A genuinely zero LTV bound is therefore unexpressible. Partially addressed since OpenZeppelin
N-10 (the partial-decrease case now reverts loudly); the rest is accepted design.
- **The last-repaid transient slot is reset before a partial decrease's unlock but never cleared
after it, or after** `execute`. `_partialDecrease` writes zero at `:205` before its plan, and `execute`
clears only the active-account slot (`:373`), so `LAST_REPAID_SLOT` can hold a stale value when an
entry point returns. No reader exists today that would consume it; flagged against the stated
invariant that every transient slot is zero on return. Source: Simao I-2. Severity Info.
- `_readPosition`**'s NatSpec still claims withdraw and repay are "never market-gated"** (`:676-677`),
but every adapter's `encodeWithdrawCollateral`/`encodeRepay` resolves the key first and reverts
`MarketNotSupported` for a delisted Comet asset or an unlisted Aave v4 reserve, so a venue-side delist
makes the curated exits revert and the escape hatch is the guaranteed exit. Documentation defect;
the surviving substance of OpenZeppelin L-06. Source: Simao I-1. Severity Info.

### 6.2 MarginAccount

- **Permissionless venue actions against a predictable, possibly undeployed account.** Morpho, Aave
v3, and Comet all accept a supply or repay "on behalf of" any address, and every account address
is CREATE2-derivable from `(owner, subId)` before it is even deployed. This single venue property
is the root cause of several items below (the Aave dust-collateral griefing in Section 6.3, and
the documented-but-not-reproduced "third party repays ahead of a partial decrease" scenario). None
of the reachable variants are profitable for the attacker, and all are recoverable through the
owner escape hatch. Source: Simao systemic observation, cross-referenced against current
`MarginAccountFactory`'s deterministic address derivation (Section 2.2).
- `repay`**'s max-sentinel path approves the account's entire debt-token balance to the adapter's
venue target before the call.** A hostile adapter's call target could in principle pull the whole
approved balance during that single call; this reaches only the account of whoever named that
adapter (Section 4.1), so it adds no cross-user risk, but is worth noting for any new adapter
review. Verified at `src/MarginAccount.sol:124-128`: `approveAmount` is the account's full debt-token
balance for the max sentinel, reset to zero after the sequence. Source: Pashov lead (seen 1/3 runs).
- `CollateralWithdrawn` **logs the account's own balance delta, which is zero on Morpho, Aave v3, and
Compound** because those venues deliver straight to the validated receiver (`:83-87`); only Aave v4
routes through the account and logs the delivered amount. The transfer is correct; only the logged
amount is a false zero, and it is the only withdrawal event on the escape-hatch path. Source:
OpenZeppelin N-05. Severity Info, observability only.
- **Sweeping native currency from an account to the manager always reverts**, because the router's
`receive()` accepts value only from WETH9 and the `PoolManager` (`src/base/NativeWrapper.sol:31-33`);
sweeping to the owner works, and no in-tree adapter puts native currency into an account, so this is
unreachable today. Source: OpenZeppelin N-11. Severity Info.

### 6.3 Lending Adapters

- **Compound:** `positionOf`**/**`describePosition` **never read the account's base-asset supply, so a
liquidated account's Comet refund is invisible to every router read and the curated close leaves
it at Comet.** When Comet absorbs a liquidatable account it seizes the collateral, cancels the
debt, and credits any surplus above the debt (less the liquidation discount) to the account as a
positive base-asset balance (Comet's signed-principal accounting: negative is a borrow, positive
is a supply). `CompoundV3LendingAdapter.positionOf`/`_positionValues`/`encodeRepay` all define debt
as `comet.borrowBalanceOf(account)` alone; nothing reads `comet.balanceOf(account)`. After an
absorption, every adapter read reports an empty, healthy position, and the router's debt-free
close branch withdraws only `collateralBefore` (zero), so the refund stays at Comet and only the
owner escape hatch (`execute` a Comet `withdraw(base, max)` then `sweep`) reaches it. Verified on a
mainnet fork at block 25,598,384: a 2x UNI long absorbed after a 35% price drop left 272.220323
USDC of base supply recoverable only via the escape hatch; the curated full close returned 0/0.
Confirmed still open: no `comet.balanceOf` or equivalent read exists anywhere in
`src/CompoundV3LendingAdapter.sol`. Source: Simao L-1. Severity Low (funds stay owned by the
account, one escape-hatch call recovers them, and a later Comet open on the same `subId` draws the
supply down first).
- **Aave v3: a third party can pre-supply dust of another reserve to a predictable, undeployed
account, and the adapter's unconditional collateral-enable call can then revert for the real
owner.** `Pool.supply(asset, 1, victimAccount, 0)` needs no permission; Aave auto-enables a
non-isolated first supply as collateral. If the injected reserve is later made isolated (a debt
ceiling), the victim's own explicit enable of an isolated collateral reverts
`USER_IN_ISOLATION_MODE_OR_LTV_ZERO`, blocking every `increasePosition`/`addCollateral` on that
market until the owner intervenes via the escape hatch. Likelihood is low today (no mainnet Aave
v3 Core reserve carries a debt ceiling as of the source audit), and the owner recovers with one
`setUserUseReserveAsCollateral(dust, false)` call, though a motivated attacker can re-grief a fresh
`subId`. Line-citation drift: Simao's report cites a separate `encodeEnableCollateral` function at
the commit it audited; current HEAD has folded that logic directly into `encodeSupplyCollateral`
(`src/AaveLendingAdapter.sol:124-138`), unconditional enable call included, so the mechanism is
unchanged, only the function name and lines moved. Not independently re-verified against live Aave
reserve configuration for this document. A second variant: if governance later sets the injected
reserve's LTV to zero, Aave's `validateHFAndLtv` rejects the victim's partial-decrease withdraws
(`LTV_VALIDATION_FAILED`) while debt is open; the full close still works because it repays before it
withdraws. Source: Simao L-4; Pashov lead (seen 3/3 runs) corroborates the same mechanism
independently. Severity Low.
- **Aave v3: reserves whose base LTV is zero are accepted as markets but cannot be used as margin
collateral outside eMode.** `isSupportedMarket`/`_resolve` accept any pair of live reserves
(`src/AaveLendingAdapter.sol:110-114`, `:272-278`); for a zero-LTV reserve (roughly 45 of 67 mainnet
reserves at the source audit, including DAI, LINK, and UNI), the adapter's unconditional
`setUserUseReserveAsCollateral` reverts at Aave (`UserHasAssetWithZeroLtv`) for an account outside
eMode, and the router has no eMode path. An eMode category can only be entered through the escape
hatch. Not re-verified against live reserve configuration for this document. Source: Simao I-4.
Severity Info, usability.
- **Aave v3 / Compound: a dust debt whose base-currency value floors to zero reads as zero LTV and
unbounded health**, so the mandatory partial-decrease bound cannot fire on it (`_currentLtv` divides
Aave's 8-decimal base totals, `src/AaveLendingAdapter.sol:238-239`; Compound's `describePosition`
guards on `debtValue == 0` for the same reason, `src/CompoundV3LendingAdapter.sol:249-254`, which
matters for a Comet with an 18-decimal base, not cUSDCv3). No path to a loss above dust was found.
Source: Pashov lead (seen 1/3 runs); Simao I-6. Severity Info.
- **Aave v3/v4:** `maxLtvWad` **does not account for the account's eMode category.** Confirmed by
inspection: neither `src/AaveLendingAdapter.sol` nor `src/AaveV4LendingAdapter.sol` references
eMode anywhere. `describePosition`'s `currentLtvWad`/health-factor path uses the venue's own
account-weighted values and is unaffected; only an off-chain consumer reading `maxLtvWad` directly
for a market would see a value that ignores eMode. Source: Pashov lead; Simao I-4. Severity Info/Low,
observability only.
- **Aave v4:** `maxLtv` **is read from the reserve's current dynamic config key, while the Spoke values
an existing position by the key snapshotted into the position until a refresh**
(`src/AaveV4LendingAdapter.sol:257-260`). After a governance config change, `PositionUpdated.maxLtv`
can disagree with the `healthFactorWad` emitted beside it; `healthFactorWad` is the Spoke's own and
is authoritative. Source: Simao I-5. Severity Info.
- **Aave v4: two unverified rounding leads on the full close.** (a) The close sizes its fill floor and
repay from `getUserTotalDebt`; if the Spoke's repay-all pulls a total rounded above that view, a close
with no spare debt token reverts. (b) The close withdraws the viewed `getUserSuppliedAssets` amount
rather than a Spoke full-withdraw signal, so a view below the withdrawable maximum could leave
collateral shares behind. Neither was verified against Spoke source (not vendored). Source: Pashov
leads (seen 1/3 runs each). A related lead, that the adapter's RAY normalization of the Spoke's debt
value was wrong, was refuted by the annex's live read of the Main Spoke and is closed.
- **Morpho: a partial repay one wei below the reported debt can clear the whole debt.**
`expectedBorrowAssets` rounds up, so the adapter routes any request at or above it to the
share-burning full-repay path (`src/MorphoLendingAdapter.sol:139-141`), which is what closed
OpenZeppelin L-01's underflow. A request one wei below it still takes the asset path, and Morpho's
rounded-down share conversion can map that amount to every borrow share the account holds, clearing
the debt for one wei less than reported. Not reproduced against a live market for this document.
Source: Pashov lead (seen 1/3 runs); treat any repay within one wei of the reported debt as
equivalent to the full-close sentinel rather than a precise partial repay.
- **Compound: a base-token donation just below a planned borrow can make an increase revert.**
`Comet.supplyTo(victimAccount, base, x)` needs no permission; a subsequent `withdraw(base, amount)`
spends that supply first, so a donation sized just under the planned borrow can push the
post-donation borrow below `baseBorrowMin` and revert `BorrowTooSmall`. The donation itself stays
with the victim (an attacker gains nothing and pays for each attempt). Not independently
re-verified for this document. Source: Pashov lead.
- **Compound: a drifted position can reject small partial decreases even though it is not
liquidatable.** `maxLtvWad` surfaces only the liquidate collateral factor; Comet separately
re-checks the tighter borrow collateral factor on the withdraw leg of a decrease, so a position
whose LTV has drifted above the borrow factor (but remains under the liquidate factor) can reject a
partial decrease that is too small to bring it back under the borrow factor, while a full close
(which always clears to zero debt) still succeeds. This is accepted, inherent Comet behavior, not a
defect; `docs/margin-trading.md` Section 8.1 documents it and recommends sizing partial decreases
against `getAssetInfoByAddress(collateral).borrowCollateralFactor`, not `maxLtvWad`.
- **Compound: a third party can** `supplyTo` **dust of another listed collateral into the account's
asset set**, after which Comet's collateralization loop prices that asset too, so a reverting or
non-positive feed for it blocks the victim's borrows and collateral withdraws while debt is open. The
owner recovers by withdrawing the dust through the escape hatch. Venue-inherent; same root as the
base-donation item above. Source: Simao I-6. Severity Info.
- **(Fixed) Priced reads no longer revert when a Comet collateral asset is delisted**: `_assetInfo`
wraps `getAssetInfoByAddress` in try/catch (`src/CompoundV3LendingAdapter.sol:325-332`), so an
unlisted asset resolves to `MarketNotSupported` rather than an opaque revert. Source: OpenZeppelin
N-14.
- **(Fixed) The Comet base price feed is read live** through `comet.baseTokenPriceFeed()` on every
valuation (`:112-113`, `:283`) rather than cached at construction. Source: OpenZeppelin L-14.

### 6.4 Cross-cutting design decisions (accepted, not defects)

- **No governance anywhere in the suite.** No owner, no timelock, no pausable surface, no upgrade
path. A design choice, re-stated here because it means every item in this section must be fixed by
a new deployment, not a configuration change; see Section 1.2 for the migration precedent (four
superseded routers already exist for this reason).
- **ERC-20 only.** Fee-on-transfer and rebasing tokens are out of scope; the curated flows are
designed to fail closed rather than mis-account such a token, but this has not been independently
fuzz-tested for this document (Section 3.2).
- **Market and adapter selection are fully permissionless, by design.** Any venue market and any
`ILendingAdapter`-shaped contract can be named on a call; the caller (or the integrator building
the transaction) is the trust boundary, not the protocol. This is the single most load-bearing
trust assumption in the whole system (Section 4.1) and is unchanged from the architecture described
in Section 1.2, just moved from a governance-curated allowlist to caller-time validation.
- **Interest-rate, oracle, and liquidation risk is entirely the lending venue's.** The margin suite
adds no independent price source and performs no local liquidation; "the position got liquidated"
is never a margin-suite bug by itself unless the margin suite's own reads of the resulting state are
wrong (the Compound base-supply item in Section 6.3 is exactly that class of bug).

### 6.5 OpenZeppelin finding status at HEAD

Every OpenZeppelin finding, re-checked against `1c86348d` for this document. "Removed" means the code
the finding describes no longer exists.

| Finding | Status | Evidence |
| --- | --- | --- |
| M-01 cached Aave data provider | Fixed | `dataProvider()` re-resolves on every use, `src/AaveLendingAdapter.sol:104-106` |
| M-02 dust aToken blocks collateral backing | Fixed | explicit `setUserUseReserveAsCollateral`, `:134-137` (root of Simao L-4, Section 6.3) |
| M-03 absolute fill assertion | Fixed (curated flows) | live pre-unlock floor, `src/MarginRouter.sol:469-470`, `:292` (Section 6.1) |
| L-01 Morpho repay boundary reverts | Fixed | `>=` guard routes to the share path, `src/MorphoLendingAdapter.sol:137-141`; residual one-wei lead in 6.3 |
| L-02, L-03, L-06, L-11 registry lifecycle | Removed | L-03's reserve-id binding is now validated on every call (`ReserveMismatch`); L-06's substance survives as Simao I-1 (6.1) |
| L-04 unvalidated Universal Router | Open, accepted | 6.1 |
| L-05 event reads roll back mutations | Fixed | try/catch at `src/MarginRouter.sol:108`, `:229`, `:350` |
| L-07 hardcoded close event | Open | 6.1 |
| L-08 dangling Permit2 allowance | Fixed | allowance zeroed after the swap, `:611` |
| L-09 `OPEN_DELTA` supplies the full balance | Open, accepted | 6.1 |
| L-10 silent no-op close | Open | 6.1 |
| L-12 repay clamping and requested amount in the event | Open (event half) | same defect as Pashov #4 / Simao L-3 (6.1); clamping is now the share path on Morpho, Aave v3's native `repay` clamp, and the `borrowBalanceOf` cap on Compound (`:189-191`); Aave v4 not re-checked |
| L-13 Comet drifted partial decrease | Open, inherent | 6.3 |
| L-14 superseded Comet price feed | Fixed | `baseTokenPriceFeed()` read live, `src/CompoundV3LendingAdapter.sol:112-113`, `:283` |
| N-01 misconfigured Morpho market routable | Removed (transformed) | any existing Morpho market is routable by design; the caller vets it (Section 4.1) |
| N-02 Morpho views price collateral on a debt-free read | Open, narrowed | `_positionValues` still calls `IOracle.price()` unconditionally, `src/MorphoLendingAdapter.sol:220`; since every event read is best-effort, the only effect is a skipped event |
| N-03, N-08, N-12, N-16 governance and registry | Removed | no owner, no registry, no storage libraries |
| N-04 shadowed constructor parameter | Fixed | parameters suffixed `_`, `src/MarginRouter.sol:87` |
| N-05 `CollateralWithdrawn` logs zero | Open | 6.2 |
| N-06 unused named returns in the decoder | Open, cosmetic | 7 of 8 decoders still shadow a named return with an explicit `return abi.decode(...)` |
| N-07 unused code | Partially fixed | `Ltv.raw`/`Ltv.lte` are gone (`src/types/Ltv.sol` now attaches only `gt`); the Morpho adapter's import list was not re-audited |
| N-09 undefined opcode misreported as `NoActiveAccount` | Fixed | `_handleAction` tests `isAccountScoped` before the active-account guard, `src/MarginRouter.sol:518-526` |
| N-10 zero overloaded as a control value | Partially fixed | `LtvBoundRequired` added; the rest is accepted design (6.1) |
| N-11 native sweep to the manager reverts | Open, unreachable in-tree | 6.2 |
| N-13 Comet base supply unreachable through the router | Open | same as Simao L-1 (6.3) |
| N-14 priced reads revert on a delisted Comet asset | Fixed | 6.3 |
| N-15 inaccurate documentation | Partially fixed | the `execute` NatSpec now states that plans emit `PositionUpdated` (item 8 of the `execute` NatSpec in `src/interfaces/IMarginRouter.sol`); the `_readPosition` "never market-gated" claim is still wrong (Simao I-1, 6.1) |

## 7. Build, Test & PoC Setup

Build:

```bash
forge build
```

The default profile compiles with `via_ir = true` and `optimizer_runs = 44444444`; `MarginRouter.sol`
and a few other large contracts are pinned to their own `optimizer_runs` via
`compilation_restrictions` in `foundry.toml` to fit under the EIP-170 runtime-size limit, and all of
`test/**` compiles with `via_ir = false`. This mixed-profile setup is deliberate (margin test helpers
deploy `MarginRouter` and `UniversalRouter` from their own compiled artifacts via `vm.getCode(...)`
specifically so the router's via-ir source is never pulled into a test's non-via-ir compilation unit;
see `test/shared/MarginRouteHelpers.sol`), so do not "simplify" it by forcing one `via_ir` setting
repo-wide.

Run the margin-scoped test suite (account, router, adapter unit and fuzz, library, integration,
deploy):

```bash
forge test --match-path "test/{marginAccount,marginRouter,lending,libraries,integration,deploy}/**"
```

Run the invariant suite:

```bash
forge test --match-path "test/invariant/MarginRouterInvariant.t.sol"
```

Four properties (`invariant_routerHoldsNoLooseFunds`, `invariant_accountsHoldNoLooseTokens`,
`invariant_onBehalfIsAlwaysAnAccount`, `invariant_accountIsolation`), driven by
`test/invariant/MarginRouterHandler.sol` over 256 runs x 128 depth (`[invariant]` in `foundry.toml`).
The handler currently drives only happy-path state transitions (0 reverts across a full run), so it
does not itself probe the revert/boundary conditions enumerated in Section 6; those are covered
separately by the unit and integration suites' explicit revert-path tests.

Run the mainnet-fork suite (16 tests across 10 files under `test/fork/`: 9 router-level tests in the
six `MarginRouter*.fork.t.sol` files and 7 adapter-level tests in the four `*LendingAdapter.fork.t.sol`
files). Every fork test reads `MAINNET_RPC_URL` (not `ETH_RPC_URL`) and self-skips via
`vm.skip(bytes(rpc).length == 0)` when it is unset; there is no `.env.example` in the repository.
Each file pins its own `FORK_BLOCK`: 25,319,047 (Morpho, Aave v3, and the router E2E, hedge, and
execute files), 25,330,047 (Aave v4), and 25,598,384 (Compound and the route-swap file).

```bash
export MAINNET_RPC_URL=<a mainnet archive RPC URL>
forge test --match-path "test/fork/**"
```

`forge coverage` **is not reliable for this repository and its numbers should not be quoted.**
Coverage instrumentation requires disabling `via_ir`/optimizer settings project-wide (or using
`--ir-minimum`, whose source maps are independently known to misattribute lines here), and this
repository has dependencies (for example `lib/universal-router`) that only compile with `via_ir`
enabled, so a whole-project `via_ir = false` coverage run fails to compile, and a per-path override
narrow enough to avoid that produces an "incompatible settings restrictions" error from Foundry's
compiler-profile resolver whenever a test file imports both a `via_ir = false` and a `via_ir = true`
path in the same compilation unit (as the shared margin test helpers do). Reason about coverage
structurally (which source paths how many tests exercise, enumerated by file in Section 2 and
Section 7) rather than from a quoted percentage.

Integer overflow: Solidity `0.8.26` checked arithmetic throughout; no `unchecked` block exists
anywhere in the scoped margin `src/` files, and the only inline assembly is the `memory-safe`
`tstore`/`tload` pair in `src/libraries/TransientPlanState.sol` (both verified by direct search).

PoC scaffold, local pool (fast; exercises the router/account wiring end-to-end against a mock
lending protocol standing in for any venue):

```solidity
// test/integration/MarginRouterIntegration.t.sol sets up: a local PoolManager with a deep 1:1 pool
// (setupRouterCurrenciesAndPoolsWithLiquidity, from RoutingTestHelpers), a UniversalRouter bound to
// it (MarginRouteHelpers.deployUniversalRouter), a MarginRouter deployed from its own artifact
// (MarginRouteHelpers.deployMarginRouter), and a MockLendingAdapter/MockLendingProtocol pair from
// test/mocks/. DeployPermit2 is permit2/test/utils/DeployPermit2.sol.
contract PoC is RoutingTestHelpers, MarginRouteHelpers, DeployPermit2 {
    function setUp() public {
        // mirror MarginRouterIntegrationTest.setUp(): setupRouterCurrenciesAndPoolsWithLiquidity(),
        // deploy the mock adapter/protocol, deployPermit2(), deployUniversalRouter(...),
        // deployMarginRouter(...)
    }

    function test_PoC() public {
        // buildV4ExactOutRoute(poolKey, input, output, amountOut, maxIn, recipient[, minHopPriceX36])
        // builds the UR route (input = debt, output = collateral, recipient = account for an increase);
        // drive increasePosition/decreasePosition/addCollateral/execute as in Section 1.4 and assert
        // the resulting loss or violated invariant.
    }
}
```

PoC scaffold, mainnet fork (required for a finding against real venue behavior, such as anything in
Section 6.3):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.26;

import {Test, console2} from "forge-std/Test.sol";
import {MarginRouteHelpers} from "../shared/MarginRouteHelpers.sol";
// + the real venue interfaces/adapter and src/MarginAccount.sol, as in
//   test/fork/MarginRouterE2E.fork.t.sol

contract PoC is Test, MarginRouteHelpers {
    function setUp() public {
        string memory rpc = vm.envOr("MAINNET_RPC_URL", string(""));
        vm.skip(bytes(rpc).length == 0);
        vm.createSelectFork(rpc, /* pin a block number used by a prior audit for reproducibility,
                                    e.g. 25,598,384 for the Compound absorption scenario in 6.3 */);
        // deploy MarginRouter against the real mainnet PoolManager/Permit2/WETH9 and the real
        // adapter under test (deployMarginRouter, as in the local-pool scaffold above)
    }

    function test_PoC() public {
        // drive the real curated flow(s), then assert the loss/violated invariant with console2.log
    }
}
```

Run a fork PoC:

```bash
MAINNET_RPC_URL=<rpc> forge test --match-test test_PoC -vvvv
```