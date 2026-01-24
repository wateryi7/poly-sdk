# CLAUDE.md - AI Assistant Guide for poly-sdk

## Project Overview

`@catalyst-team/poly-sdk` is a TypeScript SDK for Polymarket - a prediction markets trading platform. It provides unified access to market data, trading operations, smart money analysis, arbitrage detection, and on-chain operations.

**Version:** 0.4.3
**Package:** `@catalyst-team/poly-sdk` (npm)
**License:** MIT
**Module System:** ESM only (no CommonJS)

## Quick Reference

```bash
# Build
pnpm build              # Compile TypeScript to dist/

# Development
pnpm dev                # Watch mode compilation

# Testing
pnpm test               # Run unit tests
pnpm test:watch         # Watch mode
pnpm test:integration   # Integration tests (requires network)

# Run examples
pnpm example:basic      # Basic SDK usage
pnpm example:trading    # Order placement
pnpm example:arb-service # Arbitrage detection

# Run scripts
tsx scripts/<script-name>.ts
```

## Architecture

The SDK follows a three-layer architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                      PolymarketSDK                          │
│                     (Entry Point)                           │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: High-Level Services (Recommended for most use)     │
│   TradingService, MarketService, WalletService,             │
│   RealtimeServiceV2, SmartMoneyService, ArbitrageService,   │
│   DipArbService, OnchainService                             │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Low-Level API Clients                              │
│   DataApiClient, GammaApiClient, SubgraphClient,            │
│   CTFClient, BridgeClient                                   │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: Core Infrastructure                                │
│   RateLimiter, Cache, ErrorHandling, Types                  │
└─────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
poly-sdk/
├── src/                      # Main source code
│   ├── index.ts              # SDK entry point and exports
│   ├── core/                 # Core infrastructure
│   │   ├── types.ts          # Unified type definitions
│   │   ├── rate-limiter.ts   # Per-API rate limiting (Bottleneck)
│   │   ├── cache.ts          # TTL-based in-memory cache
│   │   ├── unified-cache.ts  # Cache adapter bridge
│   │   └── errors.ts         # Error codes and handling
│   ├── clients/              # Low-level API clients
│   │   ├── data-api.ts       # Positions, trades, leaderboard
│   │   ├── gamma-api.ts      # Markets, events, search
│   │   ├── subgraph.ts       # On-chain data (Goldsky)
│   │   ├── ctf-client.ts     # CTF contract operations
│   │   └── bridge-client.ts  # Cross-chain deposits
│   ├── services/             # High-level services
│   │   ├── trading-service.ts     # Order management
│   │   ├── market-service.ts      # Market data & orderbooks
│   │   ├── wallet-service.ts      # Wallet analysis
│   │   ├── realtime-service-v2.ts # WebSocket streaming
│   │   ├── smart-money-service.ts # Smart money tracking
│   │   ├── arbitrage-service.ts   # Arbitrage detection
│   │   ├── dip-arb-service.ts     # Crypto dip arbitrage
│   │   ├── onchain-service.ts     # Unified on-chain ops
│   │   ├── swap-service.ts        # DEX swaps (Polygon)
│   │   └── authorization-service.ts # ERC20/ERC1155 approvals
│   ├── utils/                # Utility functions
│   └── __tests__/            # Test files
│       └── integration/      # Integration tests
├── examples/                 # 13 runnable examples
├── scripts/                  # Utility scripts
│   ├── dip-arb/              # Dip arbitrage scripts
│   ├── smart-money/          # Smart money analysis
│   ├── trading/              # Trading utilities
│   ├── wallet/               # Wallet operations
│   └── deposit/              # Deposit utilities
├── docs/                     # Documentation
│   ├── api/                  # API reference
│   ├── architecture/         # SDK design
│   ├── guides/               # Practical guides
│   └── concepts/             # Conceptual docs
├── vitest.config.ts          # Unit test config (30s timeout)
└── vitest.integration.config.ts # Integration test config (60s timeout)
```

## Key Services

### PolymarketSDK (Entry Point)
Main SDK class providing access to all services:
```typescript
import { PolymarketSDK } from '@catalyst-team/poly-sdk';

const sdk = await PolymarketSDK.create({
  privateKey: process.env.PRIVATE_KEY,  // Optional for trading
});
await sdk.start();
```

### Service Overview

| Service | Purpose | Key Methods |
|---------|---------|-------------|
| `TradingService` | Order management | `placeLimitOrder()`, `cancelOrder()`, `getOrders()` |
| `MarketService` | Market data | `getMarket()`, `getOrderbook()`, `getKlines()` |
| `WalletService` | Wallet analysis | `getWalletProfile()`, `getSmartScore()`, `getPnL()` |
| `RealtimeServiceV2` | WebSocket streaming | `subscribeMarket()`, `subscribePrices()` |
| `SmartMoneyService` | Smart money tracking | `trackWallet()`, `startAutoCopyTrading()` |
| `ArbitrageService` | Arbitrage detection | `scan()`, `executeOpportunity()` |
| `OnchainService` | On-chain operations | `split()`, `merge()`, `redeem()`, `approve()` |

## Core Types

```typescript
// Order sides
type Side = 'BUY' | 'SELL';

