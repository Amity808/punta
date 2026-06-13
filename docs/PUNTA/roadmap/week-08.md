# Week 8 — Messaging Engine

> **Goal:** Build the multi-channel messaging system — WhatsApp, SMS, Email — with templates and bulk campaigns.

---

## Monday — Messaging Database & Engine Core

- [ ] Write migrations: `022_create_message_templates.sql`, `023_create_messages.sql`, `024_create_campaigns.sql`
- [ ] Create `punta-messaging` crate
- [ ] Implement `engine.rs` — `MessageDispatcher`
  - `send_message()` — routes to correct channel handler
  - Channel selection logic: WhatsApp → SMS → Email (fallback chain)
  - Queues messages for async delivery (Redis queue)
  - Tracks message status (queued → sent → delivered → read → failed)
- [ ] Implement `templates.rs` — `TemplateService`
  - `create_template()` — with variable extraction (parse `{{var}}` patterns)
  - `render_template()` — substitute variables into template body
  - `list_templates()` — by channel
  - Default system templates:
    - Order confirmation
    - Payment receipt
    - Shipping notification
    - Delivery confirmation
- [ ] Handlers: `GET/POST/PATCH/DELETE /v1/message-templates`

**Deliverable:** Message engine core with template management.

---

## Tuesday — WhatsApp Integration

- [ ] Implement `whatsapp.rs` — WhatsApp Cloud API client (via Twilio)
  - `send_template_message()` — send approved template with variables
  - `send_text_message()` — send within 24h session window
  - `send_media_message()` — send image/document
  - Parse inbound message webhooks
  - Handle delivery receipts (sent, delivered, read)
- [ ] Implement WhatsApp webhook handler: `POST /v1/webhooks/whatsapp`
  - Verify webhook signature
  - Process inbound messages → create inbound message record
  - Process delivery status updates → update message status
- [ ] Implement WhatsApp template submission (submit to Meta for approval)
- [ ] Test with WhatsApp sandbox/test number
- [ ] Wallet enforcement: verify tenant has sufficient wallet balance to cover message cost

**Deliverable:** WhatsApp sending and receiving working.

---

## Wednesday — SMS & Email Integration

- [ ] Implement `sms.rs` — Termii SMS client
  - `send_sms()` — send to Nigerian phone numbers
  - Sender ID configuration
  - Delivery status tracking
- [ ] Implement `email.rs` — Resend email client
  - `send_email()` — transactional email with template
  - HTML email templates (order confirmation, receipt, reset password)
  - From address: `noreply@punta.shop` or merchant's configured address
- [ ] Wire messaging into order lifecycle events:
  - `order.created` → send order confirmation (WhatsApp first, fallback SMS)
  - `order.paid` → send payment receipt (email)
  - `order.shipped` → send shipping notification with tracking (WhatsApp)
  - `order.delivered` → send delivery confirmation (WhatsApp)
- [ ] Wire password reset → send email
- [ ] Implement `POST /v1/messages/send` — send single message to customer
  - Select channel (whatsapp, sms, email)
  - Select template or custom text
  - Variable substitution

**Deliverable:** All three messaging channels working, automated order notifications active.

---

## Thursday — Campaign Engine

- [ ] Implement `campaigns.rs` — `CampaignService`
  - `create_campaign()` — select group, template, channel, schedule
  - `schedule_campaign()` — queue for future execution
  - `execute_campaign()` — fetch group members, send messages in batches
  - `cancel_campaign()` — cancel scheduled campaign
  - Progress tracking: total, sent, delivered, read, failed counts
- [ ] Implement batch sending with rate limiting:
  - WhatsApp: max 80 messages/sec (Meta limit)
  - SMS: max 30 messages/sec (Termii limit)
  - Email: max 100/sec (Resend limit)
  - Use Redis queue with controlled dequeue rate
- [ ] Background job: `campaign_sender.rs`
  - Pick up scheduled campaigns when time arrives
  - Process in batches
  - Update progress counts
  - Handle failures with retry
- [ ] Handlers: `GET/POST /v1/campaigns`, `PATCH /v1/campaigns/:id/cancel`

**Deliverable:** Campaign creation and execution working.

---

## Friday — Frontend: Messaging Pages

- [ ] Create `(dashboard)/messaging/page.tsx` — Messaging overview
  - Stats: Messages sent today, This month, Remaining quota
  - Recent messages list (last 20)
  - Quick send button
- [ ] Create send message modal:
  - Customer search/select
  - Channel selector
  - Template selector (with preview)
  - Variable override fields
  - Send button
- [ ] Create `(dashboard)/messaging/templates/page.tsx` — Template list
  - Table: Name, Channel, Status (draft/approved/rejected), Last Used
  - Create template form (name, channel, subject, body with variable hints)
- [ ] Create `(dashboard)/messaging/campaigns/page.tsx` — Campaign list
  - Table: Name, Channel, Group, Status, Sent/Total, Scheduled Date
  - Status badges: Draft, Scheduled, Sending, Sent, Cancelled
- [ ] Create campaign builder form:
  - Name, channel, template selection
  - Customer group selection (with member count preview)
  - Schedule date/time picker
  - Preview message
  - "Send Now" vs "Schedule" buttons
- [ ] Show messaging quota usage in sidebar/header

**Deliverable:** Complete messaging experience — send, templates, campaigns.

---

## Week 8 Checklist

```
✅ Multi-channel message dispatcher (WhatsApp → SMS → Email fallback)
✅ Message templates with variable substitution
✅ WhatsApp Cloud API integration (send/receive)
✅ SMS integration via Termii
✅ Email integration via Resend
✅ Automated order lifecycle notifications
✅ Password reset email working
✅ Campaign engine with batch sending
✅ Rate-limited message delivery
✅ Wallet-based pay-as-you-grow message cost deduction
✅ Messaging dashboard with stats
✅ Template management UI
✅ Campaign builder with scheduling
✅ Integration tests for message delivery
```
