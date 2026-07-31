# Pons V2 — Genesis Example

> The first Genesis example built to demonstrate the Pons V2 launch architecture.

---

## ⚠️ Disclaimer

**Pons V2 Genesis is a test token created exclusively for demonstration, experimentation and documentation purposes.**

It has **no intrinsic utility** and should not be interpreted as an official production token, financial instrument, investment opportunity or representation of future protocol functionality.

The purpose of this Genesis example is to give the public a concrete, onchain representation of how the **Pons V2 launch architecture** is designed to work.

The token is used as a reference implementation to demonstrate:

* fixed-supply token deployment;
* bonding-curve trading;
* phantom quote reserves;
* launch fees;
* creator fees;
* protocol fee allocation;
* buyback configuration;
* graduation mechanics;
* Uniswap v4 pool creation;
* post-graduation trading fees;
* fee escrow;
* buyback supply locking and vesting;
* permanently locked liquidity.

This is a **technical demonstration of the architecture**, not a production deployment.

---

# 1. What is Pons V2?

Pons V2 introduces a launch architecture built around a **bonding curve → graduation → Uniswap v4** lifecycle.

A launch begins on a dedicated bonding curve.

Users can purchase the newly created token while the curve is active. The token supply is fixed from the moment of deployment and the entire supply is initially minted directly to the bonding curve.

Once the graduation threshold is reached, the launch transitions toward a Uniswap v4 pool.

After graduation, the system introduces a second phase:

```text
                 PONS V2 GENESIS
                       │
                       ▼
                LaunchFactory
                       │
                       ▼
                 LaunchToken
                       │
                       ▼
                BondingCurve
                       │
              threshold reached
                       │
                       ▼
                  Graduation
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       Uniswap v4 Pool       LaunchLocker
             │
             ▼
          MemeHook
             │
             ▼
         Trading Fees
             │
       ┌─────┴─────┐
       ▼           ▼
  FeeEscrow   BuybackVault
                   │
                   ▼
            Locked Buyback Supply
              + Linear Vesting
```

The Genesis example is designed to expose this entire architecture in a single reference launch.

---

# 2. The Genesis Token

The Genesis token is a standard ERC-20 created through the Pons V2 launch architecture.

Unlike many token launch mechanisms, the token contract itself is intentionally minimal.

The token has:

* a fixed total supply;
* no additional mint function;
* no blacklist;
* no pause mechanism;
* no owner-controlled token switch;
* no creator-controlled supply allocation.

The entire initial supply is minted directly to the launch's `BondingCurve`.

```solidity
_mint(curve_, totalSupply_);
```

This means the deployer does not receive a pre-mined allocation simply by creating the token.

The creator address is stored for informational purposes:

```solidity
address public immutable tokenDeployer;
```

However, the creator has no administrative privileges over the ERC-20 itself.

---

# 3. Launch Architecture

The Pons V2 architecture is composed of several contracts.

| Contract        | Purpose                                                  |
| --------------- | -------------------------------------------------------- |
| `LaunchFactory` | Main entry point for launches and protocol coordination  |
| `LaunchToken`   | Fixed-supply ERC-20 deployed per launch                  |
| `BondingCurve`  | Initial trading mechanism                                |
| `MemeHook`      | Uniswap v4 hook for post-graduation trading fees         |
| `FeeEscrow`     | Pull-based accounting for accumulated fees               |
| `BuybackVault`  | Holds bought-back token supply and manages vesting       |
| `LaunchLocker`  | Permanently locks graduation liquidity and excess supply |

The Genesis launch therefore represents an entire protocol flow rather than a standalone ERC-20 deployment.

---

# 4. Step One — Launch Creation

The launch begins through `LaunchFactory`.

A launcher provides the token metadata and launch parameters.

The launch can include:

```solidity
struct TokenParams {
    string name;
    string symbol;
    string logo;
    string description;
    LaunchToken.Socials socials;
    address creatorFeeRecipient;
    uint16 creatorTaxBps;
    bool buybackEnabled;
    bytes32 expectedEconomics;
}
```

This allows the launch to define:

* token name;
* token symbol;
* logo;
* description;
* social links;
* creator fee recipient;
* creator tax;
* buyback activation;
* expected launch economics.

The factory also verifies that the economic parameters expected by the launcher match the configuration stored by the protocol.

