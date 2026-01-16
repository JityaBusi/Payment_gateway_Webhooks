Async Payment Gateway

A production-ready, asynchronous payment gateway that allows merchants to accept payments using an embeddable checkout SDK.

The system is built around an event-driven architecture to handle high traffic reliably. Payments are processed asynchronously using Redis-backed job queues, resilient background workers, and secure HMAC-signed webhooks with automatic retries and exponential backoff.

🏗 Architecture Overview

The gateway follows an asynchronous, non-blocking flow to ensure scalability and fault tolerance.
Client requests return immediately, while payment processing and webhook delivery happen in the background.

High-Level Flow

The client (via SDK or API) creates a payment.

The API stores the payment in a pending state.

A background worker processes the payment asynchronously.

On completion, a webhook is delivered to the merchant.

Failed webhooks are retried automatically using exponential backoff.

sequenceDiagram
    participant User as User / SDK
    participant API as Gateway API
    participant DB as PostgreSQL
    participant Redis as Redis Queue
    participant Worker as Worker Service
    participant Merchant as Merchant Webhook

    User->>API: POST /payments
    API->>DB: Save payment (pending)
    API->>Redis: Enqueue ProcessPaymentJob
    API-->>User: 201 Created (pending)

    Redis->>Worker: ProcessPaymentJob
    Worker->>Worker: Simulate processing (5–10s)

    alt Payment success
        Worker->>DB: Update status = success
        Worker->>Redis: Enqueue DeliverWebhookJob
    else Payment failed
        Worker->>DB: Update status = failed
    end

    Redis->>Worker: DeliverWebhookJob
    Worker->>Worker: Generate HMAC signature
    Worker->>Merchant: POST /webhook

    alt Webhook success
        Worker->>DB: Log success
    else Webhook failed
        Worker->>DB: Log retry
        Worker->>Redis: Re-queue with delay
    end

🚀 Setup & Running the Project
Prerequisites

Docker

Docker Compose

Start All Services

Build and start the API, worker, Redis, Postgres, dashboard, and SDK:

docker-compose up -d --build

Running Services
Service	URL / Port	Purpose
API	http://localhost:8000
	Core payment gateway API
Dashboard	http://localhost:3000
	Webhook configuration & logs
Checkout SDK	http://localhost:3001
	Embeddable checkout widget
Redis	6379	Background job queue
Postgres	5432	Persistent data storage
🔌 API Reference
1. Create Payment

Creates a new payment request.
The API responds immediately with status pending.

POST /api/v1/payments

curl -X POST http://localhost:8000/api/v1/payments \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "currency": "INR",
    "method": "upi",
    "order_id": "ord_test_001",
    "vpa": "test@upi"
  }'

2. Capture Payment

Captures a successfully authorized payment.

POST /api/v1/payments/{id}/capture

curl -X POST http://localhost:8000/api/v1/payments/{payment_id}/capture \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789"

3. Refund Payment

Initiates a refund for a completed payment.

POST /api/v1/payments/{id}/refunds

curl -X POST http://localhost:8000/api/v1/payments/{payment_id}/refunds \
  -H "X-Api-Key: key_test_abc123" \
  -H "X-Api-Secret: secret_test_xyz789" \
  -H "Content-Type: application/json" \
  -d '{ "amount": 1000, "reason": "Customer requested" }'

4. Background Job Status (Testing)

Returns the current state of background jobs.

GET /api/v1/test/jobs/status

🔧 Environment Configuration

Environment variables are managed via docker-compose.yml.

Variable	Description	Default
DATABASE_URL	PostgreSQL connection string	jdbc:postgresql://postgres:5432/payment_gateway
REDIS_URL	Redis connection URL	redis://redis:6379
WEBHOOK_RETRY_INTERVALS_TEST	Use fast retry intervals for testing	false
SPRING_PROFILES_ACTIVE	Active service profile (default / worker)	default
📦 Checkout SDK Integration

Use the SDK to embed payments directly into your website.

1. Include the Script
<script src="http://localhost:3001/checkout.js"></script>

2. Initialize and Open Checkout
const gateway = new window.PaymentGateway({
  key: 'key_test_abc123',
  orderId: 'ORDER_123',
  onSuccess: (data) => console.log('Payment Success:', data),
  onFailure: (error) => console.error('Payment Failed:', error)
});

gateway.open();

🔒 Webhook Integration
1. Configure Webhook URL

Visit the dashboard to configure your webhook endpoint and view your signing secret:

http://localhost:3000/webhooks.html

2. Verify Webhook Signatures

Each webhook includes an X-Webhook-Signature header generated using HMAC-SHA256.

Node.js Example:

const crypto = require('crypto');

function verifyWebhook(req, secret) {
  const signature = req.headers['x-webhook-signature'];
  const payload = JSON.stringify(req.body); // must be raw JSON

  const expected = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');

  return signature === expected;
}

3. Webhook Retry Policy

If the webhook endpoint responds with a non-200 status, delivery is retried automatically:

Immediate

After 1 minute

After 5 minutes

After 30 minutes

After 2 hours

Testing Tip:
Set WEBHOOK_RETRY_INTERVALS_TEST=true to use 5-second retry intervals.

🧪 Testing the System

Start all services

docker-compose up


Test the checkout flow

Open
http://localhost:3001/checkout.html?order_id=TEST_1&key=key_test_abc123

Click Pay Now

Observe the loading and success state

Verify backend activity

Worker logs:

docker logs -f gateway_worker


Webhook logs:
http://localhost:3000/webhooks.html

📌 Project Tags
#payment-gateway
#async-architecture
#event-driven
#production-ready
