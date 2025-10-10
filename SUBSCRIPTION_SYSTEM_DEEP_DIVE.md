# Journey Subscription System - Deep Dive

**Version:** 1.0
**Last Updated:** October 9, 2025
**Purpose:** Comprehensive documentation of Journey's subscription engine and billing logic

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Core Data Models](#core-data-models)
3. [Subscription Lifecycle](#subscription-lifecycle)
4. [Frequency & Recipe System](#frequency--recipe-system)
5. [Delivery Synchronization Algorithm](#delivery-synchronization-algorithm)
6. [Payment States & Transitions](#payment-states--transitions)
7. [Automated Billing Flow](#automated-billing-flow)
8. [Dunning Management](#dunning-management)
9. [Edge Cases & Special Scenarios](#edge-cases--special-scenarios)
10. [API Integration Guide](#api-integration-guide)

---

## 🎯 Overview

Journey's subscription system is designed as a **recipe-based recurring billing engine** where:
- **Subscriptions** are containers holding a customer and configuration
- **Subscription Order Items** are the "recipe" - what products to deliver and how often
- **Orders** are created from the recipe on each billing cycle
- **Deliveries** represent when the order will be delivered (and charged)
- **Payments** track the financial transaction for each order

### Key Innovation: Multi-Frequency Subscriptions

Unlike traditional subscription systems where all items share one frequency, Journey supports:
- **Different frequencies per item** (e.g., milk weekly, eggs bi-weekly)
- **Automatic delivery synchronization** to minimize shipping costs
- **Smart merging algorithm** that combines items due within 5 days

---

## 📊 Core Data Models

### 1. Subscription

**File:** `apps/django/apps/order/models.py:450`

```python
class subscription(models.Model):
    id = models.AutoField(primary_key=True)
    customer_id = ForeignKey(customer)           # Who pays
    customer_receiver_id = ForeignKey(customer)  # Who receives (for gifts)
    subscription_status = CharField(choices=SubscriptionStatusChoices)
    delivery_option_id = ForeignKey(delivery_option)  # Default shipping method
    delivery_option_location_json = JSONField()       # Dropp location, etc.
    delivery_custom_data = JSONField()                # Custom delivery fields
    payment_processor_id = ForeignKey(payment_processor)
    created = DateTimeField(auto_now_add=True)
```

**Key Properties:**

```python
@property
def last_delivery_date(self):
    """Returns the most recent delivery date (past)"""
    return delivery.objects.filter(
        subscription_id=self,
        cancelled=False,
        order_id__isnull=False
    ).latest("delivery_date").delivery_date

@property
def next_delivery_date(self):
    """Calculates when the next delivery should occur"""
    last_delivery_date = self.last_delivery_date or timezone.now()
    day_after_last = last_delivery_date + timedelta(days=1)

    # Check if we have pre-generated orders fulfilling subscription
    last_order_fulfilled = self.order_set.latest(
        "fullfills_subscription_until"
    ).fullfills_subscription_until

    return max(last_order_fulfilled, day_after_last)
```

### 2. Subscription Order Item (The Recipe)

**File:** `apps/django/apps/order/models.py:268`

```python
class subscription_order_item(models.Model):
    id = models.AutoField(primary_key=True)
    subscription_id = ForeignKey(subscription, related_name="order_items")
    product_variation_id = ForeignKey(product_variation)
    subscription_frequency_id = ForeignKey(subscription_frequency)
    quantity = PositiveIntegerField(default=1)

    def to_order_item(self, order):
        """Converts recipe item to actual order item"""
        return order_item(
            product_variation_id=self.product_variation_id,
            quantity=self.quantity,
            order=order,
        )

    @property
    def frequency(self) -> relativedelta:
        """Returns timedelta for this item's frequency"""
        return self.subscription_frequency_id.frequency_timedelta
```

**Example Subscription with Multiple Frequencies:**

```json
{
  "subscription_id": 789,
  "order_items": [
    {
      "id": 101,
      "product_variation_id": 5,  // Milk
      "quantity": 2,
      "subscription_frequency_id": 1  // Weekly (7 days)
    },
    {
      "id": 102,
      "product_variation_id": 8,  // Eggs
      "quantity": 1,
      "subscription_frequency_id": 2  // Bi-weekly (14 days)
    },
    {
      "id": 103,
      "product_variation_id": 12, // Coffee
      "quantity": 1,
      "subscription_frequency_id": 3  // Monthly (30 days)
    }
  ]
}
```

### 3. Subscription Frequency

**File:** `apps/django/apps/merchant/models.py:304`

```python
class subscription_frequency(models.Model):
    description = CharField(max_length=200)           # "Every 7 days"
    long_description = CharField(max_length=200)      # "Every week"
    frequency_name = CharField(choices=FREQUENCY_CHOICES)  # days/weeks/months
    frequency = IntegerField()                        # The number (7, 14, 30)

    @property
    def frequency_timedelta(self):
        """Returns relativedelta for date calculations"""
        if self.frequency_name == 'days':
            return relativedelta(days=self.frequency)
        elif self.frequency_name == 'weeks':
            return relativedelta(weeks=self.frequency)
        elif self.frequency_name == 'months':
            return relativedelta(months=self.frequency)
```

**Common Frequencies:**

| ID | Description | Frequency Name | Frequency | Usage |
|----|-------------|---------------|-----------|-------|
| 1 | Every 7 days | days | 7 | Weekly delivery |
| 2 | Every 14 days | days | 14 | Bi-weekly |
| 3 | Every 30 days | months | 1 | Monthly |
| 4 | Every 60 days | days | 60 | Bi-monthly |
| 5 | Every 90 days | months | 3 | Quarterly |

### 4. Order (Generated from Recipe)

**File:** `apps/django/apps/order/models.py:319`

```python
class order(models.Model):
    id = models.AutoField(primary_key=True)
    customer_id = ForeignKey(customer)
    subscription_id = ForeignKey(subscription)  # Links back to recipe
    payment_id = ForeignKey(payment)
    date_time = DateTimeField()                 # When order created
    fullfills_subscription_until = DateTimeField()  # Fulfillment date
    subscription_data = JSONField()             # Snapshot of recipe used
```

### 5. Delivery (Charging Event)

**File:** `apps/django/apps/delivery/models.py`

```python
class delivery(models.Model):
    id = models.AutoField(primary_key=True)
    order_id = ForeignKey(order)
    subscription_id = ForeignKey(subscription)
    delivery_date = DateTimeField()  # When it will be delivered AND charged
    delivered = BooleanField(default=False)
    packed = BooleanField(default=False)
    cancelled = BooleanField(default=False)
```

**Key Concept:** `delivery_date` serves dual purpose:
1. Physical delivery date (when item arrives)
2. **Charging date** (when payment is attempted)

### 6. Payment

**File:** `apps/django/apps/order/models.py`

```python
class payment(models.Model):
    id = models.AutoField(primary_key=True)
    order_id = ForeignKey(order)
    payment_method_id = ForeignKey(payment_method)
    payment_status = CharField(choices=PaymentStatus.choices)
    line_items_json = JSONField()  # Invoice line items
    created = DateTimeField()
    last_attempt = DateTimeField()
    settling_attempts = IntegerField(default=0)  # Retry counter
```

---

## 🔄 Subscription Lifecycle

### Subscription States

**File:** `apps/django/apps/order/models.py:422`

```python
class SubscriptionStatusChoices(models.TextChoices):
    INCOMPLETE = "incomplete"  # Payment info missing
    ACTIVE = "active"          # Billing successfully
    PAST_DUE = "past_due"      # Payment failed, retrying
    ON_HOLD = "on_hold"        # Customer paused
    EXPIRED = "expired"        # Dunning failed, cancelled by system
    CANCELLED = "cancelled"    # Customer cancelled
    ERROR = "error"            # Technical payment error
```

### State Transition Diagram

```
┌─────────────┐
│ INCOMPLETE  │ Customer signs up
└──────┬──────┘
       │ First payment succeeds
       ↓
┌─────────────┐     Payment fails      ┌─────────────┐
│   ACTIVE    │ ───────────────────→   │  PAST_DUE   │
└──────┬──────┘                        └──────┬──────┘
       │                                      │
       │ Customer pauses                      │ Retry fails (tech error)
       ↓                                      ↓
┌─────────────┐                        ┌─────────────┐
│  ON_HOLD    │                        │    ERROR    │
└──────┬──────┘                        └──────┬──────┘
       │                                      │
       │ Customer resumes                     │ Max retries reached
       │                                      │ (20 days default)
       │                                      ↓
       │                                ┌─────────────┐
       │                                │   EXPIRED   │
       │                                └─────────────┘
       │
       │ Customer cancels permanently
       ↓
┌─────────────┐
│  CANCELLED  │
└─────────────┘
```

### State Transitions Logic

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:520-540`

```python
def update_subscription_status(delivery_item, payment_object):
    subscription_object = delivery_item.subscription_id
    subscription_status_before = subscription_object.subscription_status

    if payment_object.payment_status == PaymentStatus.FAILED:
        # Determine if it's past_due (retry) or error (technical)
        if past_due:  # Based on error code
            subscription_object.subscription_status = SubscriptionStatusChoices.PAST_DUE
        else:
            subscription_object.subscription_status = SubscriptionStatusChoices.ERROR

    elif payment_object.payment_status == PaymentStatus.SETTLED:
        subscription_object.subscription_status = SubscriptionStatusChoices.ACTIVE

    subscription_object.save()

    # Trigger events on status change
    if has_changed:
        if subscription_object.subscription_status == SubscriptionStatusChoices.PAST_DUE:
            event_handler(EventHandles.SUBSCRIPTION_STATUS_SET_TO_PAST_DUE_FOR_THE_FIRST_TIME)
```

---

## 🍳 Frequency & Recipe System

### How the Recipe Works

1. **Subscription** = Container
2. **Subscription Order Items** = Recipe ingredients
3. **Each item** has its own frequency
4. **System generates orders** by applying the recipe

### Example: Insurance Subscription

```python
# Create subscription
subscription = subscription.objects.create(
    customer_id=customer,
    subscription_status="active",
    delivery_option_id=3
)

# Add recipe items with different frequencies
subscription_order_item.objects.create(
    subscription_id=subscription,
    product_variation_id=premium_car,  # Car insurance
    quantity=1,
    subscription_frequency_id=monthly  # Billed monthly
)

subscription_order_item.objects.create(
    subscription_id=subscription,
    product_variation_id=premium_home,  # Home insurance
    quantity=1,
    subscription_frequency_id=quarterly  # Billed quarterly
)
```

### Recipe to Order Conversion

**File:** `apps/django/apps/delivery/delivery_date_logic_functions.py:131-161`

```python
def submit_preliminary_order(preliminary_order: PreliminaryOrder):
    """Converts recipe to actual order"""
    subscription_obj = preliminary_order.subscription_object

    # Create order from recipe
    new_order = order.objects.create(
        subscription_id=subscription_obj,
        customer_id=subscription_obj.customer_id,
        date_time=preliminary_order.delivery_date,
        fullfills_subscription_until=preliminary_order.subscription_fullfilled_until,
        subscription_data=preliminary_order.serialize()  # Snapshot of recipe
    )

    # Convert each recipe item to order item
    for item in preliminary_order.items:
        new_order_item = item.subscription_order_item.to_order_item(order=new_order)
        new_order_item.save()

    # Create delivery (charging event)
    new_delivery = delivery.objects.create(
        subscription_id=subscription_obj,
        order_id=new_order,
        delivery_date=preliminary_order.delivery_date
    )

    return new_order, new_delivery
```

---

## 🔁 Delivery Synchronization Algorithm

### The Problem

With multi-frequency subscriptions, you could end up with many small deliveries:
- **Monday:** Milk only (weekly)
- **Tuesday:** Eggs only (bi-weekly)
- **Friday:** Coffee only (monthly)

**Problem:** Inefficient shipping, high costs, poor customer experience.

### The Solution: Join-by-Week Algorithm

**File:** `apps/django/apps/delivery/shop_logic_functions.py:167-240`

The algorithm merges deliveries that are **within 5 days** of each other.

#### Algorithm Steps:

**1. Calculate Next Delivery Date for Each Item**

```python
# For each subscription_order_item:
last_order_date = get_last_order_date_for_product(item.product_variation_id)
next_delivery_date = last_order_date + item.frequency
```

**Example:**
- Milk last delivered: Oct 1 → Next: Oct 8 (7 days)
- Eggs last delivered: Oct 1 → Next: Oct 15 (14 days)
- Coffee last delivered: Sep 1 → Next: Oct 1 (30 days)

**2. Find Earliest Delivery Date**

```python
earliest_date = min([milk_next, eggs_next, coffee_next])  # Oct 1 (Coffee)
```

**3. Check Which Items Are Due Within 5 Days**

```python
num_joinable_days = 5

items_to_include = []
for item in subscription_items:
    if (item.next_date - earliest_date).days < num_joinable_days:
        items_to_include.append(item)
```

**Example:** On Oct 1:
- Coffee: 0 days away → **Include**
- Milk: 7 days away → **Don't include** (outside 5-day window)
- Eggs: 14 days away → **Don't include**

**Result:** Deliver only Coffee on Oct 1

**4. Repeat for Next Order**

On Oct 8:
- Milk: 0 days away → **Include**
- Eggs: 7 days away → **Don't include** (but close!)
- Coffee: Already fulfilled

On Oct 15:
- Eggs: 0 days away → **Include**
- Milk: Already fulfilled

### Algorithm Code

**File:** `apps/django/apps/delivery/shop_logic_functions.py:167`

```python
def run_algorithm_join_by_week_generator(
    subscription_items: List[VirtualSubscriptionItem],
    previous_orders: List[VirtualOrder],
    start_date: datetime.datetime,
    num_joinable_days: int = 5,
    weekdays: List[int] = None,  # Allowed delivery weekdays
):
    """
    Generates orders by joining items that are due within num_joinable_days
    """
    orders_to_be_joined = []
    first_order = None

    # Generate next order
    generator = run_algorithm_generator(subscription_items, previous_orders, start_date)

    for order in generator:
        if first_order is None:
            first_order = order
            orders_to_be_joined.append(order)
            continue

        # Check if within joinable window
        days_apart = (order.delivery_date - first_order.delivery_date).days

        if days_apart < num_joinable_days:
            orders_to_be_joined.append(order)  # Merge it
        else:
            # Too far apart - ship first batch
            merged_order = merge_orders(orders_to_be_joined)

            # Align with allowed weekdays
            if weekdays:
                merged_order.delivery_date = align_with_weekdays(
                    merged_order.delivery_date,
                    weekdays
                )

            yield merged_order

            # Reset for next batch
            first_order = order
            orders_to_be_joined = [order]
```

### Weekday Alignment

**File:** `apps/django/apps/delivery/shop_logic_functions.py:266`

Deliveries are aligned to postal code delivery schedules:

```python
def align_with_multiple_weekdays_and_cutoff(
    date: datetime.datetime,
    weekdays: List[int],  # [0, 2, 4] = Mon, Wed, Fri
    cutoff_days: int = 3  # Packing time
):
    """
    Aligns delivery date with allowed weekdays, accounting for packing time
    """
    date_with_cutoff = date + timedelta(days=cutoff_days)
    current_weekday = date_with_cutoff.weekday()

    # Find next allowed weekday
    for weekday in sorted(weekdays):
        if current_weekday <= weekday:
            days_to_add = weekday - current_weekday
            return date_with_cutoff + timedelta(days=days_to_add)

    # Wrap to next week
    next_week_day = min(weekdays)
    days_to_add = 7 - current_weekday + next_week_day
    return date_with_cutoff + timedelta(days=days_to_add)
```

### Example: Postal Code 101 (Reykjavík)

```python
postal_code_schedule = {
    "postal_code": "101",
    "weekday_number_subscription": [2, 4],  # Wednesday, Friday
    "weekday_number_first_delivery": [1, 3, 5]  # Tue, Thu, Sat for first orders
}
```

If next delivery calculates to **Monday**, it's shifted to **Wednesday** (next allowed weekday).

---

## 💳 Payment States & Transitions

### Payment Status Enum

**File:** `apps/django/apps/order/models.py`

```python
class PaymentStatus(models.TextChoices):
    NEW_CHARGE = "new_charge"      # Created, not yet attempted
    AUTHORIZED = "authorized"       # Card authorized (pre-auth)
    SETTLED = "settled"             # Money captured ✅
    FAILED = "failed"               # Payment declined ❌
    CANCELLED = "cancelled"         # Order cancelled
    REFUNDED = "refunded"           # Money returned
    UNCAPTURABLE = "uncapturable"   # Authorization expired
```

### Payment Lifecycle

```
┌──────────────┐
│  NEW_CHARGE  │  Delivery date arrives
└──────┬───────┘
       │
       │ Charge attempt
       ↓
    ┌──────┐
    │ Try  │
    └──┬───┘
       │
       ├─→ Success ─→ ┌────────────┐
       │              │  SETTLED   │ ✅ Money received
       │              └────────────┘
       │
       ├─→ Fail ────→ ┌────────────┐
       │              │   FAILED   │ ❌ Payment declined
       │              └──────┬─────┘
       │                     │
       │                     │ Retry tomorrow
       │                     ↓
       │              ┌────────────┐
       │              │ Retry (1)  │
       │              └──────┬─────┘
       │                     │
       │                     ├─→ Success ─→ SETTLED ✅
       │                     │
       │                     ├─→ Fail ────→ Retry (2)
       │                     │
       │                     │ ... up to 20 attempts
       │                     │
       │                     ↓
       │              ┌────────────┐
       │              │  After 20  │
       │              │   days     │
       │              └──────┬─────┘
       │                     │
       │                     ↓
       │              ┌────────────┐
       │              │ CANCELLED  │ Subscription EXPIRED
       │              └────────────┘
       │
       └─→ Customer cancels before charge
                      ┌────────────┐
                      │ CANCELLED  │
                      └────────────┘
```

### Payment Attempt Logic

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:438-520`

```python
def charge_for_delivery(delivery_item: delivery):
    """Main payment charging function"""
    payment_object = delivery_item.order_id.payment_id
    subscription_object = delivery_item.subscription_id
    customer_object = subscription_object.customer_id

    # Get payment method (card)
    payment_method_object = customer_object.primary_payment_method

    # Create invoice line items from order
    cart_entries = line_items_for_order(delivery_item.order_id)
    line_items_object = LineItem(cart_entries)

    # Store line items in payment
    payment_object.line_items = line_items_object.display()
    payment_object.line_items_json = line_items_object.serialize()

    # Get payment processor
    processor = payment_object.payment_processor_id.name  # "Straumur", "Reepay", etc.

    # Attempt charge based on processor
    if processor == "Straumur":
        response = straumur.pay_with_token(
            amount=line_items_object.total(),
            currency="ISK",
            reference=f"order-{order_object.id}",
            token_value=payment_method_object.virtual_card_token,
            terminal_identifier=config.terminal_identifier_gateway,
        )
    elif processor == "Reepay":
        response = reepay.charge(...)
    elif processor == "Rapyd":
        response = rapyd.charge(...)

    # Update payment status based on response
    if response.get('success'):
        payment_object.payment_status = PaymentStatus.SETTLED
        past_due = False
    else:
        payment_object.payment_status = PaymentStatus.FAILED
        error_code = response.get('errorCode', '')

        # Determine if retryable (past_due) vs technical error
        past_due = error_code in ['insufficient_funds', 'card_declined']

    # Update counters
    payment_object.settling_attempts += 1
    payment_object.last_attempt = timezone.now()
    payment_object.response = response
    payment_object.save()

    # Update subscription status
    update_subscription_status(delivery_item, payment_object, past_due)
```

### Retry Schedule Logic

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:149`

```python
def check_payment_attempt_already_happened_today(payment_object):
    """Prevents multiple retries per day"""
    is_before_8am = timezone.now().hour < 8

    if payment_object.last_attempt is None:
        return False

    last_attempt_date = payment_object.last_attempt.date()
    today = timezone.now().date()

    # Allow retry if:
    # - Last attempt was yesterday OR
    # - It's before 8am (for early morning retries)
    if last_attempt_date == today and not is_before_8am:
        return True  # Already tried today

    return False  # OK to retry
```

**Retry Schedule:**
- **Day 1:** Initial charge attempt
- **Day 2:** Retry #1 (after midnight)
- **Day 3:** Retry #2
- **...**
- **Day 20:** Retry #19 (final attempt)
- **Day 21:** Cancel delivery, expire subscription

---

## 🤖 Automated Billing Flow

### Daily Cron Job

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:83`

```bash
# Runs daily at midnight
python manage.py payment_for_delivery
```

### Cron Job Flow

```python
class Command(BaseCommand):
    def handle(self, *args, **options):
        # Run for each tenant
        for tenant in Tenant.objects.all():
            with tenant_context(tenant):
                # 1. Process new deliveries
                process_deliveries_to_be_charged()

                # 2. Retry failed payments
                process_deliveries_with_past_due_subscription_status_and_failed_payment_status()

                # 3. Handle error subscriptions
                process_deliveries_with_error_subscription_status()

                # 4. Cancel old failed invoices
                process_failed_invoices()

                # 5. Auto-mark delivered (for auto_delivery products)
                process_deliveries_for_dropp()
```

### Step 1: Process New Deliveries

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:617`

```python
def process_deliveries_to_be_charged():
    """Charges deliveries whose delivery_date has arrived"""
    payment_ctx = PaymentContext(merchant_id=1)

    # Find deliveries due today
    deliveries_to_be_charged = delivery.objects.filter(
        delivery_date__range=payment_ctx.range,  # Today's date range
        delivered=False,
        cancelled=False,
    ).exclude(
        order_id__payment_id__payment_status=PaymentStatus.SETTLED
    )

    for delivery_item in deliveries_to_be_charged:
        # Create payment if doesn't exist
        if not delivery_item.order_id.payment_id:
            new_charge_for_delivery(delivery_item)
        else:
            # Attempt charge
            charge_for_delivery(delivery_item)
```

### Step 2: Retry Failed Payments

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:695`

```python
def process_deliveries_with_past_due_subscription_status_and_failed_payment_status():
    """Retries payments for past_due subscriptions"""

    _30_days_ago = timezone.now() - timedelta(days=30)
    _14_days_future = timezone.now() + timedelta(days=14)

    # Find failed payments within retry window
    deliveries_to_retry = delivery.objects.filter(
        subscription_id__subscription_status__in=[
            SubscriptionStatusChoices.ACTIVE,
            SubscriptionStatusChoices.PAST_DUE
        ],
        order_id__payment_id__payment_status=PaymentStatus.FAILED,
        delivery_date__date__range=(_30_days_ago, _14_days_future),
        cancelled=False
    )

    for delivery_item in deliveries_to_retry:
        payment_object = delivery_item.payment_id

        # Check if already retried today
        if check_payment_attempt_already_happened_today(payment_object):
            continue

        # Determine action based on delivery status
        if delivery_item.delivered:
            # Already delivered - must charge
            charge_for_delivery(delivery_item)
        else:
            # Not delivered - can reschedule
            reschedule_delivery(delivery_item)
```

### Step 3: Cancel Old Failed Invoices

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:845`

```python
def process_failed_invoices():
    """Cancels deliveries/payments that have failed for too long"""
    merchant_obj = merchant.objects.get(id=1)
    max_failed_days = merchant_obj.failed_payment_cancelled_days  # Default: 20

    cutoff_date = timezone.now() - timedelta(days=max_failed_days)

    # Find old failed payments
    old_failed_deliveries = delivery.objects.filter(
        order_id__payment_id__created__lte=cutoff_date.date(),
        delivered=False,
    ).exclude(
        order_id__payment_id__payment_status__in=["authorized", "settled"]
    )

    for delivery_item in old_failed_deliveries:
        # Cancel delivery
        delivery_item.cancelled = True
        delivery_item.save()

        # Cancel payment
        delivery_item.payment_id.payment_status = PaymentStatus.CANCELLED
        delivery_item.payment_id.save()

        # Unreserve stock
        unreserve_stock_for_order(delivery_item.order_id)

        # Update subscription to EXPIRED
        delivery_item.subscription_id.subscription_status = SubscriptionStatusChoices.EXPIRED
        delivery_item.subscription_id.save()
```

---

## 🚨 Dunning Management

### Configuration (Per Merchant)

**File:** `apps/django/apps/merchant/models.py:63`

```python
class merchant(models.Model):
    dunning_settling_attempts = IntegerField(
        default=20,
        help_text="Max retry attempts for failed invoice (1 per day)"
    )

    failed_payment_cancelled_days = IntegerField(
        default=20,
        help_text="Days after failed payment to cancel subscription"
    )
```

### Dunning Workflow

**Day 1: Payment Fails**
```
Subscription: active → past_due
Payment: settled → failed
settling_attempts: 0 → 1
Event: SUBSCRIPTION_STATUS_SET_TO_PAST_DUE_FOR_THE_FIRST_TIME
Email: "Your payment failed, will retry tomorrow"
```

**Days 2-19: Daily Retries**
```
Each day at midnight:
- Check: settling_attempts < dunning_settling_attempts?
- If yes: Retry payment
  - Success: past_due → active ✅
  - Fail: settling_attempts += 1
- If settling_attempts >= 20: Stop retrying
```

**Day 20: Final Attempt**
```
settling_attempts: 19 → 20
If still fails:
- Payment: failed (stays)
- Subscription: past_due → error
- Event: SUBSCRIPTION_STATUS_SET_TO_ERROR_FOR_THE_FIRST_TIME
- Email: "Final attempt failed"
```

**Day 21+: Grace Period**
```
- Subscription remains in ERROR state
- No more retries
- Waiting for cancellation window
```

**Day 21 (if failed_payment_cancelled_days=20):**
```
- Payment: failed → cancelled
- Delivery: cancelled = True
- Subscription: error → expired
- Event: SUBSCRIPTION_STATUS_SET_TO_EXPIRED
- Email: "Subscription cancelled due to payment failure"
```

### Error Code Handling

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:431`

```python
# Determine if retryable based on error code
error_code = invoice_response.get('errorCode', '')

if error_code in ['insufficient_funds', 'card_declined', 'do_not_honor']:
    past_due = True  # Retryable - card might have funds later
else:
    past_due = False  # Technical error - needs manual intervention
```

**State Transitions Based on Error Type:**

| Error Type | Error Code Examples | Subscription State | Retry? |
|------------|---------------------|-------------------|---------|
| **Insufficient Funds** | `insufficient_funds`, `51` | PAST_DUE | Yes ✅ |
| **Card Declined** | `card_declined`, `05` | PAST_DUE | Yes ✅ |
| **Expired Card** | `expired_card`, `54` | ERROR | No ❌ |
| **Invalid Card** | `invalid_card_number` | ERROR | No ❌ |
| **Fraud Detected** | `fraud_detected` | ERROR | No ❌ |
| **Technical Error** | `gateway_timeout` | ERROR | No ❌ |

---

## 🎯 Edge Cases & Special Scenarios

### 1. Customer Updates Payment Method During Dunning

**Scenario:** Payment fails, customer adds new card during retry period.

**Flow:**
```python
# Customer adds new card
POST /api/v1/straumur/create-session-add-card/
# → Creates new payment_method with status="active"

# Next day cron runs
def charge_for_delivery(delivery_item):
    # Gets customer's primary_payment_method
    customer_object.primary_payment_method
    # → Returns NEWEST active payment method

    # Retries with new card
    response = straumur.pay_with_token(token=new_card.virtual_card_token)

    # If successful:
    subscription.subscription_status = "active"  # past_due → active ✅
```

### 2. Subscription with Already-Delivered Unpaid Delivery

**Scenario:** Delivery was marked as delivered but payment failed.

**Rule:** Must attempt payment regardless of retry limit (customer received goods).

**File:** `apps/django/apps/order/management/commands/payment_for_delivery.py:723`

```python
if delivery_item.delivered == True:
    # Force charge even if retry limit reached
    charge_for_delivery(delivery_item)
else:
    # Can reschedule if not yet delivered
    reschedule_delivery(delivery_item)
```

### 3. Multi-Frequency Sync After Resume

**Scenario:** Customer pauses subscription for 2 months, then resumes.

**File:** `apps/django/apps/order/models.py:605`

```python
def sync_with_schedule(self, action: Literal["resume", None] = None):
    """Regenerate orders after pause/changes"""
    new_prelim_orders = generate_orders_until_fulfilled(self, action=action)

    # If resuming, start from today
    if action == 'resume':
        min_start_date = timezone.now()

    # Generate next 1-4 orders to fulfill all items
    for preliminary_order in new_prelim_orders:
        submit_preliminary_order(preliminary_order)
```

**Result:** All items sync to next available delivery date after today.

### 4. Frequency Change Mid-Cycle

**Scenario:** Customer changes milk from weekly to bi-weekly.

**Current Behavior:**
```python
# Update subscription_order_item
subscription_order_item.objects.filter(id=101).update(
    subscription_frequency_id=2  # 14 days instead of 7
)

# Next order generation uses new frequency
next_delivery = last_delivery_date + new_frequency  # 14 days
```

**Note:** Existing scheduled deliveries are NOT updated. Only future orders use new frequency.

### 5. Product Out of Stock

**Scenario:** Delivery scheduled but product sold out.

**File:** `apps/django/apps/merchant/models.py:389`

```python
def can_fulfill_quantity(self, quantity):
    """Check stock availability"""
    if not self.stock_item:
        return True  # No tracking = unlimited
    return self.stock_item.can_fulfill_quantity(quantity)
```

**Current:** No automatic handling. Would need to:
1. Check stock before creating order
2. Notify customer
3. Skip item or reschedule

**Proposed Enhancement:**
```python
# Before creating order
for item in subscription.order_items.all():
    if not item.product_variation_id.can_fulfill_quantity(item.quantity):
        # Option A: Skip this item
        # Option B: Delay entire order
        # Option C: Substitute product
        pass
```

---

## 🔌 API Integration Guide

### Creating a Subscription

**Step 1: Create Customer**
```bash
POST /api/v1/customer/
{
  "full_name": "Jón Jónsson",
  "email": "jon@example.is",
  "phone_number": "7771234",
  "postal_code": "101"
}
# Response: { "id": 42, "customer_handle": "cust-journey-42" }
```

**Step 2: Add Payment Method**
```bash
POST /api/v1/straumur/create-session-add-card/
{
  "customer_id": 42,
  "return_url": "https://insurance.com/success"
}
# Response: { "checkoutUrl": "https://checkout.straumur.is/...", "uuid": "..." }
```

**Step 3: Create Subscription**
```bash
POST /api/v1/subscription/
{
  "customer_id": 42,
  "subscription_status": "active",
  "delivery_option_id": 3,
  "start_date": "2025-11-01"
}
# Response: { "id": 789 }
```

**Step 4: Add Subscription Items (Recipe)**
```bash
POST /api/v1/subscription/789/update_cart/
{
  "order_items": [
    {
      "product_variation_id": 5,  // Car insurance premium
      "quantity": 1,
      "subscription_frequency_id": 3  // Monthly
    },
    {
      "product_variation_id": 8,  // Home insurance add-on
      "quantity": 1,
      "subscription_frequency_id": 3  // Monthly
    }
  ]
}
```

**Step 5: System Auto-Generates Orders**
```python
# Runs automatically via cron
subscription.sync_with_schedule()
# → Creates first order with delivery_date = Nov 1, 2025
# → Creates delivery object
# → On Nov 1, charges customer automatically
```

### Monitoring Subscription Status

```bash
# Check subscription
GET /api/v1/subscription/789/
# Response:
{
  "id": 789,
  "subscription_status": "active",
  "order_items": [...],
  "next_delivery_date": "2025-12-01"
}

# Check payment history (🔮 PROPOSED)
GET /api/v1/subscription/789/payments/
# Response:
{
  "payments": [
    {
      "id": 456,
      "date": "2025-11-01",
      "amount": 15000,
      "status": "settled"
    }
  ]
}
```

### Handling Failed Payments

**Webhook (🔮 PROPOSED):**
```bash
POST https://insurance.com/webhooks
{
  "event": "subscription.past_due",
  "data": {
    "subscription_id": 789,
    "payment_id": 457,
    "failure_reason": "insufficient_funds",
    "next_retry": "2025-11-02T00:00:00Z"
  }
}
```

**Manual Retry (🔮 PROPOSED):**
```bash
POST /api/v1/payment/457/retry/
{
  "force": true  # Retry now, not tomorrow
}
```

---

## 📊 Key Metrics

### Subscription Health Indicators

```python
# Calculate for dashboard
active_subscriptions = subscription.objects.filter(
    subscription_status="active"
).count()

past_due_subscriptions = subscription.objects.filter(
    subscription_status="past_due"
).count()

# Churn rate
expired_this_month = subscription.objects.filter(
    subscription_status="expired",
    updated__gte=timezone.now() - timedelta(days=30)
).count()

churn_rate = expired_this_month / (active_subscriptions + expired_this_month)
```

### Payment Success Rate

```python
# Last 30 days
payments_attempted = payment.objects.filter(
    created__gte=timezone.now() - timedelta(days=30)
).count()

payments_settled = payment.objects.filter(
    created__gte=timezone.now() - timedelta(days=30),
    payment_status=PaymentStatus.SETTLED
).count()

success_rate = payments_settled / payments_attempted
```

---

## 🎓 Summary

### Key Concepts

1. **Subscription** = Container for customer and configuration
2. **Subscription Order Items** = Recipe (what to deliver, how often)
3. **Multi-Frequency** = Each item can have different billing frequency
4. **Synchronization** = Items due within 5 days are merged
5. **Delivery** = Physical delivery AND charging event (dual purpose)
6. **Automated Billing** = Cron job runs daily at midnight
7. **Dunning** = 20 daily retries, then cancel after 20 days
8. **State Machine** = Clear subscription states with defined transitions

### For Documentation

When documenting for Mintlify:
- Explain subscription as "recipe" metaphor
- Emphasize multi-frequency capability (unique selling point)
- Diagram the state machine
- Show code examples for subscription creation
- Explain synchronization algorithm with visuals
- Document dunning flow day-by-day

---

**End of Subscription System Deep Dive**