This prevents a launch from being created under silently modified parameters.

The economic configuration can be previewed through:

```solidity
previewLaunchEconomics(
    uint256 launchConfigId,
    address pairToken
)
```

The resulting hash includes important parameters such as:

* total supply;
* curve fee;
* phantom quote;
* graduation threshold;
* pool fee;
* tick spacing;
* pair token;
* maximum creator tax.

The launcher must provide the matching `expectedEconomics`.

---

# 5. Fixed Supply Token

Once the launch is accepted, the factory deploys a new `LaunchToken`.

The complete supply is minted directly to the bonding curve.

Conceptually:

```text
LaunchFactory
      │
      ├── deploys LaunchToken
      │
      └── deploys BondingCurve
                  │
                  ▼
          Entire token supply
```

The Genesis token therefore begins with:

```text
Total Supply
    │
    ▼
Bonding Curve
    │
    ├── Tokens sold through curve
    │
    └── Remaining tokens
            │
            ▼
       Graduation flow
```

There is no creator pre-mine in the token contract.

---

# 6. The Bonding Curve

The initial market is powered by a constant-product bonding curve.

The curve uses a virtual, or **phantom**, quote reserve.

The invariant is conceptually:

```text
k = phantomQuote × totalSupply

quoteReserve × tokenReserve = k
```

At launch:

```text
quoteReserve = phantomQuote
tokenReserve = totalSupply
```

The phantom reserve creates an initial non-zero price without requiring the launch to begin with the entire graduation threshold already deposited.

The graduation point is determined by the real quote reserve reaching the configured threshold.

---

# 7. Phantom Quote Reserve

The Genesis example uses the following relationship:

```text
reservedTokens =
    totalSupply × phantomQuote
    ────────────────────────────
    phantomQuote + graduationThreshold
```

This represents the amount of token supply remaining in the bonding curve when the real quote reserve reaches the graduation threshold.

For example, conceptually:

```text
Total Supply:          1,000,000,000 tokens
Phantom Quote:         2 ETH
Graduation Threshold:  10 ETH
```

Then:

```text
reservedTokens =
1,000,000,000 × 2
─────────────────
       12
```

The bonding curve therefore provides a mathematically consistent relationship between:

* quote raised;
* tokens sold;
* remaining supply;
* graduation progress.

---

# 8. Buying From the Curve

While the curve is active, users can call:

```solidity
buy(
    uint256 quoteIn,
    uint256 minTokensOut,
    address recipient
)
```

The buyer provides the quote asset and receives tokens based on the current curve price.

The function also applies the configured base fee.

The contract emits a `CurveBuy` event containing the relevant trade information, including the fee.

A simplified flow is:

```text
User
 │
 │ quote asset
 ▼
BondingCurve
 │
 ├── calculate fee
 │
 ├── update reserves
 │
 ├── calculate token output
 │
 ├── send tokens to recipient
 │
 └── account for protocol / buyback fees
```

The user can protect the trade by specifying a minimum acceptable token output.

---

# 9. Oversized Buys and Graduation

The curve also handles a purchase that would push the launch beyond its available curve inventory.

Instead of allowing the curve to oversell its remaining supply, the purchase is clamped to the amount required to complete graduation.

Any excess quote is refunded.

The flow becomes:

```text
Large Buy
    │
    ▼
Calculate maximum available curve purchase
    │
    ▼
Fill remaining curve inventory
    │
    ▼
Mark curve graduated
    │
    ▼
Refund excess quote
```

This behavior is represented by the `CurveBuyRefunded` event.

Once graduated, additional curve trades are rejected.

The curve therefore has a clear state transition:

```text
ACTIVE
  │
  │ graduation threshold reached
  ▼
GRADUATED
```

A graduated curve cannot continue accepting normal bonding-curve trades.

---

# 10. Fees

Pons V2 separates the fee system into several components.

The curve configuration can define:

* base curve fee;
* creator tax;
* protocol fee share;
* buyback share;
* buyback activation.

The protocol fee policy contains parameters such as:

```solidity
struct FeePolicy {
    address protocolFeeRecipient;
    uint16 protocolFeeShareBps;
    uint16 buybackBurnBps;
    uint16 hookFeeBps;
    uint16 maxInternalPriceImpactBps;
}
```

Basis points use:

