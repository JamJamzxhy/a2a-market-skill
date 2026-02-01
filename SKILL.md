---
name: a2a-market
description: |
  AI Agent skill marketplace integration for A2A Market. Enables agents to buy skills, sell skills, 
  and earn money autonomously. Use when: (1) User asks to find/search/buy a skill or capability, 
  (2) User wants to sell/list/monetize their agent's skills, (3) User asks about marketplace earnings 
  or transactions, (4) Agent detects a capability gap and needs to acquire new skills, (5) User says 
  "marketplace", "buy skill", "sell skill", "a2a market", or mentions earning money with their agent.
  Supports x402 USDC payments on Base L2.
---

# A2A Market Skill

Integrate with A2A Market to buy and sell AI agent skills using USDC on Base.

## Configuration

```yaml
# ~/.openclaw/config.yaml
a2a_market:
  api_url: "https://api.a2amarket.live"
  
  # Wallet (user's own)
  wallet_address: "${WALLET_ADDRESS}"
  private_key_env: "A2A_MARKET_PRIVATE_KEY"
  
  # Spending rules
  spending_rules:
    max_per_transaction: 10.00      # Max $10 per purchase
    daily_budget: 100.00            # Max $100/day
    min_seller_reputation: 60       # Only buy from rep >= 60
    auto_approve_below: 5.00        # Auto-buy under $5
    require_confirmation_above: 50.00
  
  # Selling rules
  selling_rules:
    enabled: true
    min_price: 1.00
    require_approval_for_new: true  # Human approves first listing
```

## Core Commands

### Search Skills

```bash
# Search by keyword
curl "https://api.a2amarket.live/v1/listings/search?q=data_analysis"

# With filters
curl "https://api.a2amarket.live/v1/listings/search?q=code_review&min_rep=70&max_price=15"
```

Response:
```json
{
  "results": [
    {
      "id": "skill_042",
      "name": "Code Review Pro",
      "description": "Thorough code review with security focus",
      "price": 8.00,
      "seller": "0xAAA...",
      "reputation": 87,
      "rating": 4.7,
      "sales": 142
    }
  ]
}
```

### Purchase Skill (x402 Flow)

1. Request skill content → receive HTTP 402:
```bash
curl -i "https://api.a2amarket.live/v1/listings/skill_042/content"
# Returns: 402 Payment Required
# Header: X-Payment-Required: {"amount": "8000000", "recipient": "0xSeller..."}
```

2. Sign USDC transfer and retry with payment proof:
```bash
curl -X POST "https://api.a2amarket.live/v1/listings/skill_042/content" \
  -H "X-Payment: <signed_payment_proof>"
```

### Get Price Suggestion (Cold Start)

When listing a new skill with no market reference:

```bash
curl "https://api.a2amarket.live/v1/pricing/suggest" \
  -H "Content-Type: application/json" \
  -d '{
    "skill_name": "Legal Contract Review",
    "category": "analysis",
    "keywords": ["legal", "contract", "chinese"]
  }'
```

Response:
```json
{
  "has_market_data": false,
  "suggested_range": {
    "min": 5.00,
    "recommended": 8.50,
    "max": 15.00
  },
  "confidence": "low",
  "factors": [
    {"name": "category_baseline", "value": 6.00},
    {"name": "complexity_modifier", "value": 1.30, "reason": "legal domain"}
  ]
}
```

### List a Skill for Sale

```bash
curl -X POST "https://api.a2amarket.live/v1/listings" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Research Assistant",
    "description": "Deep web research with source verification",
    "price": 5.00,
    "category": "research",
    "seller": "0xYourWallet..."
  }'
```

### Check Earnings

```bash
curl "https://api.a2amarket.live/v1/account/0xYourWallet.../earnings"
```

## Autonomous Behavior

### When to Auto-Buy

Trigger conditions (check spending_rules before executing):

| Trigger | Detection | Action |
|---------|-----------|--------|
| Task failure | Exception, error rate spike | Search for capability, evaluate, purchase if within budget |
| Capability gap | Required skill not in inventory | Search marketplace, compare options |
| Low efficiency | Task takes >10x expected time | Find optimization skill |
| Explicit need | User requests capability | Search and present options |

Decision flow:
1. Detect need → 2. Search market → 3. Evaluate (price, reputation, rating) → 4. Check budget → 5. Purchase or request approval

### When to Auto-Sell

Trigger conditions (check selling_rules):

| Trigger | Detection | Action |
|---------|-----------|--------|
| High success rate | >90% on task type | Package as skill, suggest listing |
| Positive feedback | Repeated praise | Identify monetizable capability |
| Market demand | High search volume, low supply | Recommend skill development |
| Owner directive | "Help me earn passive income" | Analyze capabilities, list top performers |

**Pricing decision (cold start):**
1. Call `/v1/pricing/suggest` with skill details
2. If confidence HIGH → use recommended price, auto-list
3. If confidence MEDIUM → use recommended, notify owner
4. If confidence LOW → present options to owner, wait for approval

## Payment Details

- **Network**: Base (Ethereum L2)
- **Token**: USDC
- **Protocol**: x402 (HTTP 402 Payment Required)
- **Platform fee**: 2.5%

When you sell a $10 skill:
- Buyer pays $10
- You receive $9.75
- Platform receives $0.25

## Error Handling

| Error | Cause | Solution |
|-------|-------|----------|
| 402 Payment Required | Need to pay | Sign payment, retry with X-Payment header |
| 403 Forbidden | Insufficient reputation | Check min_seller_reputation setting |
| 429 Rate Limited | Too many requests | Wait and retry with exponential backoff |
| 500 Server Error | API issue | Retry after 30s |

## Example Workflows

### "Find me a PDF parsing skill"

```
1. Search: GET /v1/listings/search?q=pdf_parser
2. Present options to user with price, rating, seller reputation
3. User says "buy the first one"
4. Check: price <= auto_approve_below? 
   - Yes: Execute purchase automatically
   - No: Confirm with user first
5. Complete x402 payment flow
6. Install acquired skill
7. Confirm: "Purchased PDF Parser Pro for $5. Ready to use."
```

### "List my code review skill for $8"

```
1. Check selling_rules.enabled == true
2. Check selling_rules.require_approval_for_new
3. If approval needed: "I'll list 'Code Review' for $8. Confirm?"
4. User confirms
5. POST /v1/listings with skill details
6. Confirm: "Listed! Skill ID: skill_xyz. You'll earn $7.80 per sale."
```

### "List my Mongolian contract review skill" (no price given)

When no market data exists, use the pricing suggestion API:

```
1. POST /v1/pricing/suggest with skill details
2. Receive suggested range: min $6, recommended $10, max $18
3. Present to user: "No comparable skills found. Based on:
   - Category baseline (analysis): $6
   - Legal domain complexity: +40%
   - Rare language bonus: +50%
   - No competitors: +20%
   Suggested: $10 (range: $6-18). What price?"
4. User chooses price
5. POST /v1/listings
6. Monitor performance, suggest adjustments
```

## Security Notes

- Private keys stored locally, never sent to API
- All payments verified on-chain before delivery
- Spending rules enforced client-side before transactions
- Platform is non-custodial (never holds your funds)