// Order types
type OrderType = 'GTC' | 'GTD' | 'FOK' | 'FAK';

// Orderbook
interface OrderbookLevel { price: number; size: number; }
interface ProcessedOrderbook {
  bids: OrderbookLevel[];
  asks: OrderbookLevel[];
  effectiveBuyPrice: number;
  effectiveSellPrice: number;
}

// Market with YES/NO tokens
interface UnifiedMarket {
  tokenId: string;
  conditionId: string;
  question: string;
  yesToken: { tokenId: string; price: number; };
  noToken: { tokenId: string; price: number; };
}
```

## Coding Conventions

### File Naming
- Services: `*-service.ts` (e.g., `trading-service.ts`)
- Clients: `*-client.ts` (e.g., `data-api.ts`, `ctf-client.ts`)
- Types: Defined in same file or `types.ts`
- Tests: `*.test.ts` (unit), `*.integration.test.ts` (integration)

### Import Style
```typescript
// Use .js extension for local imports (ESM requirement)
import { RateLimiter } from './core/rate-limiter.js';
import { DataApiClient } from './clients/data-api.js';
```

### Type Exports
```typescript
// Re-export types explicitly
export type { Position, Trade } from './clients/data-api.js';
```

### Error Handling
```typescript
import { PolymarketError, ErrorCode, withRetry } from './core/errors.js';

// Use withRetry for API calls
const result = await withRetry(() => api.fetchData(), { maxRetries: 3 });
```

## Rate Limiting Configuration

Built-in rate limits per API:

| API | Limit |
|-----|-------|
| Data API | 100ms min interval |
| Gamma API | 10 req/s |
| CLOB API | 10 req/s |
| Subgraph | 50ms min interval |
| Binance | 10 req/s |

## Cache TTL Values

| Data Type | TTL |
|-----------|-----|
| Market info | 1 minute |
| Wallet positions | 5 minutes |
| Leaderboard | 1 hour |
| Binance K-line | 1 minute |

## Testing

### Unit Tests
```bash
pnpm test               # Run all unit tests
pnpm test:watch         # Watch mode
```

Unit tests are located in `src/**/*.test.ts` (excluding integration/).

### Integration Tests
```bash
pnpm test:integration   # Run integration tests
```

Integration tests are in `src/__tests__/integration/` and have a 60s timeout for network calls.

### Test Configuration
- Framework: Vitest 2.1.8
- Environment: Node
- Coverage: V8 provider

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `@polymarket/clob-client` | Official trading client |
| `@polymarket/real-time-data-client` | WebSocket client |
| `ethers` (v5) | Blockchain operations |
| `bottleneck` | Rate limiting |
| `@catalyst-team/cache` | Cache adapter |
| `ws` | WebSocket support |

## Important Notes

### Polymarket-Specific Concepts
- **YES/NO Tokens**: Markets have complementary YES and NO tokens (prices sum to ~$1)
- **USDC.e**: Bridged USDC is required for CTF operations (not native USDC)
- **CTF**: Conditional Token Framework for split/merge/redeem operations
- **CLOB**: Central Limit Order Book for trading

### Orderbook Mirroring
YES and NO orderbooks are mirrors of each other:
- YES bid at 0.60 = NO ask at 0.40
- YES ask at 0.65 = NO bid at 0.35

### Environment Variables
```bash
PRIVATE_KEY=           # Wallet private key (for trading)
POLYGON_RPC_URL=       # Optional: Custom Polygon RPC
```

## Common Patterns

### SDK Initialization
```typescript
// Read-only (no trading)
const sdk = await PolymarketSDK.create({});

// With trading capabilities
const sdk = await PolymarketSDK.create({
  privateKey: process.env.PRIVATE_KEY,
});
await sdk.initialize();  // Initialize CLOB client for trading
```

### Market Data
```typescript
// Get market info
const market = await sdk.markets.getMarket(conditionId);

// Get orderbook
const orderbook = await sdk.tradingService.getProcessedOrderbook(tokenId);

// Subscribe to real-time updates
sdk.realtime.subscribeMarket(tokenId, {
  onPriceChange: (data) => console.log('Price:', data),
});
```

### Trading
```typescript
// Place limit order
const order = await sdk.tradingService.placeLimitOrder({
  tokenId,
  side: 'BUY',
  price: 0.50,
  size: 100,
});
```

## Breaking Changes Log

### v0.3.0
- Removed legacy `RealtimeService` - use `RealtimeServiceV2`
- Removed `ClobApiClient` - use `TradingService` instead
- Changed to ESM-only module system

## Files to Know

| File | Description |
|------|-------------|
| `src/index.ts` | Main SDK exports and PolymarketSDK class |
| `src/core/types.ts` | All shared type definitions |
| `src/services/trading-service.ts` | Order placement and management |
| `src/services/market-service.ts` | Market data aggregation |
| `src/services/arbitrage-service.ts` | Arbitrage detection logic |
| `package.json` | Scripts and dependencies |
| `vitest.config.ts` | Test configuration |
