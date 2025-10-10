# Journey API Architecture & Documentation Blueprint

**Version:** 1.0
**Last Updated:** October 9, 2025
**Status:** Current State + Proposed Enhancements
**Purpose:** Complete architecture reference for creating Mintlify documentation

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Current API Endpoints](#current-api-endpoints)
4. [Proposed API Endpoints](#proposed-api-endpoints)
5. [Data Models](#data-models)
6. [Authentication](#authentication)
7. [Multi-Tenant Architecture](#multi-tenant-architecture)
8. [Payment Providers](#payment-providers)
9. [Business Flows](#business-flows)
10. [Admin Capabilities](#admin-capabilities)

---

## 🎯 Project Overview

**GetJourney** is a multi-tenant subscription billing and delivery management platform designed for:
- **Insurance companies** - Recurring premium billing
- **SaaS businesses** - Subscription management
- **E-commerce subscriptions** - Box subscriptions, meal kits, etc.

### Key Features
- Multi-tenant with complete data isolation
- Multiple payment provider support (Rapyd, Reepay, Straumur)
- Automatic dunning and retry logic
- Delivery scheduling and fulfillment
- Customer self-service portals
- Inventory management
- Coupon/discount system

### Technology Stack
- **Backend:** Django 4.2.6 + Django REST Framework
- **Database:** PostgreSQL 12.6 with tenant schemas
- **Frontend:** Next.js 14 (separate apps for admin/customer)
- **Task Queue:** Celery + Redis
- **API Version:** v1

---

## 🏗️ Architecture

### Application Structure

```
apps/
├── django/           # Backend API (port 7001)
│   ├── api/         # REST API endpoints
│   ├── order/       # Orders, subscriptions, payments
│   ├── delivery/    # Delivery management
│   ├── merchant/    # Products, merchant config
│   ├── communication/ # SMS/email events
│   ├── coupon/      # Discount codes
│   ├── tenants/     # Multi-tenant management
│   ├── webhooks/    # Webhook infrastructure
│   ├── claims/      # Insurance claims (Klar integration)
│   └── statistic/   # Analytics events
│
├── me/              # Customer portal (Next.js, port 3002)
├── dashboard/       # Admin dashboard (Next.js, port 3001)
└── react/           # Legacy (deprecated)
```

### Database Architecture
- **Multi-tenant:** Each merchant has isolated schema
- **ORM:** Django ORM for backend, Drizzle for Next.js
- **Migrations:** Per-tenant schema migrations
- **Connection:** Environment-based configuration

### API Base URL
```
Production:  https://api.journey.io
Staging:     https://staging-api.journey.io
Development: http://localhost:7001
```

### API Versioning
- Current: `/api/v1/`
- All endpoints prefixed with version

---

## 🔌 Current API Endpoints

### 1. Authentication

#### **POST /api/v1/send-sms-code/**
Send SMS login code to customer's phone
- **Auth:** Public
- **Body:** `{ "phone_number": "7771234" }`
- **Response:** 200 OK (code sent via SMS)
- **Use Case:** Passwordless login for customers

#### **POST /api/v1/send-email-code/**
Send email login code with magic link
- **Auth:** Public
- **Body:** `{ "email": "user@example.com" }`
- **Response:** 200 OK (code sent via email with magic URL)
- **Use Case:** Email-based passwordless login

#### **POST /api/v1/login/sms/code/**
Authenticate with phone number and code
- **Auth:** Public
- **Body:** `{ "phone_number": "7771234", "code": "1337", "login_type": "sms" }`
- **Response:** `{ "customer": {...}, "cookie": "session_key", "user": {...} }`
- **Use Case:** Complete passwordless login

#### **POST /api/v1/phone/**
Login with phone number (legacy)
- **Auth:** Public
- **Body:** `{ "phone_number": "7771234" }`
- **Response:** Session cookie
- **Status:** Legacy, prefer SMS code flow

#### **GET /api/v1/auth/**
Check authentication status
- **Auth:** Required (session)
- **Response:** `{ "user": "username", "id": 42, "is_superuser": false, "is_staff": false }`
- **Use Case:** Verify user is logged in

---

### 2. Customers

#### **GET /api/v1/customer/**
List all customers
- **Auth:** Required
- **Query Params:** None currently (🔮 PROPOSED: filters)
- **Response:** Array of customer objects
- **Use Case:** Admin customer listing

#### **POST /api/v1/customer/**
Create a new customer
- **Auth:** Required
- **Body:**
  ```json
  {
    "full_name": "Jón Jónsson",
    "email": "jon@example.is",
    "phone_number": "7771234",
    "address": "Laugavegur 1",
    "postal_code": "101",
    "city": "Reykjavík",
    "social_security_number": "010190-1234",
    "company_name": "Company Ltd",
    "invoice_notes": "Net 30 payment terms",
    "auto_send_invoices": false
  }
  ```
- **Response:** Customer object with auto-generated `customer_handle`
- **Use Case:** Customer registration

#### **GET /api/v1/customer/{id}/**
Get customer details
- **Auth:** Required
- **Response:** Full customer object
- **Use Case:** View customer profile

#### **PATCH /api/v1/customer/{id}/**
Update customer information
- **Auth:** Required
- **Body:** Partial customer fields
- **Response:** Updated customer object
- **Use Case:** Update customer details

#### **DELETE /api/v1/customer/{id}/**
Delete customer
- **Auth:** Required (admin)
- **Response:** 204 No Content
- **Use Case:** Remove customer account

#### **GET /api/v1/customer/{id}/subscriptions/**
List customer's subscriptions
- **Auth:** Required
- **Response:** Array of subscription objects
- **Use Case:** View all subscriptions for customer

#### **POST /api/v1/customer/{id}/pick-and-pack-notes/**
Update packing notes for customer
- **Auth:** Required
- **Body:** `{ "pickAndPack_notes": "Handle with care" }`
- **Response:** Updated customer
- **Use Case:** Add fulfillment instructions

---

### 3. Subscriptions

#### **GET /api/v1/subscription/**
List all subscriptions
- **Auth:** Required
- **Query Params:** None currently (🔮 PROPOSED: status, product_variation_id filters)
- **Response:** Array of subscription objects
- **Use Case:** Admin subscription listing

#### **POST /api/v1/subscription/**
Create a subscription
- **Auth:** Required
- **Body:**
  ```json
  {
    "customer_id": 42,
    "subscription_status": "active",
    "delivery_option_id": 3,
    "start_date": "2025-11-01"
  }
  ```
- **Response:** Subscription object
- **Note:** Must call `update_cart` separately to add items
- **Use Case:** Create new subscription

#### **GET /api/v1/subscription/{id}/**
Get subscription details
- **Auth:** Required
- **Response:** Full subscription object with order items
- **Use Case:** View subscription

#### **PATCH /api/v1/subscription/{id}/**
Update subscription
- **Auth:** Required
- **Body:** Partial subscription fields
- **Response:** Updated subscription
- **Use Case:** Modify subscription details

#### **POST /api/v1/subscription/{id}/pause/**
Pause subscription
- **Auth:** Required
- **Body:** `{ "reason": "Going on vacation" }` (optional)
- **Response:** Subscription with status `on_hold`
- **Use Case:** Temporarily stop billing

#### **POST /api/v1/subscription/{id}/resume/**
Resume paused subscription
- **Auth:** Required
- **Body:** `{ "coupon_code": "WELCOME10" }` (optional)
- **Response:** Subscription with status `active`
- **Use Case:** Restart billing

#### **POST /api/v1/subscription/{id}/update_cart/**
Update subscription items
- **Auth:** Required
- **Body:**
  ```json
  {
    "order_items": [
      {
        "product_variation_id": 5,
        "quantity": 2,
        "subscription_frequency_id": 2
      }
    ]
  }
  ```
- **Response:** Updated subscription
- **Use Case:** Change subscription contents

#### **POST /api/v1/subscription/{id}/update_frequency/**
Change billing frequency
- **Auth:** Required
- **Body:** `{ "subscription_frequency_id": 3 }`
- **Response:** Updated subscription
- **Use Case:** Change from monthly to quarterly

#### **GET /api/v1/subscription/{id}/deliveries/**
List deliveries for subscription
- **Auth:** Required
- **Response:** Array of delivery objects
- **Use Case:** View delivery history

#### **GET /api/v1/subscription/{id}/plan/** *(POST also supported)*
Get/modify subscription plan
- **Auth:** Required
- **Response:** Plan details
- **Use Case:** View or change plan

---

### 4. Orders

#### **GET /api/v1/order/**
List all orders
- **Auth:** Required
- **Response:** Array of order objects
- **Use Case:** Admin order listing

#### **POST /api/v1/order/**
Create an order
- **Auth:** Required
- **Body:**
  ```json
  {
    "customer_id": 42,
    "order_items": [
      {"product_variation_id": 5, "quantity": 1}
    ],
    "delivery_date": "2025-11-05"
  }
  ```
- **Response:** Order object
- **Use Case:** One-time purchase

#### **GET /api/v1/order/{id}/**
Get order details
- **Auth:** Required
- **Response:** Full order object with items
- **Use Case:** View order

#### **PATCH /api/v1/order/{id}/**
Update order
- **Auth:** Required
- **Body:** Partial order fields
- **Response:** Updated order
- **Use Case:** Modify order

#### **POST /api/v1/order/{id}/link_subscription/**
Link order to subscription
- **Auth:** Required
- **Body:** `{ "subscription_id": 789 }`
- **Response:** Updated order
- **Use Case:** Convert order to subscription billing

#### **POST /api/v1/order/{id}/update_items/**
Update order items
- **Auth:** Required
- **Body:** Array of order items
- **Response:** Updated order
- **Use Case:** Modify order contents

#### **POST /api/v1/order/{id}/change_delivery_date/**
Change delivery date
- **Auth:** Required
- **Body:** `{ "delivery_date": "2025-11-10" }`
- **Response:** Updated order
- **Use Case:** Reschedule delivery

#### **POST /api/v1/order/order-handler**
Order processing handler
- **Auth:** Required
- **Body:** Complex order object
- **Response:** Processed order
- **Use Case:** Internal order workflow

#### **POST /api/v1/order/settle-charge-or-set-plan**
Settle payment or set plan
- **Auth:** Required
- **Body:** Payment details
- **Response:** Payment result
- **Use Case:** Payment processing workflow

#### **GET /api/v1/order/order-confirmed**
Order confirmation page
- **Auth:** Public
- **Response:** Confirmation HTML/data
- **Use Case:** Post-checkout confirmation

---

### 5. Payments

#### **POST /api/v1/settle-balance/{operation}/**
Settle customer balance
- **Auth:** Required
- **Path Params:** `operation` (e.g., "charge", "credit")
- **Body:** Balance operation details
- **Response:** Updated balance
- **Use Case:** Adjust customer credits/debits

---

### 6. Payment Methods

#### **DELETE /api/v1/payment-method/delete**
Delete payment method
- **Auth:** Required
- **Body:** `{ "payment_method_id": 123 }`
- **Response:** 204 No Content
- **Use Case:** Remove saved card

---

### 7. Payment Providers

#### **Rapyd**

**POST /api/v1/rapyd/card-verification/**
Initiate card verification
- **Auth:** Required
- **Body:** Customer and card details
- **Response:** Verification URL and UUID
- **Use Case:** Verify card before charging

**GET /api/v1/rapyd/card-verification-callback/{uuid}/**
Card verification callback
- **Auth:** Public (signed)
- **Response:** Verification result
- **Use Case:** Rapyd redirect after verification

**POST /api/v1/rapyd/check-card-verification/**
Check verification status
- **Auth:** Required
- **Body:** `{ "uuid": "550e8400..." }`
- **Response:** Verification status
- **Use Case:** Poll verification status

**POST /api/v1/rapyd/create-virtual-card/**
Create virtual card token
- **Auth:** Required
- **Body:** Card details
- **Response:** Virtual card token
- **Use Case:** Tokenize card for recurring billing

#### **Reepay**

**POST /api/v1/reepay/add-card**
Add payment card
- **Auth:** Required
- **Body:** Card token
- **Response:** Payment method object
- **Use Case:** Save card to customer

**POST /api/v1/reepay/session/**
Create checkout session
- **Auth:** Required
- **Body:** Session configuration
- **Response:** Session ID and URL
- **Use Case:** Hosted checkout page

**GET /api/v1/reepay/subscription-customer-payment-provider/**
Get subscription info from Reepay
- **Auth:** Required
- **Query:** `customer_handle`
- **Response:** Reepay subscription data
- **Use Case:** Sync with Reepay

**GET /api/v1/reepay/customer-list-of-payment-methods-payment-provider/**
List payment methods from Reepay
- **Auth:** Required
- **Query:** `customer_handle`
- **Response:** Array of payment methods
- **Use Case:** Show saved cards

#### **Straumur**

**POST /api/v1/straumur/create-checkout/**
Create hosted checkout
- **Auth:** Required
- **Body:**
  ```json
  {
    "amount": 15000,
    "currency": "ISK",
    "reference": "order-123",
    "return_url": "https://example.com/success"
  }
  ```
- **Response:** Checkout URL
- **Use Case:** Redirect to Straumur payment page

**POST /api/v1/straumur/create-session/**
Create checkout session
- **Auth:** Required
- **Body:** Session configuration
- **Response:** Session ID and URL
- **Use Case:** Embedded checkout

**POST /api/v1/straumur/create-session-add-card/**
Create add-card session
- **Auth:** Required
- **Body:** `{ "customer_id": 42, "return_url": "..." }`
- **Response:** `{ "checkoutUrl": "...", "uuid": "..." }`
- **Use Case:** Add card without charging

**GET /api/v1/straumur/checkout-status/**
Check checkout status
- **Auth:** Required
- **Query:** `uuid`
- **Response:** Checkout status and payment method details
- **Use Case:** Verify payment completed

**GET /api/v1/straumur/checkout-callback/{uuid}/**
Checkout callback handler
- **Auth:** Public (signed)
- **Response:** Redirect or status
- **Use Case:** Straumur redirect after checkout

**POST /api/v1/straumur/webhook/**
Straumur webhook receiver
- **Auth:** Webhook signature
- **Body:** Webhook event
- **Response:** 200 OK
- **Use Case:** Receive payment notifications

---

### 8. Deliveries

#### **GET /api/v1/delivery/**
List deliveries
- **Auth:** Required
- **Response:** Array of delivery objects
- **Use Case:** View all deliveries

#### **GET /api/v1/delivery/{id}/**
Get delivery details
- **Auth:** Required
- **Response:** Full delivery object
- **Use Case:** View delivery

#### **PATCH /api/v1/delivery/{id}/**
Update delivery
- **Auth:** Required
- **Body:** Partial delivery fields
- **Response:** Updated delivery
- **Use Case:** Modify delivery

#### **GET /api/v1/delivery/{id}/details/**
Get detailed delivery info
- **Auth:** Required
- **Response:** Delivery with option details
- **Use Case:** View full delivery configuration

#### **POST /api/v1/delivery/{id}/reset/**
Reset delivery status
- **Auth:** Required (admin)
- **Response:** Reset delivery
- **Use Case:** Undo delivery status

#### **POST /api/v1/delivery/{id}/charge/**
Manually charge for delivery
- **Auth:** Required (admin)
- **Response:** Payment result
- **Use Case:** Force payment attempt

#### **POST /api/v1/delivery/{id}/update_packed/**
Update packing status
- **Auth:** Required
- **Body:** `{ "packed": true }`
- **Response:** Updated delivery
- **Use Case:** Mark as packed

#### **POST /api/v1/delivery/{id}/change_delivery_option/**
Change shipping method
- **Auth:** Required
- **Body:** `{ "delivery_option_id": 5 }`
- **Response:** Updated delivery
- **Use Case:** Switch to different carrier

#### **GET /api/v1/delivery/delivered/{id}/{status}**
Mark as delivered (legacy)
- **Auth:** Public (from route planner)
- **Response:** Updated delivery
- **Use Case:** Driver marks delivered

#### **POST /api/v1/delivery-update/**
Delivery status update webhook
- **Auth:** Public (from route planner)
- **Body:** Route planner webhook payload
- **Response:** 200 OK
- **Use Case:** Receive status from Routific/Dropp

#### **GET /api/v1/delivery-option/**
List delivery options
- **Auth:** Required
- **Query Params:**
  - `product_variation`: Filter by product
  - `delivery_schedule`: Filter by schedule
  - `postal_code`: Filter by postal code
- **Response:** Array of available delivery options
- **Use Case:** Show shipping methods at checkout

#### **GET /api/v1/delivery-schedule/**
List delivery schedules
- **Auth:** Required
- **Response:** Array of schedules (postal codes, weekdays)
- **Use Case:** Show available delivery windows

#### **GET /api/v1/delivery-partner/**
List delivery partners
- **Auth:** Required
- **Response:** Array of carriers
- **Use Case:** Show available carriers

#### **GET /api/v1/delivery-list-subscriptions/**
List deliveries by subscription
- **Auth:** Required
- **Response:** Deliveries grouped by subscription
- **Use Case:** Admin delivery overview

#### **GET /api/v1/deliveries-for-subscription/**
Get deliveries for specific subscription
- **Auth:** Required
- **Query:** `subscription_id`
- **Response:** Array of deliveries
- **Use Case:** Subscription delivery history

---

### 9. Products

#### **GET /api/v1/product/**
List products
- **Auth:** Public
- **Response:** Array of product objects
- **Use Case:** Product catalog

#### **GET /api/v1/product/{id}/**
Get product details
- **Auth:** Public
- **Response:** Full product object
- **Use Case:** Product page

#### **PATCH /api/v1/product/{id}/**
Update product
- **Auth:** Required (admin)
- **Body:** Partial product fields
- **Response:** Updated product
- **Use Case:** Edit product

#### **GET /api/v1/product-variations/**
List product variations
- **Auth:** Public
- **Query Params:**
  - `product_name`: Filter by product URL name
  - `type`: Filter by variation type
  - `hidden`: Filter by visibility (true/false)
- **Response:** Array of variations (SKUs)
- **Use Case:** Show product options

#### **GET /api/v1/stock/**
List stock levels
- **Auth:** Required
- **Query Params:**
  - `search`: Search by SKU or name
- **Response:** Array of stock items with quantities
- **Use Case:** Inventory management

---

### 10. Coupons

#### **GET /api/v1/coupons/**
List coupons
- **Auth:** Required
- **Response:** Array of coupon objects
- **Use Case:** View available discounts

#### **POST /api/v1/coupons/**
Create coupon
- **Auth:** Required (admin)
- **Body:**
  ```json
  {
    "code": "SAVE20",
    "discount_type": "percentage",
    "amount": 20,
    "validity_start_time": "2025-11-01",
    "validity_end_time": "2025-12-31"
  }
  ```
- **Response:** Coupon object
- **Use Case:** Create discount code

#### **GET /api/v1/coupons/{id}/**
Get coupon details
- **Auth:** Required
- **Response:** Full coupon object
- **Use Case:** View coupon

#### **PATCH /api/v1/coupons/{id}/**
Update coupon
- **Auth:** Required (admin)
- **Body:** Partial coupon fields
- **Response:** Updated coupon
- **Use Case:** Edit coupon

#### **DELETE /api/v1/coupons/{id}/**
Delete coupon
- **Auth:** Required (admin)
- **Response:** 204 No Content
- **Use Case:** Remove coupon

---

### 11. Merchant

#### **GET /api/v1/merchant/**
List merchants
- **Auth:** Required
- **Response:** Array of merchant objects
- **Use Case:** Get merchant configuration

#### **GET /api/v1/merchant/{id}/**
Get merchant details
- **Auth:** Required
- **Response:** Full merchant object
- **Use Case:** View merchant settings

#### **GET /api/v1/merchant/{id}/subscription_frequencies/**
Get available billing frequencies
- **Auth:** Public
- **Response:** Array of frequency options (monthly, quarterly, etc.)
- **Use Case:** Show billing options

---

### 12. Analytics & Reporting

#### **GET /api/v1/dashboard/**
Get dashboard overview
- **Auth:** Required (admin)
- **Response:** Dashboard metrics
- **Use Case:** Admin dashboard

#### **GET /api/v1/dashboard/deliveries-for-subscription**
Get delivery details for dashboard
- **Auth:** Required (admin)
- **Query:** Subscription filters
- **Response:** Delivery breakdown
- **Use Case:** Dashboard drill-down

#### **GET /api/v1/journey/**
Get customer journey data
- **Auth:** Required
- **Response:** Customer portal data
- **Use Case:** "My Journey" customer view

---

### 13. Scanning & Operations

#### **POST /api/v1/scan/production-event/**
Log production scan event
- **Auth:** Required
- **Body:** `{ "event_type": "...", "reference": "..." }`
- **Response:** 201 Created
- **Use Case:** Track production workflow

#### **POST /api/v1/scan/pick-event/**
Log pick/pack scan event
- **Auth:** Required
- **Body:** `{ "event_type": "...", "reference": "..." }`
- **Response:** 201 Created
- **Use Case:** Track fulfillment

#### **POST /api/v1/google-sheet-label-printing/**
Print labels from Google Sheet
- **Auth:** Required (admin)
- **Body:** Sheet configuration
- **Response:** Print job status
- **Use Case:** Bulk label printing

---

### 14. Content & Pages

#### **GET /api/v1/pages/**
List CMS pages
- **Auth:** Public
- **Response:** Array of page objects
- **Use Case:** Dynamic content pages

#### **POST /api/v1/pages/**
Create page
- **Auth:** Required (admin)
- **Body:** `{ "name": "...", "page_name_slug": "...", "content": "..." }`
- **Response:** Page object
- **Use Case:** Add content page

#### **GET /api/v1/pages/{id}/**
Get page content
- **Auth:** Public
- **Response:** Full page object
- **Use Case:** Render page

#### **PATCH /api/v1/pages/{id}/**
Update page
- **Auth:** Required (admin)
- **Body:** Partial page fields
- **Response:** Updated page
- **Use Case:** Edit page

#### **GET /api/v1/pdf-templates/**
List PDF templates
- **Auth:** Required
- **Response:** Array of templates
- **Use Case:** Invoice templates

---

### 15. Utilities

#### **GET /api/v1/ping/**
Health check
- **Auth:** Public
- **Response:** 200 OK
- **Use Case:** Monitor API health

#### **GET /api/v1/schema/**
Get API schema
- **Auth:** Public
- **Response:** OpenAPI/Swagger schema
- **Use Case:** API documentation

#### **GET /api/v1/me/**
Get current user info
- **Auth:** Required
- **Response:** User object
- **Use Case:** Check logged-in user

#### **GET /api/v1/head_block/**
Get merchant tracking scripts
- **Auth:** Public
- **Response:** HTML head block
- **Use Case:** Inject tracking codes

#### **POST /api/v1/log_event/**
Log analytics event
- **Auth:** Required
- **Body:** `{ "event_type": "...", "data": {...} }`
- **Response:** 200 OK
- **Use Case:** Track user actions

#### **POST /api/internal/testing-setup/**
Setup test environment
- **Auth:** Required (internal)
- **Response:** Test data created
- **Use Case:** E2E testing

---

## 🔮 Proposed API Endpoints

*These endpoints are planned but not yet implemented. Mark as "Coming Soon" in docs.*

### Payment Management

#### **GET /api/v1/payment/** ⚠️ PROPOSED
List all payments (invoices)
- **Query Params:**
  - `customer_id`: Filter by customer
  - `subscription_id`: Filter by subscription
  - `status`: Filter by payment status
  - `from_date`: Date range start
  - `to_date`: Date range end
  - `limit`: Pagination limit
  - `offset`: Pagination offset
- **Response:** Paginated payment list
- **Priority:** 🔥 Critical

#### **GET /api/v1/payment/{id}/** ⚠️ PROPOSED
Get payment details
- **Response:** Full payment object with line items
- **Priority:** 🔥 Critical

#### **POST /api/v1/payment/{id}/retry/** ⚠️ PROPOSED
Manually retry failed payment
- **Body:** `{ "force": true }`
- **Response:** Payment result
- **Priority:** 🔥 Critical

#### **POST /api/v1/payment/{id}/refund/** ⚠️ PROPOSED
Refund payment
- **Body:** `{ "amount": 15000, "reason": "Customer request" }`
- **Response:** Refund object
- **Priority:** 🔥 Critical

#### **GET /api/v1/payment/{id}/receipt/** ⚠️ PROPOSED
Get payment receipt/invoice PDF
- **Response:** PDF file
- **Priority:** 🔴 High

#### **POST /api/v1/payment/export/** ⚠️ PROPOSED
Export payments to CSV/Excel
- **Body:** Filter criteria
- **Response:** CSV/Excel file
- **Priority:** 🔴 High

---

### Payment Method Management

#### **GET /api/v1/customer/{id}/payment-methods/** ⚠️ PROPOSED
List customer's payment methods
- **Response:** Array of payment method objects
- **Priority:** 🔥 Critical

#### **GET /api/v1/payment-method/{id}/** ⚠️ PROPOSED
Get payment method details
- **Response:** Payment method object
- **Priority:** 🔥 Critical

#### **PATCH /api/v1/payment-method/{id}/** ⚠️ PROPOSED
Update payment method status
- **Body:** `{ "status": "inactive" }`
- **Response:** Updated payment method
- **Priority:** 🔴 High

#### **POST /api/v1/payment-method/{id}/set-default/** ⚠️ PROPOSED
Set as default payment method
- **Response:** Updated customer with new default
- **Priority:** 🔥 Critical

---

### Subscription Management

#### **POST /api/v1/subscription/{id}/cancel/** ⚠️ PROPOSED
Cancel subscription
- **Body:**
  ```json
  {
    "reason": "Switching provider",
    "cancel_at_period_end": true,
    "refund_unused": false
  }
  ```
- **Response:** Cancelled subscription
- **Priority:** 🔥 Critical

#### **POST /api/v1/subscription/{id}/reactivate/** ⚠️ PROPOSED
Reactivate cancelled subscription
- **Response:** Active subscription
- **Priority:** 🔴 High

#### **GET /api/v1/subscription/{id}/upcoming-invoice/** ⚠️ PROPOSED
Preview next invoice
- **Response:** Invoice preview with line items and total
- **Priority:** 🔴 High

#### **POST /api/v1/subscription/{id}/change-payment-method/** ⚠️ PROPOSED
Switch payment method
- **Body:** `{ "payment_method_id": 125 }`
- **Response:** Updated subscription
- **Priority:** 🔴 High

#### **GET /api/v1/subscription/?status={status}** ⚠️ PROPOSED
Filter subscriptions by status
- **Query:** `status` (active, past_due, on_hold, expired)
- **Response:** Filtered subscriptions
- **Priority:** 🔴 High

---

### Order Management

#### **POST /api/v1/order/{id}/charge/** ⚠️ PROPOSED
Manually charge order
- **Response:** Payment result
- **Priority:** 🔥 Critical

#### **POST /api/v1/order/{id}/refund/** ⚠️ PROPOSED
Refund order
- **Body:** `{ "amount": 15000, "reason": "..." }`
- **Response:** Refund object
- **Priority:** 🔥 Critical

---

### Customer Management

#### **POST /api/v1/customer/export/** ⚠️ PROPOSED
Export customers to CSV/Excel
- **Body:** Filter criteria
- **Response:** CSV/Excel file
- **Priority:** 🔴 High

#### **GET /api/v1/customer/{id}/payments/** ⚠️ PROPOSED
List customer's payment history
- **Response:** Array of payments
- **Priority:** 🔥 Critical

#### **GET /api/v1/customer/{id}/balance/** ⚠️ PROPOSED
Get customer balance (credits/debits)
- **Response:** Balance object
- **Priority:** 🔴 High

#### **POST /api/v1/customer/{id}/balance/** ⚠️ PROPOSED
Adjust customer balance
- **Body:** `{ "amount": 5000, "reason": "..." }`
- **Response:** Updated balance
- **Priority:** 🔴 High

#### **POST /api/v1/customer/{id}/send-password-reset/** ⚠️ PROPOSED
Trigger password reset email
- **Response:** 200 OK
- **Priority:** 🟡 Medium

---

### Delivery Management

#### **POST /api/v1/delivery/export/** ⚠️ PROPOSED
Export deliveries to CSV/Excel
- **Body:** Filter criteria
- **Response:** CSV/Excel file
- **Priority:** 🔴 High

#### **GET /api/v1/delivery/{id}/exchanges/** ⚠️ PROPOSED
List delivery exchanges/returns
- **Response:** Array of exchange records
- **Priority:** 🟡 Medium

#### **POST /api/v1/delivery/{id}/exchange/** ⚠️ PROPOSED
Create exchange/return record
- **Body:** Exchange details
- **Response:** Exchange object
- **Priority:** 🟡 Medium

---

### Coupon Management

#### **POST /api/v1/coupon/{code}/validate/** ⚠️ PROPOSED
Validate coupon code
- **Body:** `{ "customer_id": 42, "order_total": 15000 }`
- **Response:** Validation result with discount amount
- **Priority:** 🔴 High

#### **POST /api/v1/subscription/{id}/apply-coupon/** ⚠️ PROPOSED
Apply coupon to subscription
- **Body:** `{ "coupon_code": "SAVE20" }`
- **Response:** Updated subscription with discount
- **Priority:** 🔴 High

#### **POST /api/v1/delivery/{id}/apply-coupon/** ⚠️ PROPOSED
Apply coupon to delivery
- **Body:** `{ "coupon_code": "SAVE20" }`
- **Response:** Updated delivery with discount
- **Priority:** 🔴 High

---

### Inventory Management

#### **POST /api/v1/stock/** ⚠️ PROPOSED
Create stock item
- **Body:** Stock item details
- **Response:** Stock object
- **Priority:** 🟡 Medium

#### **PATCH /api/v1/stock/{id}/** ⚠️ PROPOSED
Update stock levels
- **Body:** `{ "current_quantity": 100 }`
- **Response:** Updated stock
- **Priority:** 🟡 Medium

#### **GET /api/v1/stock/{id}/movements/** ⚠️ PROPOSED
Get stock movement history
- **Response:** Array of movements
- **Priority:** 🟡 Medium

#### **POST /api/v1/stock/{id}/recalculate/** ⚠️ PROPOSED
Recalculate stock from movements
- **Response:** Recalculated stock
- **Priority:** 🟡 Medium

#### **GET /api/v1/stock-movement/** ⚠️ PROPOSED
List stock movements
- **Query Params:** Filters
- **Response:** Array of movements
- **Priority:** 🟡 Medium

#### **POST /api/v1/stock-movement/** ⚠️ PROPOSED
Create stock movement
- **Body:** Movement details
- **Response:** Movement object
- **Priority:** 🟡 Medium

#### **GET /api/v1/stock-alert/** ⚠️ PROPOSED
List stock alerts
- **Query:** `status` filter
- **Response:** Array of alerts
- **Priority:** 🟡 Medium

#### **POST /api/v1/stock-alert/{id}/acknowledge/** ⚠️ PROPOSED
Acknowledge stock alert
- **Response:** Updated alert
- **Priority:** 🟡 Medium

---

### Merchant Configuration

#### **GET /api/v1/merchant/settings/** ⚠️ PROPOSED
Get merchant settings
- **Response:** Full merchant configuration
- **Priority:** 🔴 High

#### **PATCH /api/v1/merchant/settings/** ⚠️ PROPOSED
Update merchant settings
- **Body:** Partial settings
- **Response:** Updated settings
- **Priority:** 🔴 High

#### **PATCH /api/v1/merchant/dunning-config/** ⚠️ PROPOSED
Update dunning configuration
- **Body:**
  ```json
  {
    "dunning_settling_attempts": 20,
    "failed_payment_cancelled_days": 30
  }
  ```
- **Response:** Updated config
- **Priority:** 🔥 Critical

#### **GET /api/v1/merchant/branding/** ⚠️ PROPOSED
Get logos and CSS
- **Response:** Branding assets
- **Priority:** 🟡 Medium

#### **POST /api/v1/merchant/branding/logo/** ⚠️ PROPOSED
Upload merchant logo
- **Body:** File upload
- **Response:** Logo URL
- **Priority:** 🟡 Medium

---

### Product Management

#### **POST /api/v1/product/** ⚠️ PROPOSED
Create product
- **Body:** Product details
- **Response:** Product object
- **Priority:** 🟡 Medium

#### **DELETE /api/v1/product/{id}/** ⚠️ PROPOSED
Delete product
- **Response:** 204 No Content
- **Priority:** 🟡 Medium

#### **POST /api/v1/product/{id}/image/** ⚠️ PROPOSED
Upload product image
- **Body:** File upload
- **Response:** Image URL
- **Priority:** 🟡 Medium

#### **POST /api/v1/product-variation/** ⚠️ PROPOSED
Create product variation
- **Body:** Variation details
- **Response:** Variation object
- **Priority:** 🟡 Medium

#### **PATCH /api/v1/product-variation/{id}/** ⚠️ PROPOSED
Update variation
- **Body:** Partial variation fields
- **Response:** Updated variation
- **Priority:** 🟡 Medium

#### **DELETE /api/v1/product-variation/{id}/** ⚠️ PROPOSED
Delete variation
- **Response:** 204 No Content
- **Priority:** 🟡 Medium

---

### Product Organization

#### **GET /api/v1/product-category/** ⚠️ PROPOSED
List product categories
- **Response:** Array of categories
- **Priority:** 🟡 Medium

#### **POST /api/v1/product-category/** ⚠️ PROPOSED
Create category
- **Body:** Category details
- **Response:** Category object
- **Priority:** 🟡 Medium

#### **GET /api/v1/product-tag/** ⚠️ PROPOSED
List product tags
- **Response:** Array of tags
- **Priority:** 🟡 Medium

#### **POST /api/v1/product-tag/** ⚠️ PROPOSED
Create tag
- **Body:** Tag details
- **Response:** Tag object
- **Priority:** 🟡 Medium

#### **GET /api/v1/product-attribute/** ⚠️ PROPOSED
List product attributes
- **Response:** Array of attributes (Color, Size, etc.)
- **Priority:** 🟡 Medium

---

### Webhooks

#### **POST /api/v1/webhook/** ⚠️ PROPOSED
Register webhook endpoint
- **Body:**
  ```json
  {
    "url": "https://example.com/webhooks",
    "events": ["payment.succeeded", "subscription.past_due"],
    "secret": "whsec_..."
  }
  ```
- **Response:** Webhook object
- **Priority:** 🔥 Critical

#### **GET /api/v1/webhook/** ⚠️ PROPOSED
List registered webhooks
- **Response:** Array of webhooks
- **Priority:** 🔥 Critical

#### **DELETE /api/v1/webhook/{id}/** ⚠️ PROPOSED
Delete webhook
- **Response:** 204 No Content
- **Priority:** 🔥 Critical

#### **GET /api/v1/webhook/{id}/events/** ⚠️ PROPOSED
List webhook event log
- **Response:** Array of sent events
- **Priority:** 🔴 High

---

### Communication

#### **GET /api/v1/event/** ⚠️ PROPOSED
List event types
- **Response:** Array of events
- **Priority:** 🟡 Medium

#### **PATCH /api/v1/event/{id}/** ⚠️ PROPOSED
Update event configuration
- **Body:** Event settings
- **Response:** Updated event
- **Priority:** 🟡 Medium

#### **POST /api/v1/event/{id}/trigger/** ⚠️ PROPOSED
Manually trigger event
- **Body:** Event context
- **Response:** 200 OK
- **Priority:** 🟡 Medium

#### **GET /api/v1/message-template/** ⚠️ PROPOSED
List message templates
- **Response:** Array of templates
- **Priority:** 🟡 Medium

#### **POST /api/v1/message-template/** ⚠️ PROPOSED
Create message template
- **Body:** Template details
- **Response:** Template object
- **Priority:** 🟡 Medium

#### **GET /api/v1/message-log/** ⚠️ PROPOSED
View sent messages
- **Query:** Filter by customer, event, date
- **Response:** Array of message logs
- **Priority:** 🔴 High

---

### Audit & Compliance

#### **GET /api/v1/audit-log/** ⚠️ PROPOSED
List audit logs
- **Query Params:**
  - `model`: Filter by model (subscription, order, etc.)
  - `action`: Filter by action (create, update, delete)
  - `initiator`: Filter by user
  - `instance_pk`: Filter by object ID
  - `from_date`: Date range
  - `to_date`: Date range
- **Response:** Array of audit logs
- **Priority:** 🔥 Critical

#### **GET /api/v1/{resource}/{id}/audit-log/** ⚠️ PROPOSED
Get audit trail for specific object
- **Response:** Audit history
- **Priority:** 🔥 Critical

---

### Analytics & Reporting

#### **GET /api/v1/reports/failed-payments/** ⚠️ PROPOSED
Failed payments report
- **Query:** Date range
- **Response:** Report data
- **Priority:** 🔴 High

#### **GET /api/v1/reports/churn/** ⚠️ PROPOSED
Churn analysis
- **Query:** Period
- **Response:** Churn metrics
- **Priority:** 🔴 High

#### **GET /api/v1/reports/mrr/** ⚠️ PROPOSED
Monthly recurring revenue
- **Query:** Date range
- **Response:** MRR data
- **Priority:** 🔴 High

#### **GET /api/v1/reports/payment-success-rate/** ⚠️ PROPOSED
Payment health metrics
- **Query:** Date range
- **Response:** Success rates
- **Priority:** 🔴 High

---

### Multi-Tenant Management

#### **GET /api/internal/tenant/** ⚠️ PROPOSED
List tenants (superuser only)
- **Response:** Array of tenants
- **Priority:** 🟢 Low

#### **POST /api/internal/tenant/** ⚠️ PROPOSED
Create tenant (superuser only)
- **Body:** Tenant configuration
- **Response:** Tenant object
- **Priority:** 🟢 Low

---

## 📊 Data Models

### Customer
```json
{
  "id": 42,
  "customer_handle": "cust-journey-42",
  "phone_number": "7771234",
  "full_name": "Jón Jónsson",
  "email": "jon@example.is",
  "address": "Laugavegur 1",
  "address2": "",
  "city": "Reykjavík",
  "postal_code": "101",
  "social_security_number": "010190-1234",
  "company_name": "Example Ltd",
  "invoice_notes": "Net 30",
  "delivery_detail": "Ring doorbell",
  "pickAndPack_notes": "Handle with care",
  "auto_send_invoices": false,
  "date_time": "2025-10-09T14:23:45Z",
  "merchant_id": 1
}
```

### Subscription
```json
{
  "id": 789,
  "customer_id": 42,
  "subscription_status": "active",
  "start_date": "2025-11-01",
  "delivery_option_id": 3,
  "order_items": [
    {
      "id": 101,
      "product_variation_id": 5,
      "quantity": 2,
      "subscription_frequency_id": 2
    }
  ],
  "created": "2025-10-09T14:23:45Z"
}
```

### Order
```json
{
  "id": 456,
  "customer_id": 42,
  "customer_receiver_id": null,
  "payment_id": 789,
  "subscription_id": 789,
  "gift": false,
  "date_time": "2025-11-01T00:00:00Z",
  "created_at": "2025-10-31T12:00:00Z",
  "order_items": [
    {
      "id": 1001,
      "product_variation_id": 5,
      "quantity": 2
    }
  ]
}
```

### Payment
```json
{
  "id": 1234,
  "order_id": 456,
  "payment_status": "settled",
  "payment_method_id": 125,
  "payment_reference": "ch_1234567890",
  "line_items": "2x Product Name @ 7500 ISK",
  "line_items_json": [
    {
      "ordertext": "Product Name",
      "quantity": 2,
      "amount": 7500,
      "vat": 0.24
    }
  ],
  "created": "2025-11-01T00:00:00Z",
  "last_attempt": "2025-11-01T00:05:23Z",
  "settling_attempts": 1,
  "response": {...}
}
```

### Payment Method
```json
{
  "id": 125,
  "customer_id": 42,
  "payment_processor_id": 1,
  "virtual_card_token": "tok_...",
  "masked_card_number": "************4242",
  "card_type": "visa",
  "expiration_date": "12/2025",
  "status": "active",
  "created": "2025-10-08T12:00:00Z"
}
```

### Delivery
```json
{
  "id": 5678,
  "order_id": 456,
  "subscription_id": 789,
  "delivery_date": "2025-11-05",
  "delivered": false,
  "cancelled": false,
  "packed": false,
  "delivery_option_selected_id": 12,
  "greeting_card_text": "Happy Birthday!"
}
```

### Product
```json
{
  "id": 10,
  "name": "Car Insurance Premium",
  "url_name": "car-insurance",
  "short_description": "Comprehensive car insurance",
  "image": "/media/products/car.jpg",
  "subscription_frequencies": [2, 3],
  "categories": [1],
  "tags": [5],
  "current_quantity": 1000
}
```

### Coupon
```json
{
  "id": 50,
  "code": "SAVE20",
  "discount_type": "percentage",
  "amount": 20,
  "description": "20% off first month",
  "validity_start_time": "2025-11-01",
  "validity_end_time": "2025-12-31",
  "activation_start_time": "2025-11-01",
  "activation_end_time": "2025-11-30",
  "fixed_deliveries": 1,
  "product_variations": [5, 6]
}
```

---

## 🔐 Authentication

### Session-Based (Current)
1. Customer requests login code via SMS/email
2. Code sent to phone/email
3. Customer submits code
4. Session cookie returned
5. Cookie used for subsequent requests

### API Key (Planned)
- Bearer token authentication
- Merchant-level API keys
- Scoped permissions

---

## 🏢 Multi-Tenant Architecture

### Tenant Isolation
- Each merchant has separate PostgreSQL schema
- Complete data isolation
- Shared application code
- Domain-based routing (subdomain.journey.io)

### Tenant Configuration
- Payment processor (per tenant)
- Email/SMS providers (per tenant)
- Branding (logos, CSS)
- Dunning settings
- Business rules

---

## 💳 Payment Providers

### Supported Providers
1. **Rapyd** - Card verification and tokenization
2. **Reepay** - Nordic recurring billing
3. **Straumur** - Icelandic payment gateway

### Payment Flow
1. Customer adds card via provider checkout
2. Card tokenized and stored as payment_method
3. Recurring charges use stored token
4. Payment status tracked in payment object
5. Webhooks notify of status changes

### Dunning Logic
- Configured per merchant:
  - `dunning_settling_attempts`: Max retry count (default 20)
  - `failed_payment_cancelled_days`: Days before cancellation (default 20)
- Automatic daily retries via cron
- Status transitions: `active` → `past_due` → `error` → `expired`

---

## 🔄 Business Flows

### Subscription Signup Flow
1. `POST /api/v1/customer/` - Create customer
2. `POST /api/v1/straumur/create-session-add-card/` - Add payment method
3. `POST /api/v1/subscription/` - Create subscription
4. `POST /api/v1/subscription/{id}/update_cart/` - Add items
5. System creates first delivery and charges

### Recurring Billing Flow (Automated)
1. **Cron Job** runs daily (`payment_for_delivery` command)
2. Finds deliveries due for billing
3. Creates order with subscription items
4. Creates payment object
5. Charges payment method
6. Updates subscription status based on result
7. Sends notifications via event system

### Failed Payment Flow (Automated)
1. Payment fails → Status: `FAILED`
2. Subscription → Status: `PAST_DUE`
3. Next day: Retry payment (attempt 2)
4. Continue daily retries up to `dunning_settling_attempts`
5. After max attempts: Subscription → `ERROR`
6. After `failed_payment_cancelled_days`: Subscription → `EXPIRED`, Delivery → `cancelled`

---

## 🛠️ Admin Capabilities

### Available in Django Admin (Not API)
- Export to CSV/Excel (customers, subscriptions, orders, payments, deliveries)
- Bulk refund orders
- Manual charge button
- Recalculate stock levels
- Acknowledge stock alerts
- View audit logs
- Manage event templates
- Configure merchant settings
- Manage product categories/tags/attributes

### Needs API Exposure
All of the above should eventually have API equivalents for insurance companies and SaaS businesses to build their own admin tools.

---

## 📝 Documentation Notes

### For Mintlify Implementation:

1. **Current Endpoints:** Document as fully available with complete examples
2. **Proposed Endpoints:** Mark with `⚠️ PROPOSED` badge and "Coming Soon" notice
3. **Priority Indicators:**
   - 🔥 Critical (Phase 1)
   - 🔴 High (Phase 2)
   - 🟡 Medium (Phase 3)
   - 🟢 Low (Phase 4)

4. **Code Examples:** Provide for all current endpoints in:
   - cURL
   - Python
   - JavaScript
   - PHP

5. **Use Case Guides:** Create separate guides for:
   - Insurance integration
   - Recurring billing setup
   - Dunning management
   - Customer portal building
   - Webhook integration

6. **Interactive Examples:** Use Mintlify's API playground for all current endpoints

---

**End of Architecture Document**
