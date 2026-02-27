# Order Block Detection Algorithm - Technical Documentation

## Overview

This document describes the algorithm used to detect and validate Order Blocks (OBs) in financial markets. Order Blocks are price zones where institutional traders accumulated significant buy or sell orders. The algorithm identifies these zones through a multi-step validation process that ensures only high-probability institutional activity zones are marked.

## Algorithm Flow

The detection process follows a strict seven-step validation sequence. Each step must pass for an Order Block to be drawn on the chart.

### Step 1: Reference Candle Tracking

The algorithm continuously tracks the most recent candles of each direction. For bearish Order Blocks, it tracks the last bullish candle before a potential break. For bullish Order Blocks, it tracks the last bearish candle.

**Key Rule**: The algorithm always uses the most recent candle, even if it has a very small body or wick. This ensures the Order Block is based on the actual last candle before the reversal, not an earlier one.

**Special Handling**: Doji candles (where open equals close) are treated as continuation of the most recent trend direction. If the last tracked candle was bullish, a doji is treated as bullish continuation.

### Step 2: Break Detection

A break occurs when price moves beyond the reference candle's boundaries. For a bearish Order Block, price must break below the last bullish candle's open price or low. For a bullish Order Block, price must break above the last bearish candle's high.

This break signals that price is moving away from the reference zone, potentially creating an Order Block.

### Step 3: Order Block Zone Calculation (Wick Rule)

When a break is detected, the algorithm calculates the Order Block zone using the "Wick Rule." This rule uses only the candle body (the area between open and close prices) rather than the full candle range including wicks.

**Why the Wick Rule?** Wicks often represent stop-loss orders or market noise, while the body represents actual institutional participation. Using only the body creates more accurate Order Block zones.

**Special Case**: If a candle's wick extends beyond the previous candle, the algorithm uses the candle's open price as the boundary instead of the wick extreme. This prevents the Order Block from including areas that were likely just stop-loss hunting.

**Edge Cases Handled**:
- Candles with no top wick (where high equals close) are still valid
- Very small candles (tiny body and wick) are still used if they're the last before the break
- The algorithm ensures the Order Block always starts at the exact last candle before the break

### Step 4: Immediate Cancellation Check

Before any further validation, the algorithm checks if a same-direction candle touches the pending Order Block zone. If a bullish candle touches a pending bullish Order Block, or a bearish candle touches a pending bearish Order Block, the Order Block is immediately cancelled.

**Rationale**: If a same-direction candle appears after the break and touches the zone, it means the original candle wasn't truly the "last" one, so the Order Block is invalid.

### Step 5: Pre-Sweep Mitigation Check

The algorithm checks if price touches the pending Order Block zone and closes in the opposite direction before confirmation. If this happens, the Order Block is considered mitigated (used up) and is marked as such.

**Exception - Wich OB Rule**: If a same-direction wick touches the Order Block first, the mitigation is prevented. This rule recognizes that same-direction wicks touching the zone don't invalidate it, only opposite-direction closes do.

**Visual Result**: Mitigated Order Blocks are displayed with faded colors and green borders, or can be deleted entirely based on user preference.

### Step 6: Confirmation with Marker Validation

For an Order Block to be confirmed, an opposite-direction candle must appear within a specified distance from the Order Block zone. This confirmation candle must also follow a specific pattern that indicates institutional activity.

**Distance Check**: The confirmation candle must be within a maximum distance (configurable in pips) from the Order Block zone. This ensures the Order Block and confirmation are related.

**Marker Pattern Validation**: The algorithm searches backward from the confirmation candle to find one of two marker patterns:

**Straight Marker Pattern**: This pattern shows a clear institutional flow. It requires:
- A bullish candle (institutions accumulating)
- Followed by one or more bearish candles (institutions exiting, creating the Order Block zone)
- Followed by a bullish candle (institutions re-entering, confirming the zone)

The pattern proves that institutions were active in this area, exited, and then returned, validating the Order Block.

**Swing Marker Pattern**: This pattern shows a V-shaped reversal. It requires:
- A high candle
- Followed by a low candle (the deepest point, which becomes the Order Block zone)
- Followed by another high candle

This V-shape indicates a clear reversal point where institutions likely placed orders.

**Marker Search Window**: The algorithm searches up to a configurable number of bars backward (default 20 bars) to find these patterns. This allows markers that formed earlier to still validate the Order Block.

**Timeout Handling**: If marker validation fails, the algorithm waits up to 3 bars for a valid marker to form. If no marker appears within this grace period, the pending Order Block is cleared.

**Gap Zone Handling**: If a price gap exists when confirmation is attempted, the Order Block is stored but not confirmed until price prints inside the gap region. Marker validation then occurs after the gap is filled, allowing markers to form naturally after gap closure.

