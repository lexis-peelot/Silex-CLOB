# Silex-CLOB: Central Limit Order Book DEX

A permissionless, trustless, distributed decentralized exchange (DEX) contract for XELIS blockchain implementing a Central Limit Order Book (CLOB) mechanism with MEV resistance.

## Features

- **MEV Resistant**: All orders are executed at block end in fair price-priority order, preventing front-running and sandwich attacks
- **Permissionless**: Supports any asset pair dynamically - no whitelisting required
- **Trustless**: Fully on-chain execution with no intermediaries
- **Gas Efficient**: Users pay gas, unused gas automatically refunded by the protocol
- **Integrator Fees**: Front-end providers can earn fees on trades routed through their interface
- **Order Expiry**: Optional expiry topoheight for time-limited orders
- **Trade History**: Permanent on-chain trade history with versioned storage
- **Real-Time Ticker**: WebSocket events for real-time trade updates

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
- `integrator: optional<(Address, u16)>` - Optional integrator fee: (recipient address, fee in basis points 1-5000)

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
// Offer 1000 XELIS for 500 USDT with 1% integrator fee
// Deposit: 1000 XELIS
Order(
    asset_wanted: USDT_HASH,
    amount_wanted: 500,
    place_on_book: true,
    expiry: null,
    integrator: (FRONTEND_ADDRESS, 100)  // 100 bps = 1%
)
```

### `Cancel`

Cancel a specific order by ID.

**Parameters:**
- `order_id: u64` - The ID of the order to cancel
- `asset1: Hash` - First asset in the pair
- `asset2: Hash` - Second asset in the pair

**Note:** Only orders already on the book can be cancelled. Pending orders in the current block cannot be cancelled (by design, for MEV resistance).

### `CancelAll`

Cancel all orders where the caller is offering `asset_offered` for `asset_wanted`.

**Parameters:**
- `asset_offered: Hash` - The asset you're offering
- `asset_wanted: Hash` - The asset you want

### `WithdrawIntegratorFees`

Withdraw all accumulated integrator fees for the caller.

**Parameters:** None

**Notes:**
- Withdraws all accumulated fees across all assets
- Only the address that received the fees can withdraw them
- Fees accumulate in the contract until withdrawn

## Integrator Fees

Front-end providers and integrators can earn a percentage of trades routed through their interface.

### How It Works

1. **Pass integrator info with orders**: Include `(your_address, fee_bps)` in the `integrator` parameter
2. **Fee deduction**: When the order trades, the fee is deducted from the *received* amount
3. **Accumulation**: Fees accumulate in the contract, grouped by recipient address
4. **Withdrawal**: Call `WithdrawIntegratorFees()` to claim all accumulated fees

### Fee Calculation

- Fee is specified in basis points: 100 bps = 1%
- Valid range: 1-5000 bps (0.01% to 50%)
- Fee is calculated as: `fee = (amount * fee_bps) / 10000`
- Fee is deducted from the asset the trader *receives*

### Example

If a user buys 1000 USDT with an integrator fee of 100 bps (1%):
- User receives: 990 USDT
- Integrator earns: 10 USDT (accumulated until withdrawn)

## Order Execution Flow

1. **Order Submission**: User calls `Order()` with deposit
   - Order is added to pending queue
   - Batch execution is scheduled for block end (if not already scheduled)

2. **Block End**: `execute_batch()` is called automatically
   - All pending orders are loaded
   - Orders are sorted by price (highest first)
   - Each order is processed:
     - Matched against counter-orders in the book
     - Integrator fees deducted from filled amounts
     - Unmatched remainder either placed on book or refunded
   - Trade history and integrator fees are flushed to permanent storage

3. **Matching Algorithm**:
   - For each pending order, traverse counter-orders in ascending price order
   - Calculate maximum fill based on price compatibility
   - Execute transfers for each match (minus integrator fees)
   - Update or remove maker orders based on remaining amounts
   - Handle expired orders automatically

## Technical Details

### Price Scale

- `PRICE_SCALE = 10^12` (12 decimals)
- Prices are stored as `u128` values scaled by `PRICE_SCALE`
- Price formula: `price = (amount_wanted * PRICE_SCALE) / amount_offered`

### Basis Points Scale

- `BPS_SCALE = 10000` (100% = 10000 bps)
- 1 basis point = 0.01%
- Fee formula: `fee = (amount * fee_bps) / 10000`

### Unfillable Orders

An order is considered "unfillable" if no integer give exists such that:
- `1 <= floor(give * PRICE_SCALE / price) <= rem`

Unfillable remainders are automatically refunded to prevent dust accumulation.

### Trade History

Trade history is stored in versioned storage:
- Each block creates a new version with `previous_topoheight` linking
- Use `get_contract_data` RPC to get latest version
- Follow `previous_topoheight` chain via `get_contract_data_at_topoheight` to walk back through history

### Real-Time Ticker

The contract fires events for each executed trade, which can be subscribed to via WebSocket:
- Event ID `1`: Trade Execution
- Contains: `asset_offered`, `asset_wanted`, `amount_give`, `amount_receive`
- Events are sent only for successfully executed blocks (reorg-safe)

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
- **Address Validation**: XELIS VM validates all addresses are syntactically valid normal addresses
- **Fee Bounds**: Integrator fees are bounded to 1-5000 bps (max 50%)

## Authors

Peelot, Slixe
