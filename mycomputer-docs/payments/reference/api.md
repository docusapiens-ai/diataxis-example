# Payments Service API Documentation

## Overview

The Payments Service API provides comprehensive payment processing capabilities integrated with Stripe. This RESTful API handles payment transactions, refunds, subscriptions, and financial reporting for MyComputer's repair services across Europe.

**Base URL**: `https://api.mycomputer.eu/payments/v1`  
**Authentication**: Bearer token (JWT)  
**Content-Type**: `application/json`

## Authentication

All API requests require authentication using a JWT bearer token:

```http
Authorization: Bearer <your-jwt-token>
```

## API Endpoints

### Payments

#### Create Payment Intent

Creates a new payment intent for processing a payment.

**POST** `/payments/intent`

**Request Body:**
```json
{
  "amount": 15000,
  "currency": "EUR",
  "customer_id": "cust_123456789",
  "invoice_id": "inv_987654321",
  "payment_method_id": "pm_1234567890",
  "description": "Repair service - Order #12345",
  "metadata": {
    "order_id": "12345",
    "location_id": "loc_paris_01",
    "service_type": "screen_replacement"
  },
  "capture_method": "automatic",
  "setup_future_usage": "off_session"
}
```

**Response:**
```json
{
  "id": "pi_1234567890",
  "payment_id": "550e8400-e29b-41d4-a716-446655440000",
  "client_secret": "pi_1234567890_secret_abcdef",
  "amount": 15000,
  "currency": "EUR",
  "status": "requires_payment_method",
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Status Codes:**
- `201 Created` - Payment intent created successfully
- `400 Bad Request` - Invalid request parameters
- `404 Not Found` - Customer or invoice not found
- `500 Internal Server Error` - Server error

---

#### Confirm Payment

Confirms a payment intent and processes the payment.

**POST** `/payments/{payment_id}/confirm`

**Path Parameters:**
- `payment_id` - UUID of the payment

**Request Body:**
```json
{
  "payment_method_id": "pm_1234567890",
  "return_url": "https://mycomputer.eu/payment/complete"
}
```

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "succeeded",
  "amount": 15000,
  "currency": "EUR",
  "paid_at": "2024-01-15T10:31:00Z",
  "receipt_url": "https://pay.stripe.com/receipts/..."
}
```

---

#### Get Payment Details

Retrieves details of a specific payment.

**GET** `/payments/{payment_id}`

**Path Parameters:**
- `payment_id` - UUID of the payment

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_payment_intent_id": "pi_1234567890",
  "customer": {
    "id": "cust_123456789",
    "email": "john.doe@example.com",
    "name": "John Doe"
  },
  "amount": 15000,
  "currency": "EUR",
  "status": "succeeded",
  "description": "Repair service - Order #12345",
  "invoice_id": "inv_987654321",
  "payment_method": {
    "type": "card",
    "brand": "visa",
    "last4": "4242"
  },
  "metadata": {
    "order_id": "12345",
    "location_id": "loc_paris_01"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "paid_at": "2024-01-15T10:31:00Z"
}
```

---

#### List Payments

Retrieves a paginated list of payments.

**GET** `/payments`

**Query Parameters:**
- `customer_id` (optional) - Filter by customer ID
- `status` (optional) - Filter by status (pending, succeeded, failed)
- `from_date` (optional) - Start date (ISO 8601)
- `to_date` (optional) - End date (ISO 8601)
- `page` (optional, default: 1) - Page number
- `limit` (optional, default: 20, max: 100) - Items per page

**Response:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "amount": 15000,
      "currency": "EUR",
      "status": "succeeded",
      "customer_email": "john.doe@example.com",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "pages": 8
  }
}
```

---

### Refunds

#### Create Refund

Initiates a refund for a payment.

**POST** `/refunds`

**Request Body:**
```json
{
  "payment_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 5000,
  "reason": "requested_by_customer",
  "metadata": {
    "ticket_id": "TICK-12345",
    "approved_by": "manager_123"
  }
}
```

**Response:**
```json
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "payment_id": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 5000,
  "currency": "EUR",
  "status": "pending",
  "reason": "requested_by_customer",
  "created_at": "2024-01-16T14:20:00Z"
}
```

**Status Codes:**
- `201 Created` - Refund initiated successfully
- `400 Bad Request` - Invalid amount or payment already refunded
- `404 Not Found` - Payment not found
- `422 Unprocessable Entity` - Refund amount exceeds payment amount

---

#### Get Refund Status

Retrieves the status of a refund.

**GET** `/refunds/{refund_id}`

**Path Parameters:**
- `refund_id` - UUID of the refund

**Response:**
```json
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "payment_id": "550e8400-e29b-41d4-a716-446655440000",
  "stripe_refund_id": "re_1234567890",
  "amount": 5000,
  "currency": "EUR",
  "status": "succeeded",
  "reason": "requested_by_customer",
  "created_at": "2024-01-16T14:20:00Z",
  "processed_at": "2024-01-16T14:21:30Z"
}
```