### Step 7: Sweep Detection and Order Block Drawing

After confirmation, the algorithm waits for a "sweep" to occur. A sweep happens when price breaks beyond the confirmation candle, proving that institutional orders are being executed.

**Sweep Types**:
- **Primary Sweep**: Price breaks above (for bullish) or below (for bearish) the confirmation candle's high or low
- **Secondary Sweep**: Price breaks beyond an additional opposite-direction candle that appeared after confirmation
- **Partial Break**: As an alternative to sweeps, price can break above or below a group of consecutive opposite-direction candles

**Why Sweeps Matter**: The sweep proves that institutional orders at the Order Block zone are being executed. Without a sweep, the Order Block might not be active.

**Drawing the Order Block**: Once a sweep is detected, the Order Block box is drawn on the chart. The box extends from the original Order Block candle to the current bar where the sweep occurred. The vertical boundaries are set by the high and low prices calculated using the Wick Rule.

## Special Algorithm Rules

### Wich OB Rule

This rule prevents premature mitigation of Order Blocks. If a same-direction wick touches the Order Block zone first, the Order Block remains valid even if an opposite-direction candle later touches it. This recognizes that same-direction wicks don't invalidate the zone - only opposite-direction price action does.

### Immediate Cancellation Rule

If a same-direction candle touches a pending Order Block before confirmation, the Order Block is immediately cancelled. This ensures only the true "last" candle before a break is used for Order Block creation.

### Doji Candle Handling

Doji candles (where open equals close, meaning no body) are treated as continuation of the most recent trend. If the last tracked candle was bullish, a doji is treated as bullish continuation. This ensures Order Blocks can still be created even when doji candles appear before breaks.

### Very Small Candle Handling

The algorithm always uses the most recent candle before a break, even if it has an extremely small body or wick. This ensures accuracy - if the last bullish candle before a bearish break is tiny, it's still the correct candle to use for the Order Block.

### Gap Zone Handling

When price gaps occur, Order Blocks in those gap regions are handled specially:
- The Order Block is stored when the break occurs
- Confirmation is deferred until price prints inside the gap
- Marker validation occurs after the gap is filled
- The algorithm continues searching for markers for several bars after gap fill, allowing natural pattern formation

## Algorithm Validation Summary

For an Order Block to be successfully drawn, all of the following must occur in sequence:

1. **Break Detected**: Price breaks beyond reference candle
2. **Zone Calculated**: Order Block zone determined using Wick Rule
3. **Not Cancelled**: No same-direction candle touched the zone
4. **Not Mitigated**: No opposite-direction close touched the zone (or Wich OB Rule protected it)
5. **Confirmation Valid**: Opposite-direction candle appears within distance
6. **Marker Found**: Valid Straight or Swing marker pattern detected
7. **Sweep Occurred**: Price swept the confirmation candle

If any step fails, the Order Block is not drawn. This strict validation ensures only high-probability institutional Order Blocks are marked.

## Algorithm Parameters

The algorithm behavior can be adjusted through several parameters:

- **Marker Lookback**: How far back to search for marker patterns (default: 20 bars)
- **Maximum Distance**: Maximum distance between Order Block and confirmation candle (default: 100 pips)
- **Marker Requirements**: Whether to require Straight marker, Swing marker, or either one
- **Marker Grace Period**: How long to wait for marker formation after confirmation attempt (3 bars)

## Algorithm Output

When all validation steps pass, the algorithm draws a rectangular box on the chart representing the Order Block zone. The box:
- Starts at the original Order Block candle
- Extends to the bar where the sweep occurred
- Shows the high and low boundaries calculated using the Wick Rule
- Can display the marker type that validated it (optional)
- Changes appearance when mitigated (faded colors, green border)

## Algorithm Advantages

This algorithm provides several advantages over simpler Order Block detection methods:

1. **Accuracy**: Uses body-based logic (Wick Rule) for more precise zones
2. **Validation**: Requires marker patterns proving institutional activity
3. **Filtering**: Multiple cancellation and mitigation checks prevent false signals
4. **Flexibility**: Handles edge cases like doji candles, very small candles, and gap zones
5. **Reliability**: Strict multi-step validation ensures only high-probability zones are marked

## Conclusion

This Order Block detection algorithm uses a comprehensive seven-step validation process to identify high-probability institutional order zones. By combining break detection, body-based zone calculation, marker pattern validation, and sweep confirmation, it provides traders with reliable Order Block zones that represent actual institutional activity rather than random price movements.

---

**Version**: 1.2  
**Developer**: FX Coders Hub  
**Date**: February 2026
