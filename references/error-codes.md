# PolyBaskets Error Codes

## BasketMarketError

| Error | Trigger | Recovery |
|-------|---------|----------|
| `Unauthorized` | Caller lacks required role (admin/settler) | Use the correct account with the assigned role |
| `BasketNotFound` | Invalid basket_id | Query `GetBasketCount` to check valid range |
| `BasketNotActive` | Basket is in SettlementPending or Settled status | Cannot bet on non-active baskets |
| `BasketAssetMismatch` | Betting with wrong asset kind (e.g., VARA bet on a Bet basket) | Check basket's `asset_kind` field |
| `NoItems` | Creating basket with empty items array | Add at least 1 item |
| `NotEnoughItems` | Creating basket with fewer than `min_items_per_basket` items | Query `GetConfig` and add enough items |
| `InvalidWeights` | Item weights don't sum to 100% | Ensure `sum(weight_bps) == 10000` (100% in basis points) |
| `DuplicateBasketItem` | Same poly_market_id + selected_outcome appears twice | Remove duplicate items |
| `TooManyItems` | More than 32 items in basket | Reduce to 32 or fewer items |
| `NameTooLong` | Basket name exceeds 128 characters | Shorten the name |
| `DescriptionTooLong` | Description exceeds 512 characters | Shorten the description |
| `MarketIdTooLong` | poly_market_id exceeds 128 characters | Use the numeric Polymarket market ID string |
| `SlugTooLong` | poly_slug exceeds 128 characters | Use valid Polymarket slug |
| `PayloadTooLong` | Settlement payload string too long | Trim payload data |
| `VaraDisabled` | VARA betting is disabled in config | Stop and report to the user. Do NOT fall back to BetToken/BetLane: that legacy lane is retired and its bets are rejected on-chain. |
| `SettlementAlreadyExists` | Settlement already proposed for this basket | Wait for existing settlement to finalize |
| `SettlementNotFound` | No settlement proposed for this basket | Propose settlement first |
| `SettlementNotProposed` | Settlement status is not Proposed | Check settlement status |
| `SettlementNotFinalized` | Trying to claim before settlement is finalized | Wait for finalization |
| `ChallengeDeadlineNotPassed` | Finalizing before the configured challenge window has passed | Wait until `challenge_deadline` timestamp passes |
| `InvalidIndexAtCreation` | index_at_creation_bps is 0 or > 10000 | Use value between 1 and 10000 |
| `InvalidBetAmount` | No VARA value attached to bet transaction | Add `--value <amount>` to vara-wallet call |
| `InvalidFreebetAmount` | Freebet amount is zero or invalid | Use a non-zero raw VARA amount |
| `InvalidResolutionCount` | Resolution count doesn't match basket items count | Provide exactly one resolution per item |
| `DuplicateResolutionIndex` | Same item_index appears twice in resolutions | Each item_index must be unique |
| `ResolutionIndexOutOfBounds` | item_index >= basket items count | Use indices 0 to items.length-1 |
| `ResolutionSlugMismatch` | poly_slug in resolution doesn't match basket item | Use exact slug from basket's items |
| `InvalidResolution` | Malformed resolution data | Check ItemResolution struct format |
| `AlreadyClaimed` | User already claimed payout for this basket | No action needed — already claimed |
| `NothingToClaim` | User has no position in this basket | Verify position exists with GetPositions |
| `TransferFailed` | On-chain VARA transfer failed | Check account balance, retry |
| `FreebetLedgerNotConfigured` | BasketMarket has no ledger configured, or direct caller is not the configured ledger | Stop and report ops/config issue |
| `FreebetLedgerReturnFailed` | Claim could not return freebet principal to ledger | Stop and report; do not blind-retry claims |
| `MathOverflow` | Arithmetic overflow in payout calculation | Bug — report to maintainers |
| `EventEmitFailed` | Failed to emit on-chain event | Retry transaction |
| `InvalidConfig` | Invalid configuration parameters | Check BasketMarketConfig values |

## BetLaneError (legacy reference only; do not route current bets through BetLane)

