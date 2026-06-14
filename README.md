# Juncta on Soroban

Concentrated liquidity and RWA-native lending for Stellar.

This repository holds the Soroban implementation of Juncta's core protocol: a concentrated liquidity DLMM and an integrated lending market that share one capital layer. The work in this repository is the subject of our Stellar Community Fund Build submission. The contracts are in active planning and the implementation begins with this grant. This README documents the architecture, the module structure, and the milestone plan so reviewers can see the full scope before the first line of contract code lands.

Our full project documentation lives in one source: https://juncta.gitbook.io/juncta-docs. It contains the complete technical architecture and diagrams, the market analysis and competitive positioning, the roadmap and milestones, the test plan, and the development artifacts.

---

## What Juncta Is

Juncta is a capital market protocol built on a unified liquidity layer. A single deposit into the DLMM does two jobs at once. It provides trading liquidity when the price is in range, and its idle reserves supply the lending market when the price is not. The same capital earns swap fees and lending interest instead of sitting still.

On Stellar specifically, Juncta is the capital market layer for the real-world assets already issued on the chain. Stellar has the highest density of institutional RWA issuers in the ecosystem. Franklin Templeton's BENJI holds its largest single-chain deployment here. Société Générale Forge, WisdomTree, and Paxos are all active. Those assets earn their native yield and little else today. Juncta gives their holders borrowing, liquidity provision, and stacked yield, with a liquidation system built for assets that have slow-moving price feeds.

---

## The Novel Primitive

Two pieces of existing Stellar infrastructure are worth naming. Aquarius provides AMM liquidity. Blend provides lending. Both are good at what they do. Neither connects the two so that the same capital serves both functions, and neither is built for the price behavior of tokenized real-world assets.

Juncta brings three things that do not exist on Soroban today:

1. **Idle-bin lending.** Liquidity sitting outside the active price range in a concentrated pool becomes the supply side of the lending market. There is no separate lending vault. On deposit, tokens go straight from the LP into the bin reserves, and an accounting record marks how much of each bin's reserve is available to lend. Tokens that no one has borrowed stay in the bin and remain available for swaps. When a borrower draws against the supply, real tokens leave the specific bins they were allocated from and move to the borrower, while the position records the debt and the interest owed. When the borrower repays, the tokens are credited back to the exact bins they came from. A liquid buffer near the active price, plus a reserve kept in every bin, means swaps in the active trading range are never blocked by lending activity.

2. **Aggregated lending markets.** Each token has one lending market, not one per trading pair. Every DLMM pool that holds a given token contributes its idle reserves of that token into a single global supply pool for that token. A borrower sees one market, one utilization number, and one rate per token, and can borrow against the combined depth of every contributing pool. This removes the liquidity fragmentation and rate divergence that come from isolated per-pair lending, and it makes recall resolve faster because a repayment from any borrower of a token helps every pool that supplies it.

3. **Progressive liquidation for RWA collateral.** Standard lending protocols close a large share of a position the moment the health factor drops below one. That model is built for crypto tick oracles that update every second. Tokenized treasury and real estate feeds update slower. A routine data lag can wipe a healthy borrower. Juncta replaces the cliff with a five-level system that closes only the minimum needed to restore health and gives borrowers recovery windows to react.

---

## Architecture Overview

### Unified Liquidity Layer

The lending market is an accounting layer over the DLMM state, not a separate pool that tokens flow into. Each bin holds its reserves. A deployment record tracks how much of each bin is currently serving as lending supply. Liquidity is never in two places at once, and the swap path always has first claim on the capital it needs. Tokens only leave a bin when a borrower actually borrows, and they return to the same bin when the borrower repays.

### The Inactive Bin Router

The router is the bridge between the DLMM and the lending market. It runs inside the swap path, and whenever a trade moves the active bin, it checks which bins have crossed a tier boundary and updates their lending deployment. Bins that move farther from the active price supply more capital to lending. Bins that move closer have capital recalled. Because only the bins at a tier boundary change, the work per swap is small and bounded regardless of pool size.