---

### Customers

#### Create Customer

Creates a new customer profile for payment processing.

**POST** `/customers`

**Request Body:**
```json
{
  "email": "jane.smith@example.com",
  "name": "Jane Smith",
  "phone": "+33123456789",
  "billing_address": {
    "line1": "123 Rue de la Paix",
    "city": "Paris",
    "postal_code": "75001",
    "country": "FR"
  },
  "tax_id": "FR12345678901",
  "metadata": {
    "customer_type": "business",
    "company_name": "Tech Solutions SARL"
  }
}
```

**Response:**
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "stripe_customer_id": "cus_1234567890",
  "email": "jane.smith@example.com",
  "name": "Jane Smith",
  "created_at": "2024-01-15T09:00:00Z"
}
```

---

#### Update Customer

Updates customer information.

**PUT** `/customers/{customer_id}`

**Path Parameters:**
- `customer_id` - UUID of the customer

**Request Body:**
```json
{
  "name": "Jane Smith-Johnson",
  "billing_address": {
    "line1": "456 Avenue des Champs",
    "city": "Paris",
    "postal_code": "75008",
    "country": "FR"
  }
}
```

---

### Payment Methods

#### Add Payment Method

Adds a new payment method for a customer.

**POST** `/customers/{customer_id}/payment-methods`

**Path Parameters:**
- `customer_id` - UUID of the customer

**Request Body:**
```json
{
  "stripe_payment_method_id": "pm_1234567890",
  "type": "card",
  "is_default": true
}
```

**Response:**
```json
{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "customer_id": "770e8400-e29b-41d4-a716-446655440000",
  "type": "card",
  "details": {
    "brand": "visa",
    "last4": "4242",
    "exp_month": 12,
    "exp_year": 2025
  },
  "is_default": true,
  "created_at": "2024-01-15T10:00:00Z"
}
```

---

#### List Payment Methods

Retrieves all payment methods for a customer.

**GET** `/customers/{customer_id}/payment-methods`

**Path Parameters:**
- `customer_id` - UUID of the customer

**Response:**
```json
{
  "data": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440000",
      "type": "card",
      "details": {
        "brand": "visa",
        "last4": "4242"
      },
      "is_default": true,
      "created_at": "2024-01-15T10:00:00Z"
    },
    {
      "id": "990e8400-e29b-41d4-a716-446655440000",
      "type": "sepa_debit",
      "details": {
        "last4": "3000",
        "bank_name": "BNP Paribas"
      },
      "is_default": false,
      "created_at": "2024-01-10T15:00:00Z"
    }
  ]
}
```

---

### Invoices

#### Create Invoice

Creates a new invoice for services.

**POST** `/invoices`

**Request Body:**
```json
{
  "customer_id": "770e8400-e29b-41d4-a716-446655440000",
  "location_id": "loc_paris_01",
  "currency": "EUR",
  "due_date": "2024-02-15",
  "line_items": [
    {
      "description": "Screen replacement - iPhone 13",
      "quantity": 1,
      "unit_price": 12000,
      "tax_rate": 20
    },
    {
      "description": "Labor charge",
      "quantity": 1,
      "unit_price": 3000,
      "tax_rate": 20
    }
  ],
  "metadata": {
    "order_id": "12345",
    "technician_id": "tech_456"
  }
}
```

**Response:**
```json
{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "invoice_number": "INV-2024-0001",
  "customer_id": "770e8400-e29b-41d4-a716-446655440000",
  "subtotal": 15000,
  "tax_amount": 3000,
  "total": 18000,
  "currency": "EUR",
  "status": "draft",
  "due_date": "2024-02-15",
  "created_at": "2024-01-15T11:00:00Z"
}
```

---

#### Send Invoice

Sends an invoice to the customer.

**POST** `/invoices/{invoice_id}/send`

**Path Parameters:**
- `invoice_id` - UUID of the invoice

**Response:**
```json
{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "status": "sent",
  "sent_at": "2024-01-15T11:05:00Z",
  "email_sent_to": "jane.smith@example.com"
}
```

---

### Subscriptions

#### Create Subscription

Creates a recurring payment subscription.

**POST** `/subscriptions`

**Request Body:**
```json
{
  "customer_id": "770e8400-e29b-41d4-a716-446655440000",
  "plan_id": "plan_premium_monthly",
  "payment_method_id": "880e8400-e29b-41d4-a716-446655440000",
  "trial_period_days": 14,
  "metadata": {
    "contract_id": "CONTRACT-2024-001"
  }
}
```

**Response:**
```json
{
  "id": "bb0e8400-e29b-41d4-a716-446655440000",
  "stripe_subscription_id": "sub_1234567890",
  "customer_id": "770e8400-e29b-41d4-a716-446655440000",
  "plan_id": "plan_premium_monthly",
  "status": "trialing",
  "amount": 9900,
  "currency": "EUR",
  "billing_cycle": "monthly",
  "current_period_start": "2024-01-15T00:00:00Z",
  "current_period_end": "2024-01-29T00:00:00Z",
  "trial_end": "2024-01-29T00:00:00Z",
  "created_at": "2024-01-15T12:00:00Z"
}
```

---

#### Cancel Subscription

Cancels an active subscription.

**POST** `/subscriptions/{subscription_id}/cancel`

**Path Parameters:**
- `subscription_id` - UUID of the subscription

**Request Body:**
```json
{
  "cancel_at_period_end": true,
  "cancellation_reason": "too_expensive"
}
```

**Response:**
```json
{
  "id": "bb0e8400-e29b-41d4-a716-446655440000",
  "status": "active",
  "cancel_at_period_end": true,
  "canceled_at": "2024-01-20T10:00:00Z",
  "current_period_end": "2024-02-15T00:00:00Z"
}
```

---

### Webhooks

#### Stripe Webhook Handler

Handles incoming webhooks from Stripe.

**POST** `/webhooks/stripe`

**Headers:**
- `Stripe-Signature` - Stripe webhook signature for verification

**Request Body:**
```json
{
  "id": "evt_1234567890",
  "object": "event",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_1234567890",
      "amount": 15000,
      "currency": "eur",
      "status": "succeeded"
    }
  }
}
```

**Response:**
```json
{
  "received": true
}
```

**Supported Event Types:**
- `payment_intent.succeeded`
- `payment_intent.payment_failed`
- `charge.refunded`
- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `invoice.payment_succeeded`
- `invoice.payment_failed`

---

### Reports

#### Generate Financial Report

Generates financial reports for a specified period.

**GET** `/reports/financial`

**Query Parameters:**
- `start_date` (required) - Start date (ISO 8601)
- `end_date` (required) - End date (ISO 8601)
- `location_id` (optional) - Filter by location
- `format` (optional, default: json) - Output format (json, csv, pdf)

**Response:**
```json
{
  "period": {
    "start": "2024-01-01",
    "end": "2024-01-31"
  },
  "summary": {
    "total_revenue": 1250000,
    "total_refunds": 25000,
    "net_revenue": 1225000,
    "transaction_count": 450,
    "average_transaction_value": 2778,
    "currency": "EUR"
  },
  "by_payment_method": {
    "card": {
      "amount": 1000000,
      "count": 350
    },
    "sepa_debit": {
      "amount": 200000,
      "count": 80
    },
    "bank_transfer": {
      "amount": 50000,
      "count": 20
    }
  },
  "by_location": [
    {
      "location_id": "loc_paris_01",
      "location_name": "Paris Central",
      "amount": 500000,
      "count": 180
    }
  ]
}
```

---

## Error Handling

The API uses standard HTTP status codes and returns detailed error messages:

### Error Response Format

```json
{
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Payment processing failed",
    "details": {
      "reason": "insufficient_funds",
      "payment_method": "card_ending_4242"
    },
    "request_id": "req_1234567890",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_REQUEST` | 400 | Request validation failed |
| `AUTHENTICATION_FAILED` | 401 | Invalid or missing authentication token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `PAYMENT_FAILED` | 422 | Payment processing failed |
| `DUPLICATE_REQUEST` | 409 | Duplicate request detected |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

## Rate Limiting

API requests are rate-limited to ensure fair usage:

- **Standard tier**: 100 requests per minute
- **Premium tier**: 500 requests per minute

Rate limit information is included in response headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642248000
```

## Idempotency

To prevent duplicate transactions, use idempotency keys:

```http
Idempotency-Key: unique-request-id-12345
```

The API will return the same response for repeated requests with the same idempotency key within 24 hours.

## Versioning

The API version is included in the URL path. When breaking changes are introduced, a new version will be released:

- Current version: `v1`
- Legacy support: Previous versions supported for 12 months
- Deprecation notices: Sent via email and included in API responses

## Testing

### Test Environment

**Base URL**: `https://api-test.mycomputer.eu/payments/v1`

Use Stripe test cards for testing:
- Success: `4242 4242 4242 4242`
- Decline: `4000 0000 0000 0002`
- Authentication Required: `4000 0025 0000 3155`

### Postman Collection

Download our [Postman collection](https://api.mycomputer.eu/docs/payments-postman.json) for easy API testing.

## SDKs

Official SDKs are available for:
- Node.js: `npm install @mycomputer/payments-sdk`
- Python: `pip install mycomputer-payments`
- PHP: `composer require mycomputer/payments-sdk`

## Support

For API support:
- Documentation: https://docs.mycomputer.eu/payments
- Email: api-support@mycomputer.eu
- Slack: #payments-api-support