| Error | Trigger | Recovery |
|-------|---------|----------|
| `AccessDenied` | Caller lacks admin role | Use admin account |
| `Paused` | BetLane contract is paused | Wait for admin to resume |
| `InvalidConfig` | Invalid BetLaneConfig values | Check min_bet, max_bet |
| `InvalidAmount` | Bet amount is zero | Provide non-zero amount |
| `AmountBelowMinBet` | Amount < min_bet from config | Increase bet amount |
| `AmountAboveMaxBet` | Amount > max_bet from config | Decrease bet amount |
| `InvalidIndexAtCreation` | index_at_creation_bps is 0 or > 10000 | Use value between 1 and 10000 |
| `QuoteSignerNotConfigured` | BetLane has no configured quote signer | Admin must configure quote signer |
| `QuoteTargetMismatch` | Signed quote targets another program | Request a fresh quote with this BetLane program ID |
| `QuoteUserMismatch` | Signed quote belongs to another user | Request a fresh quote for the caller address |
| `QuoteBasketMismatch` | Signed quote is for another basket | Request a fresh quote for this basket |
| `QuoteAmountMismatch` | Signed quote amount differs from call amount | Use the exact quoted amount or request a new quote |
| `QuoteExpired` | Quote deadline passed | Request a fresh quote and submit within the TTL |
| `QuoteNonceAlreadyUsed` | Quote was already consumed | Request a fresh quote |
| `InvalidQuoteSignature` | Signature bytes are malformed | Pass the raw quote response from the quote service |
| `InvalidQuoteSigner` | Configured signer is not a valid sr25519 key | Admin must fix quote signer config |
| `QuoteVerificationFailed` | Signature does not verify | Request a fresh quote from the configured quote service |
| `BasketQueryFailed` | Failed to query BasketMarket contract | Check BasketMarket program is active |
| `BasketNotFound` | Invalid basket_id | Verify basket exists |
| `BasketNotActive` | Basket not in Active status | Cannot bet on settled baskets |
| `BasketAssetMismatch` | Basket's asset_kind is not Bet | Use VARA lane for Vara baskets |
| `SettlementQueryFailed` | Failed to query settlement | Check BasketMarket program |
| `SettlementNotFound` | No settlement for this basket | Wait for settlement |
| `SettlementNotFinalized` | Settlement not yet finalized | Wait for finalization |
| `NothingToClaim` | No position in this basket | Verify position exists |
| `AlreadyClaimed` | Already claimed | No action needed |
| `OperationInProgress` | Another bet/claim flow on the same `(user, basket_id)` is still being processed or looks stuck | Wait briefly, query position first, then retry once with a fresh quote only if state did not change; if it keeps repeating, stop and report |
| `BetTokenTransferFromFailed` | BET token transfer failed | Check BET balance and approval |
| `BetTokenPayoutFailed` | Payout transfer failed | Retry or contact admin |
| `BetTokenRefundFailed` | Refund transfer failed | Retry or contact admin |
| `MathOverflow` | Arithmetic overflow | Bug — report |
| `InvalidPageSize` | Invalid pagination params | Use reasonable offset/limit values |
| `EventEmitFailed` | Event emission failed | Retry |
| `RoleManagementFailed` | Role grant/revoke failed | Check admin permissions |

## FreebetLedgerError

| Error | Trigger | Recovery |
|-------|---------|----------|
| `Unauthorized` | Caller is not ledger admin for admin-only methods | Normal agents must not call admin methods |
| `InvalidConfig` | Zero or invalid program/admin id | Stop and report ops/config issue |
| `InvalidAmount` | Grant/spend/return amount is zero | Use non-zero raw VARA amount |
| `InsufficientBalance` | User freebet balance is below requested amount | Lower the amount, or stop and tell the operator to earn credit at app.polybaskets.xyz/rewards |
| `GrantAlreadyApplied` | Grant id was already used | Idempotent grant already applied; no action for agents |
| `GrantIdTooLong` | Grant id exceeds 128 chars | Shorten grant id |
| `GrantReasonTooLong` | Reason exceeds 256 chars | Shorten reason |
| `BetProgramNotAuthorized` | BasketMarket is not authorized in ledger | Stop and report ops/config issue |
| `OperationInProgress` | Same `(user, bet_program_id, basket_id)` spend is pending | Wait, query `GetFreebetPositions` and `BalanceOf`, retry once only if unchanged |
| `DownstreamBetFailed` | BasketMarket rejected the downstream bet | Check basket exists, is Active, `asset_kind=Vara`, VARA enabled, and index is valid |
| `InvalidReturnValue` | Downstream return did not match expected payload | Stop and report contract issue |
| `MathOverflow` | Arithmetic overflow | Bug — report |
| `EventEmitFailed` | Failed to emit on-chain event | Retry only after checking state |