```text
10,000 BPS = 100%

1 BPS = 0.01%
```

For example:

```text
100 BPS = 1%
500 BPS = 5%
1,000 BPS = 10%
```

The fee system is therefore configurable at the protocol level rather than hard-coded into the token itself.

---

# 11. Buyback Mechanics

One of the central components demonstrated by the Genesis example is the **BuybackVault**.

The BuybackVault is designed to hold tokens acquired through the protocol's buyback mechanism.

Instead of treating bought-back tokens as immediately circulating supply, the vault locks them and applies a vesting schedule.

The architecture is:

```text
Trading Activity
      │
      ▼
Fee Generation
      │
      ▼
Buyback Allocation
      │
      ▼
Buyback Execution
      │
      ▼
Tokens Acquired
      │
      ▼
BuybackVault
      │
      ▼
5-Year Linear Vesting
```

The vault tracks the total amount locked for each token.

The core storage concept is:

```solidity
totalLocked(token)
```

The vault can also calculate the currently releasable amount:

```solidity
releasable(token)
```

---

# 12. The Buyback Wallet

The Genesis deployment explicitly exposes the address associated with the buyback infrastructure.

The relevant architecture is:

```text
                  Pons V2 Genesis
                         │
                         ▼
                  Buyback Mechanism
                         │
                         ▼
                   Buyback Wallet
                         │
                         ▼
                   BuybackVault
                         │
                         ▼
                 Locked Token Supply
```

The buyback wallet should be understood as part of the protocol's buyback flow rather than as a discretionary creator wallet.

The purpose of the wallet/vault path is to make the destination of buyback-related token supply observable and auditable.

The Genesis example therefore allows users to inspect:

* the buyback infrastructure address;
* the vault contract;
* tokens locked in the vault;
* total locked supply;
* vesting state;
* releasable supply.

The exact deployed address for the Genesis instance should be published alongside this document once the deployment is finalized.

```text
Genesis Buyback Wallet:
[DEPLOYED BUYBACK ADDRESS]

BuybackVault:
[DEPLOYED VAULT ADDRESS]
```

These addresses are intentionally left as deployment-specific values rather than hard-coded into the reference documentation.

---

# 13. Buyback Vesting

The BuybackVault uses a linear vesting model.

The reference implementation defines a five-year vesting period.

Conceptually:

```text
100% Buyback Supply
        │
        ▼
      Locked
        │
        │ 5 years
        ▼
  Gradually Releasable
```

When additional tokens are locked, the vault recomputes the effective vesting start using weighted accounting.

This means repeated buyback deposits do not simply create unrelated vesting schedules.

The vault maintains a combined vesting state for the token.

At the moment tokens are initially locked:

```text
releasable(token) = 0
```

The supply begins becoming releasable progressively according to the vesting schedule.

---

# 14. FeeEscrow

The `FeeEscrow` contract provides pull-based fee accounting.

Rather than relying on a single sweep operation that sends funds to every recipient automatically, the system records balances that can subsequently be claimed.

This architecture reduces the risk that a problematic recipient can block the distribution of fees to everyone else.

The conceptual model is:

```text
Fee Generated
     │
     ▼
FeeEscrow
     │
     ├── Protocol Balance
     │
     ├── Creator Balance
     │
     └── Other Authorized Balance
```

Authorized contracts can credit balances into the escrow.

Recipients can then pull their accumulated funds.

---

# 15. Graduation

Graduation is the transition between the bonding curve and the post-graduation market.

The conceptual lifecycle is:

```text
          LAUNCH
             │
             ▼
       BONDING CURVE
             │
             │ quote threshold reached
             ▼
         GRADUATION
             │
      ┌──────┴──────┐
      ▼             ▼
  Uniswap v4    LaunchLocker
     Pool       Liquidity
      │
      ▼
   MemeHook
      │
      ▼
 Post-Graduation
    Trading
```

The factory coordinates the graduation process.

At this stage, the system prepares:

* the remaining token supply;
* the quote asset;
* the initial pool price;
* the liquidity position;
* the Uniswap v4 pool;
* the post-graduation hook.

The `SqrtPrice` library derives the initial square-root price from the token and quote amounts.

This is intended to ensure that the graduated pool opens around the price implied by the final state of the bonding curve rather than an arbitrary starting price.

