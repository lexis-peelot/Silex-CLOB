# Silex-CLOB: Central Limit Order Book DEX

A permissionless, trustless, distributed decentralized exchange (DEX) contract for XELIS blockchain implementing a Central Limit Order Book (CLOB) mechanism with MEV resistance.

## Features

- **MEV Resistant**: All orders are executed at block end in fair price-priority order, preventing front-running and sandwich attacks
- **Permissionless**: Supports any asset pair dynamically - no whitelisting required
- **Trustless**: Fully on-chain execution with no intermediaries
- **Gas Efficient**: Users pay gas, unused gas automatically refunded by the protocol
- **Order Expiry**: Optional expiry topoheight for time-limited orders
- **Trade History**: Permanent on-chain trade history with versioned storage

## How It Works

### Delayed Batch Execution

All orders submitted during a block are collected and executed together at the end of the block. This ensures:

1. **Fair Ordering**: Orders are sorted by price (highest first) within each batch
2. **MEV Resistance**: Block producers cannot see pending orders and manipulate execution
3. **Price Priority**: Better prices execute first, ensuring optimal fills

### Order Matching

Orders are matched against existing counter-orders (orders in the opposite direction) using a greedy algorithm:

- Traverses counter-orders in ascending price order (best prices first)
- Fills as much as possible from each compatible order
- Price compatibility: `pending.price * counter_price <= PRICE_SCALE²`
- Automatically handles partial fills and unfillable remainders

### Order Book Structure

Orders are stored in B-trees keyed by:
- Price (u128, big-endian)
- Order ID (u64, big-endian)

This enables efficient price-ordered traversal for matching.

## Entry Points

### `Order`

Place a new order for delayed batch execution.

**Parameters:**
- `asset_wanted: Hash` - The asset you want to receive
- `amount_wanted: u64` - The amount of `asset_wanted` you want to receive
- `place_on_book: bool` - If true, unmatched remainder is placed on the order book; if false, remainder is refunded
- `expiry: optional<u64>` - Optional topoheight at which order expires (0 = never expires)

**Deposits:**
- Must deposit exactly one asset (the asset you're offering)
- The amount deposited determines `amount_offered`

**Price Calculation:**
```
price = (amount_wanted * PRICE_SCALE + amount_offered - 1) / amount_offered
```

**Gas Requirements:**
- Must include at least `GAS_PER_ORDER` (50000) extra gas in your transaction
- This gas funds the block-end batch execution
- Unused gas is automatically refunded

**Example:**
```rust
// Offer 1000 XELIS for 500 USDT
// Deposit: 1000 XELIS
Order(
    asset_wanted: USDT_HASH,
    amount_wanted: 500,
    place_on_book: true,
    expiry: null  // Never expires
)
```

### `Cancel`

Cancel a specific order by ID.

**Parameters:**
- `order_id: u64` - The ID of the order to cancel
- `asset1: Hash` - First asset in the pair
- `asset2: Hash` - Second asset in the pair

**Note:** Only orders already on the book can be cancelled. Pending orders in the current block cannot be cancelled (by design, for MEV resistance).

**Example:**
```rust
Cancel(
    order_id: 12345,
    asset1: XELIS_HASH,
    asset2: USDT_HASH
)
```

### `CancelAll`

Cancel all orders where the caller is offering `asset_offered` for `asset_wanted`.

**Parameters:**
- `asset_offered: Hash` - The asset you're offering
- `asset_wanted: Hash` - The asset you want

**Example:**
```rust
CancelAll(
    asset_offered: XELIS_HASH,
    asset_wanted: USDT_HASH
)
```

## Order Execution Flow

1. **Order Submission**: User calls `Order()` with deposit
   - Order is added to pending queue
   - Batch execution is scheduled for block end (if not already scheduled)

2. **Block End**: `execute_batch()` is called automatically
   - All pending orders are loaded
   - Orders are sorted by price (highest first)
   - Each order is processed:
     - Matched against counter-orders in the book
     - Unmatched remainder either placed on book or refunded
   - Trade history is flushed to permanent storage

3. **Matching Algorithm**:
   - For each pending order, traverse counter-orders in ascending price order
   - Calculate maximum fill based on price compatibility
   - Execute transfers for each match
   - Update or remove maker orders based on remaining amounts
   - Handle expired orders automatically

## Technical Details

### Price Scale

- `PRICE_SCALE = 10^12` (12 decimals)
- Prices are stored as `u128` values scaled by `PRICE_SCALE`
- Price formula: `price = (amount_wanted * PRICE_SCALE) / amount_offered`

### Unfillable Orders

An order is considered "unfillable" if no integer give exists such that:
- `1 <= floor(give * PRICE_SCALE / price) <= rem`

Unfillable remainders are automatically refunded to prevent dust accumulation.

### Trade History

Trade history is stored in versioned storage:
- Each block creates a new version with `previous_topoheight` linking
- Use `get_contract_data` RPC to get latest version
- Follow `previous_topoheight` chain via `get_contract_data_at_topoheight` to walk back through history

### Gas Model

- Each order contributes `GAS_PER_ORDER` (50000) gas to batch execution
- Users must include this extra gas in their transaction
- Unused gas is automatically refunded by the protocol
- Only one batch execution per block (shared across all orders)

## Order Sorting

Orders are sorted by a precomputed key:
- **Phase 1**: `place_on_book=true` orders (remainder goes to book)
- **Phase 0**: `place_on_book=false` orders (remainder refunded)
- Within each phase: descending price (highest first)

This ensures:
- Orders that add liquidity to the book execute first
- Better prices execute before worse prices

## Security Considerations

- **MEV Resistance**: Delayed execution prevents front-running
- **No Cancellation of Pending Orders**: Once submitted, orders are committed until block end
- **Automatic Expiry Handling**: Expired orders are automatically refunded
- **Price Compatibility Checks**: Only compatible prices can match
- **Unfillable Detection**: Prevents dust accumulation

## Authors

Peelot, Slixe