### Global Token Supply Pool

Each token has one global supply pool. Every DLMM pool that holds that token feeds its idle reserves of that token into it. The global pool tracks total supply, total borrowed, idle capital, utilization, and one borrow rate for the whole token. Borrowing draws from this combined depth, and the interest rate is a single function of the token's global utilization rather than a per-pair number. When capital needs to be recalled and the pool is fully utilized, the unmet portion is flagged as pending recall, new borrows of that token pause, and the flag clears as borrowers repay. No loan is ever force-closed to satisfy a recall.

### Deployment Tier Model

Bins are made available to lending based on their distance from the active price. Bins close to the active price stay fully liquid for trading. Bins further out, which are less likely to trade soon, supply more of their reserves to lending.

| Distance from active bin | Reserve available to lending |
|---|---|
| 0 to 2 bins | 0 percent, always fully liquid |
| 3 to 5 bins | 20 percent |
| 6 to 10 bins | 50 percent |
| 11 to 20 bins | 70 percent |
| 21 or more bins | 80 percent, with 20 percent always held in reserve |

A buffer zone of bins nearest the active price is never deployed under any condition, so a sudden large price move always meets full liquidity. The 20 percent held back in the farthest tier is an instant reserve that absorbs withdrawals and repayment spikes without touching open loans.

### How Capital Moves

A new pool does not supply lending right away. It first runs as a plain DEX and becomes eligible for lending only after it clears an activation gate: a minimum seven-day volume, a minimum total value locked, a minimum number of unique traders, and a price oracle that is at least 72 hours old. This makes sure that only pools with real, sustained activity and a mature price feed can feed the lending market, which keeps thin or easily manipulated pools out of it.

Once a pool is lending-active, the full lifecycle runs end to end:

1. An LP deposits into a pool. Tokens go straight into the bin reserves and an LP position is recorded.
2. The router assigns each bin a deployment tier based on its distance from the active price, and the deployed share is contributed to that token's global supply pool. The tokens stay in the bins. Only accounting changes.
3. A borrower borrows from the global pool. The allocation logic draws from the farthest bins first, since those are least likely to be needed for trading soon. Real tokens move from those bins to the borrower, and the exact bins debited are recorded on the borrow position.
4. As price moves, the router recalls capital from bins approaching the active price. If the global pool has idle capital, recall is instant. If not, the shortfall becomes pending recall and clears as repayments arrive.
5. The borrower repays. Tokens are credited back to the exact bins they were drawn from, the pool's contribution is restored, and any pending recall is satisfied first.

### The Three Borrower Types

- **Separate collateral.** The borrower posts assets from their wallet into a vault and borrows against them. Works like a standard money market.
- **LP position as collateral.** The LP position itself is locked as collateral and keeps earning swap fees and lending yield while it backs the loan. Liquidation unwinds the position into its underlying tokens.
- **LP plus separate collateral.** An existing LP also posts separate assets to borrow, leaving the LP position untouched.

### Progressive Liquidation

| Level | Trigger health factor | Maximum closed | Recovery window |
|---|---|---|---|
| Warning | below 1.05 | 5 percent | 20 minutes |
| Caution | below 1.00 | 15 percent | 5 minutes |
| Danger | below 0.90 | 30 percent | 2 minutes |
| Critical | below 0.80 | 60 percent | 30 seconds |
| Emergency | below 0.70 | 100 percent | immediate |

Each level closes only the minimum required to bring the position back to a safe health factor, and the bonus paid to liquidators scales up as the position worsens. Borrowers who can add collateral or repay during the recovery window keep their positions. Health factor is priced from a time-weighted average, and liquidations pause automatically if the oracle price and the DLMM price diverge beyond a set threshold, which removes the standard flash-loan oracle attack.

### Bad Debt Absorption

If a position ever goes underwater faster than it can be closed, losses are absorbed in order: first a protocol backstop reserve funded by a share of liquidation bonuses and interest, then a governance emergency fund, and only as a last resort a proportional reduction spread across all lenders of that token. Because lending is aggregated per token, that last-resort cost is spread across the widest possible base rather than concentrated on one pool.

