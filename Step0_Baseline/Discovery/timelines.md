# Timelines (Critical Scenarios)

This document outlines the critical scenarios in the HSTrader system, including Partial Fills, Cancel/Replace, and Margin Call/Stop Out. Each scenario details step-by-step events with unique IDs, based on the system's order statuses, margin modes, and workflow.

## Partial Fills

Partial fills occur when an order is not fully executed in a single transaction, resulting in multiple executions. Order status transitions from 'placed' to 'partially_filled'.

1. **001**: Order received and validated; status set to 'started'.
2. **002**: Order approved and routed to LP; status set to 'placed'.
3. **003**: Partial match found at LP; first fill executed with FillType 'partial'.
4. **004**: Fill notification sent to client; FilledVolume and FilledPrice updated.
5. **005**: Remaining order volume updated; order remains 'placed'.
6. **006**: Order re-queued for further matching.
7. **007**: Subsequent partial fill executed; additional FilledVolume added.
8. **008**: Final fill notification sent; order status set to 'filled' if fully executed.
9. **009**: If remaining volume zero, order completed; else, remains 'partially_filled'.

## Cancel/Replace

Cancel/Replace allows modifying or canceling an existing order. Order status can change to 'canceled' or 'rejected'.

1. **010**: Cancel/Replace request received from client.
2. **011**: Original order located in account observer.
3. **012**: Check if order is eligible (status 'placed' or 'partially_filled').
4. **013**: If cancel, set order status to 'canceled'; remove from matching engine.
5. **014**: If replace, update parameters (price, volume) in order; re-validate.
6. **015**: Send confirmation to client; log action.
7. **016**: If partial fills exist, handle remaining volume accordingly.
8. **017**: Update account observer and notify LP if necessary.
9. **018**: Log the cancel/replace event in journal.

## Margin Call/Stop Out

Margin Call/Stop Out occurs when account margin level drops below thresholds. Modes include percentage, money, large position, or dynamic. Account liquidation status set accordingly.

1. **019**: Account margin level calculated on each tick (Equity / UsedMargin * 100).
2. **020**: Check group margin call mode (percentage, money, etc.).
3. **021**: If margin call threshold reached (e.g., percentage of UsedMargin), set liquidation to 'marginlevel'.
4. **022**: Send margin call warning to client; prevent new orders.
5. **023**: If stop out level reached, cancel all pending orders; status 'canceled'.
6. **024**: Close positions based on mode: large position closes biggest trade; dynamic reduces volume.
7. **025**: Update account status to 'margin_call_reached' or 'margin_stop_out_reached'.
8. **026**: Log liquidation event; notify risk management.
9. **027**: Account moved to Margin Call accounts group for monitoring.
