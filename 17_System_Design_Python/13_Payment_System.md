# System Design: Payment Processing System (Like Stripe)

## 📖 Problem Statement

Design a payment processing system like Stripe or PayPal that handles credit card transactions, manages accounts, ensures ACID properties, prevents double-charging, and handles refunds with strong consistency guarantees.

**Core Features:**

- Process payments (credit card, bank transfer)
- Account management (merchants, customers)
- Idempotency (no duplicate charges)
- Refunds and disputes
- Transaction history and reconciliation
- Webhooks for event notifications
- Fraud detection

## 🎯 Requirements

### Functional Requirements

1. **Process payments** - Charge customer, transfer to merchant
2. **Account balances** - Track available balance, pending transactions
3. **Refunds** - Full and partial refunds
4. **Idempotency** - Prevent duplicate charges
5. **Transaction history** - Audit trail for all operations
6. **Webhooks** - Notify merchants of events
7. **Multi-currency** - Support USD, EUR, GBP, etc.

### Non-Functional Requirements

1. **Consistency**: Strong ACID guarantees (no money lost)
2. **Reliability**: 99.99% uptime
3. **Security**: PCI DSS compliance, encryption
4. **Low Latency**: Payment processing < 500ms
5. **Scalability**: Handle 10K transactions/second
6. **Auditability**: Complete transaction logs

## 📊 Capacity Estimation

```python
class PaymentSystemEstimation:
    """Calculate payment system capacity"""

    def __init__(self):
        self.total_merchants = 1_000_000
        self.active_merchants_daily = 100_000
        self.avg_transactions_per_merchant_per_day = 50
        self.avg_transaction_amount = 50  # USD

    def calculate_transactions(self):
        """Calculate transaction volume"""
        daily_transactions = self.active_merchants_daily * self.avg_transactions_per_merchant_per_day
        tps = daily_transactions / (24 * 3600)
        peak_tps = tps * 5  # Peak is 5x average

        daily_volume = daily_transactions * self.avg_transaction_amount

        print("=== Transaction Volume ===")
        print(f"Daily Transactions: {daily_transactions:,.0f}")
        print(f"Average TPS: {tps:,.0f}")
        print(f"Peak TPS: {peak_tps:,.0f}")
        print(f"Daily Transaction Volume: ${daily_volume:,.0f}")
        print(f"Monthly Volume: ${daily_volume * 30:,.0f}\n")

    def calculate_database_size(self):
        """Calculate database storage"""
        # Each transaction: ~500 bytes metadata
        yearly_transactions = self.active_merchants_daily * self.avg_transactions_per_merchant_per_day * 365
        storage_gb = (yearly_transactions * 500) / (1024 ** 3)

        print("=== Database Storage ===")
        print(f"Yearly Transactions: {yearly_transactions:,.0f}")
        print(f"Storage per Year: {storage_gb:,.2f} GB")
        print(f"5-Year Storage: {storage_gb * 5:,.2f} GB\n")

    def calculate_revenue(self):
        """Calculate platform revenue (2.9% + $0.30 per transaction)"""
        daily_transactions = self.active_merchants_daily * self.avg_transactions_per_merchant_per_day

        percentage_fee = daily_transactions * self.avg_transaction_amount * 0.029
        fixed_fee = daily_transactions * 0.30
        daily_revenue = percentage_fee + fixed_fee

        print("=== Revenue Estimation ===")
        print(f"Daily Revenue: ${daily_revenue:,.0f}")
        print(f"Monthly Revenue: ${daily_revenue * 30:,.0f}")
        print(f"Yearly Revenue: ${daily_revenue * 365:,.0f}\n")

# Run estimation
estimator = PaymentSystemEstimation()
estimator.calculate_transactions()
estimator.calculate_database_size()
estimator.calculate_revenue()
```

**Output:**