---

## Planned Module Structure

The Soroban implementation mirrors the module layout already built in Move, adapted to Soroban's contract model. It groups into the DEX, the lending market, the router that connects them, and supporting services.

**DEX**
- `dex` — pool creation, reserves, and pool views
- `dex_liquidity` — add and remove liquidity flows
- `bin` — per-bin reserve and fee accounting
- `lp_position` — LP ownership, ranges, shares, and lock state
- `price_math` — fixed-point price and bin math
- `fee` — dynamic swap fee that scales with volatility
- `distribution` — liquidity shape weighting across a range
- `intent_swap` — intent-based swaps with solver execution, single-hop and multi-hop

**Lending**
- `lending` — orchestration of borrow, repay, and liquidate
- `global_supply` — the per-token global supply pool, utilization, and rate
- `borrow_allocation` — farthest-bins-first debit and credit on borrow and repay
- `supply` — per-pool LP contribution accounting into the global pool
- `borrow` — borrow positions and debt accounting
- `liquidation` — the five-level progressive liquidation engine
- `vault` — external collateral vaults for separate-collateral borrowers
- `interest_rate` — utilization, proximity, and volatility rate components
- `activation` — lending gate for newly created pools

**Router and services**
- `ibr` — the inactive bin router: tier assignment, deployment, and recall on bin change
- `oracle` — time-weighted price accumulator, fallback prices, and divergence checks
- `admin` — roles, safety pauses, treasury, timelock, and keeper permissions

A keeper and indexer layer runs off-chain to monitor health factors, service the withdrawal queue, update price feeds, and trigger backstop liquidation. The on-chain contracts hold all state and enforce all invariants. The keepers only call public entry points.

---

## Milestones to Mainnet

The full milestone breakdown, deliverables, and budget mapping are in the documentation source. In summary:

1. **DLMM and liquidity layer on testnet.** The bin-based DEX, the inactive bin router, and the deployment tier accounting deployed and functional on Stellar testnet.
2. **Lending market and global supply.** The global per-token supply pool, borrow allocation, interest rate model, the three borrower types, and the oracle adapter with a slow-feed tolerant configuration, with full test coverage.
3. **Liquidation and keepers.** The five-level progressive liquidation engine, recall handling, and the keeper and indexer layer for health factor monitoring and price updates.
4. **Mainnet.** Mainnet deployment of the Soroban contracts, with a real-world asset collateral type configured and the protocol live and ready for liquidity.

---

## Test Plan

- Unit tests for each contract module, covering bin math, deployment tier transitions, global supply accounting, interest accrual, and health factor calculations
- Integration tests for the swap and lending interaction, confirming that a borrow draws from the correct bins, that repayments credit the exact bins debited, and that the accounting invariants between bins, pools, and the global supply pool always hold
- Recall tests covering both the instant case and the pending case, confirming that no open loan is force-closed and that pending recall clears as borrowers repay
- Scenario tests for each liquidation level, including recovery window behavior, partial closes, and the oracle divergence pause
- Oracle lag simulations to confirm RWA collateral is not liquidated on routine feed delays

---

## Prior Work

The team has built Juncta's core in Move on the Cedra testnet. The DLMM is live and functional there today. The integrated lending market is built and currently under internal audit. The team has also built on Stellar before, delivering a DeFi indexing pipeline for major Stellar protocols through CelerFi. The Soroban work in this repository is a port and adaptation of a design that already runs as working code, not a research effort starting from zero.

---

## Use of AI

Documentation and planning artifacts in this project were drafted with AI assistance and reviewed by the team. Protocol design, architecture decisions, and contract implementation are the team's own work.

---

## Links

- Technical Architecture Document: https://juncta.gitbook.io/juncta-docs
- Juncta on GitHub: https://github.com/JunctaXYZ
- Website: https://juncta.xyz/
- X: https://x.com/JunctaXYZ
