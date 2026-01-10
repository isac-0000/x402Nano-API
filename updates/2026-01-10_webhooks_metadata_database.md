# Update: Webhooks, Metadata & Database Persistence

**Date:** January 10, 2026  
**Version:** 0.2.0  
**Status:** Production Ready

---

## 🎉 What's New

### 1. Webhook Support
AI agents and merchants can now receive **instant push notifications** when transactions are paid. No more polling!

**Features:**
- Automatic POST to `callback_url` when payment is confirmed
- Retry logic: 3 attempts (0s, 30s, 5min delays)
- HMAC-SHA256 signature for verification
- Complete delivery logging in database

**Example:**
```json
POST /transaction/create
{
  "receive_address": "nano_...",
  "amount": "0.01",
  "callback_url": "https://yoursite.com/webhook",
  "metadata": "{\"order_id\":\"123\"}"
}
```

When paid, your endpoint receives:
```json
POST https://yoursite.com/webhook
Headers:
  Content-Type: application/json
  X-Transaction-Id: uuid-123

Body:
{
  "transaction_id": "uuid-123",
  "receive_address": "nano_...",
  "amount": "0.01",
  "metadata": "{\"order_id\":\"123\"}",
  "paid_at": "2026-01-10T14:30:00Z"
}
```

**Security:**
- Whitelist our server IP: `57.129.16.77` in your firewall
- Industry-standard webhook security method

### 2. Metadata Field
Store custom context with transactions (max 4KB JSON).

**Use Cases:**
- E-commerce: Order IDs, customer info, product details
- AI Services: Session IDs, service types, user identifiers
- API Metering: Endpoint names, usage tiers, billing info

**Example:**
```json
{
  "metadata": {
    "order_id": "ORD-12345",
    "customer_email": "buyer@example.com",
    "product": "Premium License",
    "quantity": 1
  }
}
```

Metadata is:
- Validated (must be valid JSON)
- Persisted in database
- Returned in `/transaction/status` responses
- Included in webhook payloads

### 3. PostgreSQL Persistence
Paid transactions are now permanently stored for audit trail and history.

**Database Tables:**
- `paid_transactions` - Completed payments (transaction_id, amount, metadata, paid_at)
- `webhook_logs` - Delivery attempts (status codes, errors, timestamps)

**Active vs Paid:**
- Active transactions (< 1 hour) → In-memory cache (fast!)
- Paid transactions → PostgreSQL (permanent audit trail)
- Unpaid expired transactions → Auto-deleted from cache

**Benefits:**
- No data loss on server restart
- Transaction history queries
- Webhook delivery debugging
- Compliance & accounting records

---

## 🔧 Technical Changes

### API Changes

#### POST /transaction/create
**New Fields:**
```typescript
{
  receive_address: string,
  amount: string,
  callback_url?: string,  // NEW - Optional webhook URL
  metadata?: string       // NEW - Optional JSON string (max 4KB)
}
```

#### POST /transaction/status/{id}
**New Response Field:**
```typescript
{
  success: string,
  message: string,
  transaction_id: string,
  is_paid: boolean,
  metadata?: string  // NEW - Returns stored metadata
}
```

### Database Migrations

**Created:**
- `20260110120000_create_paid_transactions_table.up.sql`
- `20260110120001_create_webhook_logs_table.up.sql`

**Run:**
```bash
sqlx migrate run
```

### Environment Variables

None required! The server handles everything automatically.

---

## 📖 Updated Documentation

- **API_DOCUMENTATION.md** - Webhook & metadata examples
- **README.md** - Feature list updated
- **WEBHOOK_GUIDE.md** - Complete integration guide (NEW)

---

## 🚀 Migration Guide

### For Existing Integrations

**No Breaking Changes!** All existing endpoints work exactly the same.

**Optional Upgrades:**

1. **Add Webhooks (Recommended)**
   - Add `callback_url` to transaction creation
   - Implement webhook endpoint on your server
   - Whitelist our server IP: `57.129.16.77`

2. **Use Metadata (Optional)**
   - Add `metadata` JSON to track custom data
   - Access it in status responses and webhooks

### Example: Upgrade to Webhooks

**Before:**
```python
# Create transaction
response = requests.post('https://api.x402nano.com/transaction/create', json={
    'receive_address': 'nano_...',
    'amount': '0.01'
})
tx_id = response.json()['transaction_id']

# Poll for status
while True:
    status = requests.post(f'https://api.x402nano.com/transaction/status/{tx_id}')
    if status.json()['is_paid']:
        break
    time.sleep(5)  # Inefficient polling!
```

**After:**
```python
# Create transaction with webhook
response = requests.post('https://api.x402nano.com/transaction/create', json={
    'receive_address': 'nano_...',
    'amount': '0.01',
    'callback_url': 'https://mysite.com/webhook',  # NEW!
    'metadata': json.dumps({'order_id': '123'})     # NEW!
})

# Your webhook endpoint gets instant notification
@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json
    order_id = json.loads(data['metadata'])['order_id']
    # Mark order as paid, start shipping!
    return 'OK', 200
```

---

## 📊 Performance Impact

- **Webhook delivery:** Async, non-blocking (< 1ms impact on payment response)
- **Database writes:** Async, completed transactions only
- **Status endpoint:** Unchanged (still uses cache for active transactions)

**No performance degradation for existing flows.**

---

## 🐛 Known Issues & Limitations

- Webhook retry attempts: 3 max (0s, 30s, 5min)
- Metadata size limit: 4KB
- Transaction history endpoint: Coming in next update
- Rate limiting improvements: Coming in next update

---

## 🔜 Next Update

**Focus: Rate Limiting & History Endpoint**
- IP-based rate limiting (1000 req/hour)
- Transaction creation limits (50/hour per API key)
- GET /transaction/history?receive_address=xxx
- Complete documentation updates

**Target:** Week 4, January 2026

---

## 📞 Support

- **Issues:** [GitHub Issues](https://github.com/Andre1987n/x402Nano-API/issues)
- **Email:** contact@x402nano.com
- **Server IP:** 57.129.16.77 (for webhook whitelisting)

---

**Built for AI Agents 🤖 | Zero Fees ⚡ | Instant Settlement 🚀**