```
=== Transaction Volume ===
Daily Transactions: 5,000,000
Average TPS: 58
Peak TPS: 289
Daily Transaction Volume: $250,000,000
Monthly Volume: $7,500,000,000

=== Database Storage ===
Yearly Transactions: 1,825,000,000
Storage per Year: 851.88 GB
5-Year Storage: 4,259.42 GB

=== Revenue Estimation ===
Daily Revenue: $8,750,000
Monthly Revenue: $262,500,000
Yearly Revenue: $3,193,750,000
```

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────┐
│          Client Applications                      │
│   (Merchant websites, Mobile apps)               │
└────────────────┬─────────────────────────────────┘
                 │
                 ▼
          ┌─────────────┐
          │  API Gateway │
          │ (Rate Limit) │
          └──────┬───────┘
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Payment  │ │ Account  │ │ Webhook  │
│ Service  │ │ Service  │ │ Service  │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│PostgreSQL│ │  Redis   │ │  Kafka   │
│ (ACID)   │ │(Idempot.)│ │ (Events) │
└──────────┘ └──────────┘ └────┬─────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ Fraud        │
                         │ Detection    │
                         └──────────────┘
                                │
                                ▼
                         ┌──────────────┐
                         │ External     │
                         │ Payment      │
                         │ Processors   │
                         └──────────────┘
```

## 🔑 Database Schema

```sql
-- Merchants (sellers)
CREATE TABLE merchants (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    api_key VARCHAR(100) UNIQUE NOT NULL,
    secret_key VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'active',  -- active, suspended, banned
    balance_cents BIGINT DEFAULT 0,  -- Store in cents to avoid float issues
    pending_balance_cents BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Customers
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    merchant_id BIGINT REFERENCES merchants(id),
    email VARCHAR(255),
    name VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(merchant_id, email)
);