---

# 16. Uniswap v4 Integration

The post-graduation market is designed around Uniswap v4.

The `MemeHook` contract acts as the singleton hook used by the graduated pools.

Its role includes handling the post-graduation trading fee and sweeping the accumulated fee flow into the protocol's accounting system.

The high-level flow is:

```text
User Trade
    │
    ▼
Uniswap v4 Pool
    │
    ▼
MemeHook
    │
    ├── Calculate Hook Fee
    │
    ├── Collect / Sweep
    │
    ▼
FeeEscrow
    │
    └── Protocol / Buyback Accounting
```

The hook therefore represents the bridge between the Uniswap v4 trading environment and the Pons V2 fee architecture.

---

# 17. Buyback Flow After Graduation

Once the token has graduated, the protocol can continue to generate fees through post-graduation trading.

The conceptual buyback loop becomes:

```text
Token Trading
      │
      ▼
MemeHook
      │
      ▼
Fee Collection
      │
      ▼
FeeEscrow
      │
      ▼
Buyback Allocation
      │
      ▼
Buyback Execution
      │
      ▼
Launch Token Acquired
      │
      ▼
BuybackVault
      │
      ▼
5-Year Linear Vesting
```

The important distinction is that the Genesis token itself does not contain the buyback logic.

The token is deliberately simple.

The buyback architecture exists outside the ERC-20, in the protocol layer.

---

# 18. LaunchLocker

The `LaunchLocker` is responsible for holding graduation liquidity and excess launch-token supply.

The design intentionally exposes no withdrawal mechanism.

There is:

* no creator withdrawal;
* no protocol withdrawal;
* no privileged unlock;
* no owner-controlled liquidity removal function.

The intention is to make permanent liquidity locking a property of the contract architecture itself.

At graduation:

```text
Graduation Assets
      │
      ├── Liquidity
      │       │
      │       ▼
      │   LaunchLocker
      │
      └── Excess Token Supply
              │
              ▼
          LaunchLocker
```

The contract does not provide a corresponding withdrawal path.

This is intended to make the statement "liquidity is permanently locked" enforceable by code rather than dependent on a promise.

---

# 19. LaunchLocker Excess Supply

The locker can receive excess token supply through:

```solidity
lockExcessSupply(
    address token,
    uint256 amount
)
```

The tokens are transferred into the locker.

There is intentionally no function that allows the tokens to be withdrawn later.

This creates an irreversible destination for supply that remains after the graduation process.

---

# 20. Launch Token Metadata

The Genesis token stores launch metadata directly in the token contract.

This includes:

```text
Token Name
Token Symbol
Token Logo
Token Description
Twitter
Telegram
Discord
Website
Farcaster
```

The metadata can be retrieved through:

```solidity
getTokenInfo()
```

The returned information includes:

```solidity
(
    address deployer,
    string logo,
    string description,
    Socials socials
)
```

The deployer address is informational only.

It does not provide administrative control over the token.

---

# 21. Main Solidity Functions

## LaunchFactory

The factory is the main protocol entry point.

### `launchToken()`

Creates a new launch.

The function:

1. validates launch permissions;
2. validates the launch fee;
3. validates creator tax limits;
4. validates the selected launch configuration;
5. validates the quote token;
6. verifies quote-token decimals;
7. verifies the expected economic configuration;
8. deploys a new `BondingCurve`;
9. deploys a new `LaunchToken`;
10. connects the token and curve;
11. registers the launch.

---

### `previewLaunchEconomics()`

Returns a hash representing the expected economic configuration of a launch.

This allows a launcher to verify that the parameters used during deployment are exactly the ones expected.

---

### `createGraduatedPool()`

Coordinates the creation of the post-graduation Uniswap v4 market.

The function is responsible for moving the launch from its bonding curve phase into its graduated pool architecture.

---

### `rescueLaunch()`

Provides a fallback mechanism for a launch that becomes stuck during the graduation process.

The current reference implementation returns swept quote assets to the original deployer as a simplified recovery mechanism.

This should be considered a reference implementation limitation rather than a final production refund design.

---

### Community Takeover Functions

The factory also contains governance paths for community takeover of launches.

These mechanisms are part of the factory-level architecture rather than the token contract.

The important design principle is that the ERC-20 itself remains immutable and free from privileged administrative controls.

