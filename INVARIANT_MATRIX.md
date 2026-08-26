# Cross-Contract Invariant Matrix

This document describes the invariant relationships between constants across
the SPulse contract suite. Each constant is individually safe, but their
combinations create emergent boundary conditions that must be verified.

## Constant Reference

### prediction_market
| Constant | Value | Description |
|----------|-------|-------------|
| `MIN_BET` | 1 XLM | Minimum net stake |
| `TOTAL_FEE_BPS` | 200 (2%) | Total fee on bet |
| `PLATFORM_FEE_BPS` | 150 (1.5%) | Platform portion |
| `BPS_DENOM` | 10,000 | Basis points denominator |
| `MAX_WITHDRAWAL_BPS` | 2,000 (20%) | Per-withdrawal cap |
| `WITHDRAW_DELAY_SECS` | 86,400 (24h) | Timelock |
| `DISPUTE_WINDOW_SECS` | 604,800 (7 days) | Challenge window |
| `TTL_BUMP` | ~1 year | Storage TTL threshold |
| `TTL_HIGH` | ~2 years | Storage TTL extend |
| `WIN_TOKENS` | 1 PULSE | Winner reward |
| `LOSE_TOKENS` | 0.2 PULSE | Loser reward |
| `WIN_POINTS` | 30 | Winner points |
| `LOSE_POINTS` | 10 | Loser points |

### leaderboard
| Constant | Value | Description |
|----------|-------|-------------|
| `MAX_TOP_PLAYERS` | 50 | Top list size |
| `DECAY_PERIOD_LEDGERS` | ~7 days | Decay period |
| `DECAY_RETAIN_NUM/DEN` | 9/10 | 90% per period |

### referral_registry
| Constant | Value | Description |
|----------|-------|-------------|
| `MAX_REFERRAL_DEPTH` | 5 | Max chain depth |
| `WELCOME_BONUS_PTS` | 5 | Registration bonus |
| `REFERRAL_BONUS_PTS` | 3 | Per-bet referrer bonus |

## Invariant Matrix

| # | Constant A | Constant B | Invariant | Test |
|---|------------|------------|-----------|------|
| 1 | AccumulatedFees | Σ(MarketFees) + LegacyFees | Cached sum never drifts | `test_inv_accumulator_equals_ledger_sum` |
| 2 | cancel_market | MarketFees | Cancel reclaims exact market fees | `test_inv_cancel_reclaims_exact_market_fees` |
| 3 | MAX_WITHDRAWAL_BPS (20%) | AccumulatedFees | Single withdraw cannot drain all | `test_inv_withdraw_cap_after_cancel` |
| 4 | WIN_TOKENS + LOSE_TOKENS | total_supply | PULSE supply tracks rewards exactly | `test_inv_pulse_supply_tracks_rewards` |
| 5 | TTL_BUMP (~1yr) | DISPUTE_WINDOW (7d) | Market TTL outlives duration + dispute | `test_inv_market_ttl_outlives_duration` |
| 6 | cancel_market | withdraw_fees | Fee conservation: total = withdrawn + reclaimed + remaining | `test_inv_fee_conservation` |
| 7 | WIN_POINTS + LOSE_POINTS | leaderboard | Points conserved across win/loss/bonus | `test_inv_leaderboard_points_conservation` |
| 8 | MAX_WITHDRAWAL_BPS (20%) | TOTAL_FEE_BPS (2%) | Cap must exceed fee rate | `test_inv_withdraw_cap_exceeds_fee_rate` |
| 9 | DISPUTE_WINDOW_SECS (7d) | TTL_BUMP (~1yr) | Dispute window fits within TTL | `test_inv_dispute_window_fits_in_ttl` |
| 10 | cancel_market | multi-market | Cancel doesn't leak fees across markets | `test_inv_multi_market_fee_isolation` |
| 11 | TTL_BUMP (~1yr) | DISPUTE_WINDOW (7d) | TTL must outlive dispute window | `test_inv_ttl_outlives_duration_plus_dispute` |
| 12 | MAX_WITHDRAWAL_BPS (20%) | cancel_market reclaim | No over-drain on interleaved ops | `test_inv_cancel_then_withdraw_no_overdrain` |
| 13 | cancel_market | withdraw_fees cap | Multi-cancel safety | `test_inv_multiple_cancels_then_withdraw` |
| 14 | PLATFORM_FEE_BPS (1.5%) | referral_fee (0.5%) | Fee split preserves accumulator | `test_inv_fee_split_conservation` |
| 15 | WIN_TOKENS (1 PULSE) | MAX_TOP_PLAYERS (50) | Minting bounded per claim | `test_inv_reward_minting_bounded` |
| 16 | Referral depth (5) | cancel_market reclaim | Deep chains don't wipe accumulator | `test_inv_referral_depth_does_not_affect_accumulator` |
| 17 | MAX_WITHDRAWAL_BPS | market count | Cap scales correctly | `test_inv_withdraw_cap_scales_with_market_count` |

## Governance Process for Constant Changes

1. **Matrix Review**: Any constant change requires reviewing all invariants
   in the row(s) it participates in.

2. **Lifecycle Testing**: Changes to `TTL_BUMP`, `DISPUTE_WINDOW_SECS`, or
   market duration require testing the full lifecycle:
   create → bet → resolve → dispute → claim.

3. **Fee Testing**: Changes to `TOTAL_FEE_BPS`, `PLATFORM_FEE_BPS`, or
   `MAX_WITHDRAWAL_BPS` require testing cancel + withdraw interleaving.

4. **Reward Testing**: Changes to `WIN_TOKENS`, `LOSE_TOKENS`, or
   `MAX_TOP_PLAYERS` require testing minting cap invariants.

5. **Referral Testing**: Changes to `MAX_REFERRAL_DEPTH` require testing
   deep chain behavior and accumulator isolation.

6. **CI Requirement**: All invariant tests must pass before a constant
   change is merged.