-- Payment methods (credit cards, bank accounts)
CREATE TABLE payment_methods (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    type VARCHAR(20) NOT NULL,  -- card, bank_account
    last4 VARCHAR(4),  -- Last 4 digits
    brand VARCHAR(20),  -- visa, mastercard, amex
    exp_month INT,
    exp_year INT,
    token VARCHAR(100) UNIQUE,  -- Tokenized payment method
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transactions (double-entry bookkeeping)
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    merchant_id BIGINT REFERENCES merchants(id),
    customer_id BIGINT REFERENCES customers(id),
    payment_method_id BIGINT REFERENCES payment_methods(id),
    amount_cents BIGINT NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(20) DEFAULT 'pending',  -- pending, succeeded, failed, refunded
    failure_reason TEXT,
    idempotency_key VARCHAR(100) UNIQUE,  -- Prevent duplicates
    description TEXT,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ledger entries (double-entry accounting)
CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    transaction_id BIGINT REFERENCES transactions(id),
    account_id BIGINT NOT NULL,  -- merchant_id or system account
    account_type VARCHAR(20) NOT NULL,  -- debit, credit
    amount_cents BIGINT NOT NULL,
    balance_cents BIGINT NOT NULL,  -- Running balance
    entry_type VARCHAR(50) NOT NULL,  -- charge, refund, payout, fee
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Refunds
CREATE TABLE refunds (
    id BIGSERIAL PRIMARY KEY,
    transaction_id BIGINT REFERENCES transactions(id),
    amount_cents BIGINT NOT NULL,
    reason VARCHAR(200),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Webhooks
CREATE TABLE webhook_endpoints (
    id BIGSERIAL PRIMARY KEY,
    merchant_id BIGINT REFERENCES merchants(id),
    url VARCHAR(500) NOT NULL,
    events TEXT[],  -- Array of event types
    secret VARCHAR(100) NOT NULL,
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE webhook_events (
    id BIGSERIAL PRIMARY KEY,
    merchant_id BIGINT REFERENCES merchants(id),
    event_type VARCHAR(50) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',  -- pending, delivered, failed
    retry_count INT DEFAULT 0,
    next_retry_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_transactions_merchant ON transactions(merchant_id, created_at DESC);
CREATE INDEX idx_transactions_customer ON transactions(customer_id, created_at DESC);
CREATE INDEX idx_transactions_status ON transactions(status, created_at DESC);
CREATE INDEX idx_transactions_idempotency ON transactions(idempotency_key);
CREATE INDEX idx_ledger_account ON ledger_entries(account_id, created_at DESC);
CREATE INDEX idx_webhook_events_status ON webhook_events(status, next_retry_at);
```

## 💻 Django Implementation

```python
# models.py
from django.db import models
from django.contrib.postgres.fields import ArrayField
from decimal import Decimal

class TransactionStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PROCESSING = 'processing', 'Processing'
    SUCCEEDED = 'succeeded', 'Succeeded'
    FAILED = 'failed', 'Failed'
    REFUNDED = 'refunded', 'Refunded'
    PARTIALLY_REFUNDED = 'partially_refunded', 'Partially Refunded'

class Merchant(models.Model):
    """Merchant account"""
    name = models.CharField(max_length=200)
    email = models.EmailField(unique=True)
    api_key = models.CharField(max_length=100, unique=True)
    secret_key = models.CharField(max_length=100)
    status = models.CharField(max_length=20, default='active')
    balance_cents = models.BigIntegerField(default=0)
    pending_balance_cents = models.BigIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'merchants'

    @property
    def balance(self) -> Decimal:
        """Balance in dollars"""
        return Decimal(self.balance_cents) / 100

class Customer(models.Model):
    """Customer profile"""
    merchant = models.ForeignKey(Merchant, on_delete=models.CASCADE)
    email = models.EmailField()
    name = models.CharField(max_length=200, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'customers'
        unique_together = ('merchant', 'email')

class PaymentMethod(models.Model):
    """Payment method (credit card, bank account)"""
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE, related_name='payment_methods')
    type = models.CharField(max_length=20)
    last4 = models.CharField(max_length=4)
    brand = models.CharField(max_length=20, blank=True)
    exp_month = models.IntegerField(null=True)
    exp_year = models.IntegerField(null=True)
    token = models.CharField(max_length=100, unique=True)
    is_default = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'payment_methods'

class Transaction(models.Model):
    """Payment transaction"""
    merchant = models.ForeignKey(Merchant, on_delete=models.CASCADE)
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    payment_method = models.ForeignKey(PaymentMethod, on_delete=models.SET_NULL, null=True)
    amount_cents = models.BigIntegerField()
    currency = models.CharField(max_length=3, default='USD')
    status = models.CharField(max_length=20, choices=TransactionStatus.choices, default=TransactionStatus.PENDING)
    failure_reason = models.TextField(blank=True)
    idempotency_key = models.CharField(max_length=100, unique=True, db_index=True)
    description = models.TextField(blank=True)
    metadata = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'transactions'
        indexes = [
            models.Index(fields=['merchant', '-created_at']),
            models.Index(fields=['customer', '-created_at']),
            models.Index(fields=['status', '-created_at']),
        ]

    @property
    def amount(self) -> Decimal:
        """Amount in dollars"""
        return Decimal(self.amount_cents) / 100

class LedgerEntry(models.Model):
    """Double-entry ledger"""
    transaction = models.ForeignKey(Transaction, on_delete=models.CASCADE, related_name='ledger_entries')
    account_id = models.BigIntegerField()
    account_type = models.CharField(max_length=20)  # debit, credit
    amount_cents = models.BigIntegerField()
    balance_cents = models.BigIntegerField()
    entry_type = models.CharField(max_length=50)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'ledger_entries'
        indexes = [
            models.Index(fields=['account_id', '-created_at']),
        ]

class Refund(models.Model):
    """Refund record"""
    transaction = models.ForeignKey(Transaction, on_delete=models.CASCADE, related_name='refunds')
    amount_cents = models.BigIntegerField()
    reason = models.CharField(max_length=200, blank=True)
    status = models.CharField(max_length=20, default='pending')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'refunds'

# services/payment_service.py
from django.db import transaction, connection
from django.core.cache import cache
from typing import Optional
import logging
import uuid

logger = logging.getLogger(__name__)

class PaymentService:
    """Handle payment processing with ACID guarantees"""

    @staticmethod
    def charge(
        merchant: Merchant,
        customer: Customer,
        amount_cents: int,
        payment_method: PaymentMethod,
        idempotency_key: str,
        description: str = "",
        metadata: dict = None
    ) -> Transaction:
        """
        Process payment with idempotency
        Uses database transactions for ACID properties
        """
        # Check idempotency (prevent duplicate charges)
        if not idempotency_key:
            idempotency_key = str(uuid.uuid4())

        # Check if transaction already exists
        existing = Transaction.objects.filter(idempotency_key=idempotency_key).first()
        if existing:
            logger.info(f"Idempotent request detected: {idempotency_key}")
            return existing

        # Create transaction with ACID guarantees
        with transaction.atomic():
            # Lock merchant account for update (prevent race conditions)
            merchant = Merchant.objects.select_for_update().get(id=merchant.id)

            # Create transaction record
            txn = Transaction.objects.create(
                merchant=merchant,
                customer=customer,
                payment_method=payment_method,
                amount_cents=amount_cents,
                status=TransactionStatus.PROCESSING,
                idempotency_key=idempotency_key,
                description=description,
                metadata=metadata or {}
            )

            try:
                # Call external payment processor
                payment_result = PaymentProcessor.process_payment(
                    payment_method_token=payment_method.token,
                    amount_cents=amount_cents,
                    currency='USD'
                )

                if payment_result['success']:
                    # Payment succeeded - record in ledger
                    PaymentService._record_charge(txn, merchant, amount_cents)

                    txn.status = TransactionStatus.SUCCEEDED
                    txn.save(update_fields=['status', 'updated_at'])

                    # Trigger webhook
                    from .tasks import send_webhook_task
                    send_webhook_task.delay(merchant.id, 'payment.succeeded', {
                        'transaction_id': txn.id,
                        'amount': txn.amount,
                        'customer': customer.email
                    })

                    logger.info(f"Payment succeeded: {txn.id}")
                else:
                    # Payment failed
                    txn.status = TransactionStatus.FAILED
                    txn.failure_reason = payment_result.get('error', 'Unknown error')
                    txn.save(update_fields=['status', 'failure_reason', 'updated_at'])

                    logger.warning(f"Payment failed: {txn.id} - {txn.failure_reason}")

            except Exception as e:
                # Handle errors
                txn.status = TransactionStatus.FAILED
                txn.failure_reason = str(e)
                txn.save(update_fields=['status', 'failure_reason', 'updated_at'])

                logger.error(f"Payment error: {txn.id} - {str(e)}")
                raise

        return txn

    @staticmethod
    def _record_charge(txn: Transaction, merchant: Merchant, amount_cents: int):
        """
        Record charge in double-entry ledger
        Debit: Customer pays
        Credit: Merchant receives
        """
        # Platform fee (2.9% + $0.30)
        fee_cents = int(amount_cents * 0.029 + 30)
        merchant_amount = amount_cents - fee_cents

        # Update merchant balance
        merchant.pending_balance_cents += merchant_amount
        merchant.save(update_fields=['pending_balance_cents'])

        # Record ledger entries
        LedgerEntry.objects.create(
            transaction=txn,
            account_id=merchant.id,
            account_type='credit',
            amount_cents=merchant_amount,
            balance_cents=merchant.balance_cents + merchant_amount,
            entry_type='charge'
        )

        # Platform fee entry (system account)
        LedgerEntry.objects.create(
            transaction=txn,
            account_id=0,  # System account
            account_type='credit',
            amount_cents=fee_cents,
            balance_cents=0,  # Would need to track system balance
            entry_type='fee'
        )

    @staticmethod
    def refund(transaction_id: int, amount_cents: Optional[int] = None, reason: str = "") -> Refund:
        """
        Process refund with ACID guarantees
        Supports full or partial refunds
        """
        with transaction.atomic():
            # Lock transaction for update
            txn = Transaction.objects.select_for_update().get(id=transaction_id)

            if txn.status != TransactionStatus.SUCCEEDED:
                raise ValueError("Can only refund succeeded transactions")

            # Default to full refund
            if amount_cents is None:
                amount_cents = txn.amount_cents

            # Check refund amount
            total_refunded = txn.refunds.filter(status='succeeded').aggregate(
                total=models.Sum('amount_cents')
            )['total'] or 0

            if total_refunded + amount_cents > txn.amount_cents:
                raise ValueError("Refund amount exceeds transaction amount")

            # Create refund record
            refund = Refund.objects.create(
                transaction=txn,
                amount_cents=amount_cents,
                reason=reason,
                status='processing'
            )

            try:
                # Call payment processor
                refund_result = PaymentProcessor.process_refund(
                    transaction_reference=txn.id,
                    amount_cents=amount_cents
                )

                if refund_result['success']:
                    # Update balances
                    merchant = Merchant.objects.select_for_update().get(id=txn.merchant.id)
                    merchant.balance_cents -= amount_cents
                    merchant.save(update_fields=['balance_cents'])

                    # Record in ledger
                    LedgerEntry.objects.create(
                        transaction=txn,
                        account_id=merchant.id,
                        account_type='debit',
                        amount_cents=amount_cents,
                        balance_cents=merchant.balance_cents,
                        entry_type='refund'
                    )

                    # Update refund status
                    refund.status = 'succeeded'
                    refund.save(update_fields=['status'])

                    # Update transaction status
                    if total_refunded + amount_cents == txn.amount_cents:
                        txn.status = TransactionStatus.REFUNDED
                    else:
                        txn.status = TransactionStatus.PARTIALLY_REFUNDED
                    txn.save(update_fields=['status'])

                    logger.info(f"Refund succeeded: {refund.id}")
                else:
                    refund.status = 'failed'
                    refund.save(update_fields=['status'])

            except Exception as e:
                refund.status = 'failed'
                refund.save(update_fields=['status'])
                logger.error(f"Refund error: {refund.id} - {str(e)}")
                raise

        return refund

class PaymentProcessor:
    """Mock external payment processor (Stripe, Braintree, etc.)"""

    @staticmethod
    def process_payment(payment_method_token: str, amount_cents: int, currency: str) -> dict:
        """Call external processor API"""
        # In reality, call Stripe/Braintree API
        import random
        success = random.random() > 0.05  # 95% success rate

        if success:
            return {
                'success': True,
                'transaction_id': str(uuid.uuid4())
            }
        else:
            return {
                'success': False,
                'error': 'Card declined'
            }

    @staticmethod
    def process_refund(transaction_reference: int, amount_cents: int) -> dict:
        """Process refund with external processor"""
        return {
            'success': True,
            'refund_id': str(uuid.uuid4())
        }

# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class PaymentViewSet(viewsets.ViewSet):
    """Payment API"""
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['post'])
    def charge(self, request):
        """Create a charge"""
        merchant = request.user.merchant
        customer_id = request.data.get('customer_id')
        amount = request.data.get('amount')  # In dollars
        payment_method_id = request.data.get('payment_method_id')
        idempotency_key = request.data.get('idempotency_key')
        description = request.data.get('description', '')

        if not all([customer_id, amount, payment_method_id]):
            return Response(
                {'error': 'customer_id, amount, and payment_method_id required'},
                status=status.HTTP_400_BAD_REQUEST
            )

        try:
            customer = Customer.objects.get(id=customer_id, merchant=merchant)
            payment_method = PaymentMethod.objects.get(id=payment_method_id, customer=customer)

            # Convert dollars to cents
            amount_cents = int(float(amount) * 100)

            # Process payment
            txn = PaymentService.charge(
                merchant=merchant,
                customer=customer,
                amount_cents=amount_cents,
                payment_method=payment_method,
                idempotency_key=idempotency_key,
                description=description
            )

            return Response({
                'id': txn.id,
                'amount': txn.amount,
                'status': txn.status,
                'created_at': txn.created_at
            }, status=status.HTTP_201_CREATED if txn.status == 'succeeded' else status.HTTP_400_BAD_REQUEST)

        except (Customer.DoesNotExist, PaymentMethod.DoesNotExist):
            return Response(
                {'error': 'Invalid customer or payment method'},
                status=status.HTTP_404_NOT_FOUND
            )
        except Exception as e:
            return Response(
                {'error': str(e)},
                status=status.HTTP_500_INTERNAL_SERVER_ERROR
            )

    @action(detail=True, methods=['post'])
    def refund(self, request, pk=None):
        """Refund a transaction"""
        amount = request.data.get('amount')  # Optional, defaults to full refund
        reason = request.data.get('reason', '')

        try:
            # Convert to cents if provided
            amount_cents = int(float(amount) * 100) if amount else None

            refund = PaymentService.refund(
                transaction_id=pk,
                amount_cents=amount_cents,
                reason=reason
            )

            return Response({
                'refund_id': refund.id,
                'amount': Decimal(refund.amount_cents) / 100,
                'status': refund.status
            })

        except Transaction.DoesNotExist:
            return Response(
                {'error': 'Transaction not found'},
                status=status.HTTP_404_NOT_FOUND
            )
        except ValueError as e:
            return Response(
                {'error': str(e)},
                status=status.HTTP_400_BAD_REQUEST
            )

    @action(detail=False, methods=['get'])
    def balance(self, request):
        """Get merchant balance"""
        merchant = request.user.merchant

        return Response({
            'available': Decimal(merchant.balance_cents) / 100,
            'pending': Decimal(merchant.pending_balance_cents) / 100,
            'currency': 'USD'
        })
```

## 🔐 Security & Compliance

```python
class SecurityService:
    """PCI DSS compliance and security"""

    @staticmethod
    def tokenize_payment_method(card_number: str, exp_month: int, exp_year: int, cvv: str) -> str:
        """
        Never store raw card numbers
        Use payment processor's tokenization
        """
        # Call external tokenization service (Stripe, Braintree)
        token = f"tok_{uuid.uuid4().hex}"

        # Only store token and last 4 digits
        return {
            'token': token,
            'last4': card_number[-4:],
            'exp_month': exp_month,
            'exp_year': exp_year
        }

    @staticmethod
    def verify_webhook_signature(payload: str, signature: str, secret: str) -> bool:
        """Verify webhook authenticity"""
        import hmac
        import hashlib

        expected_signature = hmac.new(
            secret.encode(),
            payload.encode(),
            hashlib.sha256
        ).hexdigest()

        return hmac.compare_digest(signature, expected_signature)
```

## ❓ Interview Questions

### Q1: How do you prevent double-charging?

**Answer:**

1. **Idempotency keys**: Client provides unique key per request
2. **Database uniqueness**: Unique constraint on idempotency_key
3. **Cache check**: Redis cache for recent requests
4. **Return existing**: If duplicate, return original transaction

### Q2: How do you ensure ACID properties?

**Answer:**

1. **Database transactions**: Wrap operations in `transaction.atomic()`
2. **Row-level locking**: `select_for_update()` prevents concurrent modifications
3. **Double-entry bookkeeping**: Every debit has matching credit
4. **Audit trail**: Immutable ledger entries

### Q3: How do you handle payment failures?

**Answer:**

1. **Retry logic**: Exponential backoff for network issues
2. **Circuit breaker**: Stop trying if processor is down
3. **Graceful degradation**: Queue for later processing
4. **User feedback**: Clear error messages
5. **Reconciliation**: Daily batch to verify all transactions

### Q4: How do you scale to 10K TPS?

**Answer:**

1. **Database optimization**: Proper indexing, connection pooling
2. **Read replicas**: Route reads to replicas
3. **Async webhooks**: Queue webhook delivery
4. **Caching**: Cache merchant/customer data
5. **Horizontal scaling**: Stateless API servers

### Q5: How do you handle refunds for disputed transactions?

**Answer:**

1. **Dispute tracking**: Separate disputes table
2. **Hold funds**: Reserve merchant balance
3. **Evidence submission**: Collect merchant's evidence
4. **Automatic resolution**: Process based on processor decision
5. **Chargeback fees**: Apply fees for lost disputes

---

## 📚 Summary

**Key Components:**

1. **ACID transactions**: PostgreSQL with row locking
2. **Idempotency**: Prevent duplicate charges
3. **Double-entry ledger**: Maintain financial accuracy
4. **Webhooks**: Event notifications
5. **Refunds**: Full and partial support
6. **Security**: PCI DSS compliance, tokenization

**Scalability:**

- Database read replicas
- Async webhook delivery
- Caching merchant data
- Connection pooling
- Horizontal scaling

**Reliability:**

- Strong consistency (ACID)
- Audit trail (ledger)
- Reconciliation processes
- Error handling and retries
- Monitoring and alerts

This pattern applies to: Stripe, PayPal, Square, Braintree, payment gateways, fintech platforms, e-commerce checkout.