---

# 22. BondingCurve Functions

The curve handles the initial market.

Key functions include:

```solidity
buy(
    uint256 quoteIn,
    uint256 minTokensOut,
    address recipient
)
```

Used to purchase launch tokens.

```solidity
sell(
    uint256 tokenIn,
    uint256 minQuoteOut,
    address recipient
)
```

Used to sell tokens back into the curve while the curve is active.

```solidity
getReserves()
```

Returns the current quote and token reserves.

```solidity
reservedTokens()
```

Returns the amount of tokens mathematically reserved for the graduation state.

```solidity
graduated()
```

Returns whether the curve has completed its active phase.

```solidity
readyToGraduate()
```

Indicates whether the curve has reached the conditions required for graduation.

---

# 23. BuybackVault Functions

The BuybackVault manages the supply acquired through the buyback system.

Important functions include:

```solidity
lock(
    address token,
    uint256 amount
)
```

Locks token supply inside the vault.

```solidity
totalLocked(
    address token
)
```

Returns the total amount currently accounted for as locked.

```solidity
releasable(
    address token
)
```

Returns the amount that has become releasable according to the vesting schedule.

The vault is designed around a five-year linear vesting model.

---

# 24. FeeEscrow Functions

The escrow provides pull-based accounting for fee balances.

Its architecture is designed to allow authorized protocol components to credit balances without relying on a single global distribution transaction.

This separates:

```text
Fee Generation
```

from:

```text
Fee Claiming
```

This is particularly relevant when multiple recipients or assets are involved.

---

# 25. Deployment Architecture

The reference deployment script demonstrates the intended dependency order.

The system is wired approximately as follows:

```text
Existing Uniswap v4 PoolManager
            │
            ▼
       FeeEscrow
            │
            ▼
      BuybackVault
            │
            ▼
      LaunchLocker
            │
            ▼
        MemeHook
            │
            ▼
      LaunchFactory
```

The factory subsequently becomes responsible for registering and authorizing the per-launch curves.

The deployment process also configures authorization between:

* `MemeHook`;
* `FeeEscrow`;
* `BuybackVault`;
* `LaunchFactory`;
* individual `BondingCurve` instances.

---

# 26. Example Deployment Flow

The reference deployment script follows this general sequence:

```text
1. Connect to an existing Uniswap v4 PoolManager

2. Deploy FeeEscrow

3. Deploy BuybackVault

4. Deploy LaunchLocker

5. Deploy MemeHook

6. Deploy LaunchFactory

7. Authorize the hook on FeeEscrow

8. Authorize the vault on FeeEscrow

9. Authorize the hook on BuybackVault

10. Transfer ownership of FeeEscrow to LaunchFactory

11. Transfer ownership of BuybackVault to LaunchFactory

12. LaunchFactory authorizes individual curves
```

The deployment script is intentionally documented as an **illustrative wiring script**.

It is not currently a turnkey production deployment script.

---

# 27. Genesis Walkthrough

The Pons V2 Genesis example can therefore be understood as the following lifecycle.

### Phase 1 — Creation

A launch configuration is selected.

The creator provides the token metadata and launch parameters.

```text
LaunchFactory
      │
      ├── validates configuration
      ├── validates economics
      ├── deploys token
      └── deploys curve
```

---

### Phase 2 — Bonding Curve

The token begins trading through the bonding curve.

```text
User
  │
  ▼
BondingCurve
  │
  ├── fee
  ├── price calculation
  ├── token output
  └── reserve update
```

The phantom quote reserve determines the initial pricing behavior.

---

### Phase 3 — Graduation

The real quote reserve reaches the graduation threshold.

The curve becomes graduated.

Any oversized final purchase is clamped and excess funds are refunded.

```text
ACTIVE CURVE
     │
     ▼
THRESHOLD REACHED
     │
     ▼
GRADUATED
```

---

### Phase 4 — Liquidity

The remaining launch assets are prepared for the graduated market.

Liquidity is seeded into the Uniswap v4 pool.

The liquidity position is attributed to `LaunchLocker`.

The locker has no withdrawal path.

```text
Liquidity
    │
    ▼
LaunchLocker
    │
    ▼
Permanent Lock
```

---

### Phase 5 — Post-Graduation Trading

Trading continues through the Uniswap v4 pool.

