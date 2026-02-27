---
name: stripemeter
description: Integrate Stripe usage-based billing with idempotent event ingestion, late-event handling, and pre-invoice reconciliation. Use when implementing usage metering, tracking API calls or seats, pushing usage to Stripe, handling billing drift, or building usage-based pricing.
---

# StripeMeter

StripeMeter is a Stripe-native usage metering system that ensures correct usage totals for usage-based billing. It dedupes retries, handles late events with watermarks, keeps running counters, and pushes only deltas to Stripe.

## Quick Start

```bash
git clone https://github.com/geminimir/stripemeter && cd stripemeter
cp .env.example .env && docker compose up -d && pnpm -r build
pnpm db:migrate && pnpm dev
```

## Core Concepts

### Events (Immutable Ledger)
Every usage event stored with deterministic idempotency key. Events are never deleted or modified.

### Counters (Materialized Aggregations)
Pre-computed aggregations (sum/max/last) by tenant, metric, customer, and period. Updated in near-real-time.

### Watermarks (Late Event Handling)
Each counter maintains a watermark timestamp. Events within lateness window (default 48h) trigger re-aggregation.

### Delta Push (Stripe Synchronization)
Tracks `pushed_total` per subscription item and only sends delta to Stripe.

## API Endpoints

### Ingest Events

```bash
curl -X POST http://localhost:3000/v1/events/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "events": [{
      "tenantId": "your-tenant-id",
      "metric": "api_calls",
      "customerRef": "cus_ABC123",
      "quantity": 100,
      "ts": "2025-01-16T14:30:00Z"
    }]
  }'
```

### Get Cost Projection

```bash
curl -X POST http://localhost:3000/v1/usage/projection \
  -H "Content-Type: application/json" \
  -d '{"tenantId": "your-tenant-id", "customerRef": "cus_ABC123"}'
```

### Health & Metrics

- Readiness: `GET /health/ready`
- Metrics: `GET /metrics`
- Events: `GET /v1/events?tenantId=X&limit=10`

## Node.js SDK

```typescript
import { createClient } from '@stripemeter/sdk-node';

const client = createClient({
  apiUrl: 'http://localhost:3000',
  tenantId: 'your-tenant-id',
  customerId: 'cus_ABC123'
});

// Track usage
await client.track({
  metric: 'api_calls',
  customerRef: 'cus_ABC123',
  quantity: 100,
  meta: { endpoint: '/v1/search' }
});

// Get live usage
const usage = await client.getUsage('cus_ABC123');
const projection = await client.getProjection('cus_ABC123');
```

## Python SDK

```python
from stripemeter import StripeMeterClient

client = StripeMeterClient(
    api_url="http://localhost:3000",
    tenant_id="your-tenant-id",
    customer_id="cus_ABC123"
)

client.track(
    metric="api_calls",
    customer_ref="cus_ABC123",
    quantity=100
)
```

## Stripe Billing Driver

For direct Stripe integration, use the stripe-driver package:

```typescript
import { StripeBillingDriverImpl } from '@stripemeter/stripe-driver';

const driver = new StripeBillingDriverImpl({
  liveKey: process.env.STRIPE_SECRET_KEY,
  testKey: process.env.STRIPE_TEST_SECRET_KEY
});

// Record usage to Stripe
await driver.recordUsage({
  mode: 'live',
  stripeAccount: 'default',
  subscriptionItemId: 'si_xxx',
  quantity: 100,
  periodStart: '2025-01-01',
  idempotencyKey: 'unique-key'
});

// Get usage summary
const summary = await driver.getUsageSummary(
  'si_xxx',
  '2025-01-01',
  'default'
);
```

## Shadow Mode

Test Stripe usage posting without affecting live invoices:

1. Set `STRIPE_TEST_SECRET_KEY` in environment
2. Mark price mapping with `shadow=true`
3. Provide `shadowStripeAccount`, `shadowPriceId`
4. Live invoices remain unaffected

## Pricing Simulator

```typescript
import { InvoiceSimulator } from '@stripemeter/pricing-lib';

const simulator = new InvoiceSimulator();

const result = simulator.simulate({
  customerId: 'test',
  periodStart: '2024-01-01',
  periodEnd: '2024-02-01',
  usageItems: [{
    metric: 'api_calls',
    quantity: 25000,
    priceConfig: {
      model: 'tiered',
      currency: 'USD',
      tiers: [
        { upTo: 10000, unitPrice: 0.01 },
        { upTo: 50000, unitPrice: 0.008 },
        { upTo: null, unitPrice: 0.005 }
      ]
    }
  }]
});
```

## Project Structure

```
stripemeter/
├── packages/
│   ├── core/           # Shared types, schemas
│   ├── database/       # Drizzle ORM + Redis
│   ├── pricing-lib/    # Pricing calculator
│   ├── stripe-driver/  # Direct Stripe API driver
│   ├── sdk-node/       # Node.js SDK
│   └── sdk-python/     # Python SDK
├── apps/
│   ├── api/            # REST API (Fastify)
│   ├── workers/        # Background workers (BullMQ)
│   ├── admin-ui/       # Admin dashboard
│   └── customer-widget/# Embeddable widget
```

## Environment Variables

```bash
STRIPE_SECRET_KEY=sk_live_xxx       # Live Stripe key
STRIPE_TEST_SECRET_KEY=sk_test_xxx  # Test Stripe key (for shadow mode)
DATABASE_URL=postgres://...         # PostgreSQL connection
REDIS_URL=redis://...               # Redis connection
```

## Common Tasks

### Verify Idempotency

```bash
# Send same event twice - counts once
TENANT_ID=$(uuidgen) bash examples/api-calls/send.sh
curl http://localhost:3000/metrics | grep ingest
```

### Run Reconciliation

Check drift between local totals and Stripe:
- Differences beyond 0.5% epsilon trigger investigation
- See [RECONCILIATION.md](RECONCILIATION.md) for runbook

### Replay Late Events

```bash
curl -X POST http://localhost:3000/v1/replay \
  -H "Content-Type: application/json" \
  -d '{"tenantId": "X", "dryRun": true}'
```

## Additional Resources

- [API Documentation](docs/welcome.md)
- [Simulator Guide](docs/simulator-getting-started.md)
- [Reconciliation Runbook](RECONCILIATION.md)
- [Alert Configuration](ops/ALERTS.md)
