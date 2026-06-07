# 10 — External Service Integrations

## Payment Gateways

### Paystack

| Item | Details |
|:-----|:--------|
| **Website** | https://paystack.com |
| **Supported methods** | Card, Bank Transfer, USSD, QR Code, Mobile Money |
| **Currencies** | NGN (primary), GHS, ZAR, USD |
| **Settlement** | T+1 for verified businesses |
| **Fees** | 1.5% + ₦100 (capped at ₦2,000 for local transactions) |
| **Webhook events** | `charge.success`, `transfer.success`, `transfer.failed`, `refund.processed` |

**Integration flow:**
```
1. Merchant's customer clicks "Pay"
2. Backend calls POST https://api.paystack.co/transaction/initialize
   Body: { amount, email, reference, callback_url }
3. Paystack returns authorization_url
4. Customer redirected to Paystack checkout
5. Customer pays → Paystack sends webhook to POST /v1/webhooks/paystack
6. Backend verifies signature, confirms payment
7. Backend updates order.payment_status = "paid"
8. Backend credits merchant wallet
9. Customer redirected to callback_url (order confirmation page)
```

### Flutterwave

| Item | Details |
|:-----|:--------|
| **Website** | https://flutterwave.com |
| **Supported methods** | Card, Bank Transfer, USSD, Mobile Money, Barter |
| **Currencies** | NGN + 150 currencies |
| **Settlement** | Same-day for verified businesses |
| **Webhook events** | `charge.completed`, `transfer.completed` |

**Integration note:** Same flow as Paystack, abstracted behind the `PaymentGateway` trait.

---

## Messaging

### WhatsApp Business Cloud API (via Twilio)

| Item | Details |
|:-----|:--------|
| **Provider** | Twilio (BSP) |
| **API** | WhatsApp Business Platform Cloud API |
| **Message types** | Template (outbound), Session (reply within 24h) |
| **Pricing** | Per-conversation pricing (varies by category and country) |
| **Template approval** | Required for outbound messages (24-48h review by Meta) |

**Message categories:**
- **Utility:** Order confirmations, shipping updates, payment receipts → lower cost
- **Marketing:** Promotions, campaigns → higher cost, requires opt-in
- **Authentication:** OTP, verification codes

**Integration pattern:**
```
1. Merchant creates message template in dashboard
2. Template submitted to WhatsApp for approval (via Twilio API)
3. On order status change, background job:
   a. Resolves customer's WhatsApp number
   b. Renders template with variables (order_number, status, etc.)
   c. Sends via Twilio WhatsApp API
   d. Stores message record with status tracking
4. Delivery receipts received via webhook → update message status
```

**Inbound messages:**
```
1. Customer sends WhatsApp message to business number
2. Twilio webhook → POST /v1/webhooks/whatsapp
3. Backend creates inbound message record
4. Merchant sees message in dashboard messaging inbox
5. Merchant replies → backend sends via API (within 24h session window)
```

### SMS (Termii)

| Item | Details |
|:-----|:--------|
| **Website** | https://termii.com |
| **Coverage** | Nigeria + West Africa |
| **Channels** | SMS, WhatsApp (basic), Voice |
| **Pricing** | ~₦4 per SMS (Nigeria) |
| **Use cases** | OTP, order notifications, campaigns |

**Integration:**
```rust
pub async fn send_sms(
    api_key: &str,
    to: &str,        // "+2348012345678"
    message: &str,
    sender_id: &str, // "Punta" (max 11 chars)
) -> Result<SmsResponse, AppError> {
    let client = reqwest::Client::new();
    let res = client
        .post("https://api.ng.termii.com/api/sms/send")
        .json(&json!({
            "to": to,
            "from": sender_id,
            "sms": message,
            "type": "plain",
            "channel": "generic",
            "api_key": api_key,
        }))
        .send()
        .await?;
    // ...
}
```

### Email (Resend)

| Item | Details |
|:-----|:--------|
| **Website** | https://resend.com |
| **Free tier** | 3,000 emails/month |
| **Features** | Templates, analytics, DKIM/SPF/DMARC |
| **Use cases** | Order confirmations, password reset, invoices, campaigns |

**Email types:**
| Email | Trigger |
|:------|:--------|
| Welcome email | Merchant registration |
| Password reset | Forgot password request |
| Order confirmation | New order placed |
| Payment receipt | Payment received |
| Shipping notification | Order shipped |
| Invoice | Billing cycle |
| Low stock alert | Stock below threshold |
| Campaign | Scheduled marketing campaign |

---

## Logistics

### Shipbubble

| Item | Details |
|:-----|:--------|
| **Website** | https://shipbubble.com |
| **Coverage** | Nigeria (nationwide) |
| **Providers** | Aggregates: GIG Logistics, FedEx, DHL, GIGL, Kwik, etc. |
| **Features** | Rate comparison, booking, tracking, webhooks |

**Integration flow:**
```
1. Customer enters shipping address during checkout
2. Backend calls Shipbubble rate API → gets quotes from multiple providers
3. Customer selects shipping option
4. On order confirmation, backend books shipment via Shipbubble
5. Shipbubble returns tracking number
6. Delivery status updates received via webhook
7. Customer can track via order tracking page
```

**Key endpoints:**
```
POST /v1/shipping/fetch_rates      → Get shipping rate quotes
POST /v1/shipping/labels           → Book shipment & generate label
GET  /v1/shipping/tracking/:id     → Get tracking info
POST /v1/webhook (Shipbubble)      → Status update webhooks
```

---

## Object Storage (Cloudflare R2)

**Bucket structure:**
```
punta-media/
├── tenants/
│   ├── {tenant_id}/
│   │   ├── products/
│   │   │   ├── {product_id}/
│   │   │   │   ├── original/          # Original uploads
│   │   │   │   └── optimized/         # Resized versions
│   │   │   │       ├── thumb_200.webp
│   │   │   │       ├── medium_600.webp
│   │   │   │       └── large_1200.webp
│   │   ├── store/
│   │   │   ├── logo.png
│   │   │   ├── favicon.ico
│   │   │   └── hero.jpg
│   │   ├── invoices/
│   │   │   └── INV-{order_number}.pdf
│   │   └── expenses/
│   │       └── receipts/
│   │           └── {expense_id}.jpg
│   └── ...
```

**Image optimization:**
- On upload: generate WebP thumbnails at 200px, 600px, 1200px widths
- Use Cloudflare Image Resizing (if available) or process via background job
- Serve via Cloudflare CDN with long cache TTLs

---

## Integration Error Handling

All external service calls follow this pattern:

```rust
/// Retry with exponential backoff for transient failures
pub async fn call_with_retry<F, T>(
    operation: F,
    max_retries: u32,
) -> Result<T, AppError>
where
    F: Fn() -> Pin<Box<dyn Future<Output = Result<T, AppError>> + Send>>,
{
    let mut attempt = 0;
    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if e.is_transient() && attempt < max_retries => {
                attempt += 1;
                let delay = Duration::from_millis(100 * 2u64.pow(attempt));
                tokio::time::sleep(delay).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

**Circuit breaker:** If a provider fails > 5 times in 60 seconds, circuit opens for 30 seconds (skip calling, return cached/fallback).