The `MemeHook` handles the configured post-graduation trade fee.

```text
Trader
  │
  ▼
Uniswap v4
  │
  ▼
MemeHook
  │
  ▼
FeeEscrow
```

---

### Phase 6 — Buyback

The protocol's buyback allocation can be used to acquire the launch token.

The resulting supply is routed into the buyback infrastructure.

```text
Buyback
   │
   ▼
BuybackVault
```

---

### Phase 7 — Vesting

The acquired token supply remains subject to the vault's vesting mechanism.

The reference implementation uses:

```text
5-Year Linear Vesting
```

The locked supply gradually becomes releasable over time.

---

# 28. The Full Pons V2 Genesis Flow

```text
                       PONS V2 GENESIS
                              │
                              ▼
                       LaunchFactory
                              │
                  ┌───────────┴───────────┐
                  ▼                       ▼
             LaunchToken             BondingCurve
                  │                       │
                  │                       │
                  │                  Buy / Sell
                  │                       │
                  │                       ▼
                  │                Graduation
                  │                       │
                  │              ┌────────┴────────┐
                  │              ▼                 ▼
                  │        Uniswap v4        LaunchLocker
                  │              │                 │
                  │              ▼                 │
                  │          MemeHook              │
                  │              │                 │
                  │              ▼                 │
                  │         FeeEscrow              │
                  │              │                 │
                  │              ▼                 │
                  │        Buyback Flow             │
                  │              │                 │
                  │              ▼                 │
                  │        BuybackVault ◄──────────┘
                  │              │
                  │              ▼
                  │       5-Year Vesting
                  │
                  ▼
             Fixed Supply
```

---

# 29. Why This Is the Genesis Example

Pons V2 Genesis is intended to be the first public example that makes the V2 architecture tangible.

Instead of describing the protocol only through abstract documentation, the Genesis example provides a concrete token through which users can inspect the architecture directly.

The goal is to allow anyone to follow the lifecycle:

```text
Token Creation
      ↓
Bonding Curve
      ↓
Graduation
      ↓
Uniswap v4
      ↓
Hook Fees
      ↓
Fee Escrow
      ↓
Buyback Infrastructure
      ↓
BuybackVault
      ↓
Vesting
```

The Genesis token is therefore best understood as a **living technical demonstration** of the Pons V2 architecture.

It is not intended to introduce a new utility token.

It is intended to demonstrate the protocol.

---

# 30. Important Reference Implementation Notes

This repository is an independent reference implementation of the Pons V2 architecture and is explicitly marked as unaudited.

Several components should therefore not be interpreted as production-ready without further work.

### Uniswap v4 Hook Address Mining

Uniswap v4 determines hook permissions from the deployed hook address.

A production deployment requires deterministic deployment or CREATE2 salt mining to ensure the `MemeHook` address satisfies the required permission flags.

The current deployment script does not implement this mining process.

---

### Factory Address Prediction

The deployment script currently predicts the `LaunchFactory` address using a nonce offset.

This is fragile and should be replaced with a deterministic deployment mechanism before use on a live network.

---

### Internal Graduation Conversion

The hook's internal conversion path includes a spot-price-impact check after executing the swap.

A production implementation should quote and validate the expected impact before committing to the swap.

---

### Rescue Launch

The current rescue mechanism returns swept quote assets to the original deployer.

A more complete production implementation could instead maintain historical per-buyer contribution accounting and distribute recovered assets proportionally.

---

### Audit Status

This Genesis example should therefore be treated as:

```text
REFERENCE IMPLEMENTATION
        +
TECHNICAL DEMONSTRATION
        +
PUBLIC ARCHITECTURE EXAMPLE
```

and **not** as:

```text
AUDITED PRODUCTION PROTOCOL
```

---

# 31. Final Disclaimer

**Pons V2 Genesis is a test token.**

It has no intrinsic utility.

It is not designed or presented as an investment.

It does not represent a promise of future protocol functionality.

Its purpose is to provide the public with a concrete example of the Pons V2 architecture and demonstrate how the protocol's contracts can work together across:

* token creation;
* bonding curves;
* graduation;
* Uniswap v4;
* fee collection;
* buybacks;
* vesting;
* liquidity locking.

The Genesis example exists to make the technology visible.

**The token is the example. The architecture is the point.**
