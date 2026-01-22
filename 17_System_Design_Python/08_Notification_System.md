# System Design: Notification System

## 📖 Problem Statement

Design a multi-channel notification system that delivers messages to users via different channels (email, SMS, push notifications, in-app) with priority levels, delivery tracking, and retry mechanisms.

**Core Features:**

- Send notifications via multiple channels
- Template management
- Priority-based delivery
- User preferences (opt-in/opt-out)
- Delivery tracking and retry
- Rate limiting and throttling

## 🎯 Requirements

### Functional Requirements

1. **Support multiple channels**: Email, SMS, push, in-app, webhook
2. **Priority levels**: Low, medium, high, critical
3. **Template management**: Reusable templates with variables
4. **User preferences**: Per-channel opt-in/opt-out
5. **Delivery tracking**: Status, delivery time, read receipts
6. **Retry mechanism**: Exponential backoff for failures
7. **Scheduled delivery**: Send at specific time

### Non-Functional Requirements

1. **Scalability**: Handle 100M notifications/day
2. **Reliability**: 99.9% delivery rate
3. **Low Latency**: High-priority notifications < 10s
4. **Rate Limiting**: Respect provider limits (email, SMS)
5. **Availability**: 99.99% uptime
6. **Idempotency**: No duplicate notifications

## 📊 Capacity Estimation

```python
class NotificationEstimation:
    """Calculate notification system capacity"""

    def __init__(self):
        self.daily_active_users = 50_000_000  # 50M
        self.notifications_per_user_per_day = 2

        # Channel distribution
        self.email_ratio = 0.5  # 50% via email
        self.push_ratio = 0.3   # 30% via push
        self.sms_ratio = 0.15   # 15% via SMS
        self.in_app_ratio = 0.05  # 5% in-app

        # Provider costs (per notification)
        self.email_cost = 0.0001  # $0.0001
        self.push_cost = 0.00001  # $0.00001
        self.sms_cost = 0.01      # $0.01

    def calculate_traffic(self):
        """Calculate daily traffic"""
        daily_notifications = self.daily_active_users * self.notifications_per_user_per_day
        qps = daily_notifications / (24 * 3600)
        peak_qps = qps * 5  # Peak is 5x average

        print("=== Traffic Estimation ===")
        print(f"Daily Notifications: {daily_notifications:,.0f}")
        print(f"Average QPS: {qps:,.0f}")
        print(f"Peak QPS: {peak_qps:,.0f}\n")

        return daily_notifications

    def calculate_by_channel(self, daily_total):
        """Calculate per-channel volume"""
        print("=== Channel Distribution ===")
        print(f"Email: {daily_total * self.email_ratio:,.0f}")
        print(f"Push: {daily_total * self.push_ratio:,.0f}")
        print(f"SMS: {daily_total * self.sms_ratio:,.0f}")
        print(f"In-App: {daily_total * self.in_app_ratio:,.0f}\n")

    def calculate_cost(self, daily_total):
        """Calculate daily cost"""
        email_cost = daily_total * self.email_ratio * self.email_cost
        push_cost = daily_total * self.push_ratio * self.push_cost
        sms_cost = daily_total * self.sms_ratio * self.sms_cost

        total_cost = email_cost + push_cost + sms_cost

        print("=== Daily Cost Estimation ===")
        print(f"Email: ${email_cost:,.2f}")
        print(f"Push: ${push_cost:,.2f}")
        print(f"SMS: ${sms_cost:,.2f}")
        print(f"Total: ${total_cost:,.2f}")
        print(f"Monthly: ${total_cost * 30:,.2f}\n")

# Run estimation
estimator = NotificationEstimation()
daily_total = estimator.calculate_traffic()
estimator.calculate_by_channel(daily_total)
estimator.calculate_cost(daily_total)
```

**Output:**

```
=== Traffic Estimation ===
Daily Notifications: 100,000,000
Average QPS: 1,157
Peak QPS: 5,787

=== Channel Distribution ===
Email: 50,000,000
Push: 30,000,000
SMS: 15,000,000
In-App: 5,000,000

=== Daily Cost Estimation ===
Email: $5,000.00
Push: $300.00
SMS: $150,000.00
Total: $155,300.00
Monthly: $4,659,000.00
```

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│         Application Services                     │
│  (Django/FastAPI, Microservices)                │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ Notification API │
        │   (Gateway)      │
        └────────┬─────────┘
                 │
                 ▼
        ┌─────────────────┐
        │  Message Queue   │
        │   (RabbitMQ)     │
        └────────┬─────────┘
                 │
       ┌─────────┼─────────┬─────────┐
       ▼         ▼         ▼         ▼
  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
  │ Email  │ │  SMS   │ │  Push  │ │ In-App │
  │Worker  │ │Worker  │ │Worker  │ │Worker  │
  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
      │          │          │          │
      ▼          ▼          ▼          ▼
  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
  │SendGrid│ │ Twilio │ │  FCM   │ │ WebSocket
  └────────┘ └────────┘ └────────┘ └────────┘
       │          │          │          │
       └──────────┴──────────┴──────────┘
                  │
                  ▼
        ┌──────────────────┐
        │  Tracking DB      │
        │  (PostgreSQL)     │
        └──────────────────┘
```

## 🔑 Database Schema

```sql
-- Notification templates
CREATE TABLE notification_templates (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    channel VARCHAR(20) NOT NULL,  -- email, sms, push, in_app
    subject VARCHAR(200),  -- For email
    body TEXT NOT NULL,
    variables JSONB,  -- Template variables
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User preferences
CREATE TABLE user_notification_preferences (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    channel VARCHAR(20) NOT NULL,
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, channel)
);

-- Notification queue
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    channel VARCHAR(20) NOT NULL,
    priority VARCHAR(20) DEFAULT 'medium',  -- low, medium, high, critical
    template_id BIGINT REFERENCES notification_templates(id),
    subject VARCHAR(200),
    body TEXT NOT NULL,
    metadata JSONB,  -- Additional data
    status VARCHAR(20) DEFAULT 'pending',  -- pending, sent, failed, cancelled
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    read_at TIMESTAMP,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Delivery logs
CREATE TABLE notification_logs (
    id BIGSERIAL PRIMARY KEY,
    notification_id BIGINT REFERENCES notifications(id),
    status VARCHAR(20) NOT NULL,
    provider_response JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_notifications_user_status ON notifications(user_id, status);
CREATE INDEX idx_notifications_scheduled ON notifications(scheduled_at) WHERE status = 'pending';
CREATE INDEX idx_notifications_retry ON notifications(retry_count) WHERE status = 'failed';
```

## 💻 Django Implementation

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
from django.contrib.postgres.fields import JSONField

class NotificationChannel(models.TextChoices):
    EMAIL = 'email', 'Email'
    SMS = 'sms', 'SMS'
    PUSH = 'push', 'Push Notification'
    IN_APP = 'in_app', 'In-App'
    WEBHOOK = 'webhook', 'Webhook'

class NotificationPriority(models.TextChoices):
    LOW = 'low', 'Low'
    MEDIUM = 'medium', 'Medium'
    HIGH = 'high', 'High'
    CRITICAL = 'critical', 'Critical'

class NotificationStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    QUEUED = 'queued', 'Queued'
    SENDING = 'sending', 'Sending'
    SENT = 'sent', 'Sent'
    DELIVERED = 'delivered', 'Delivered'
    READ = 'read', 'Read'
    FAILED = 'failed', 'Failed'
    CANCELLED = 'cancelled', 'Cancelled'

class NotificationTemplate(models.Model):
    """Reusable notification templates"""
    name = models.CharField(max_length=100, unique=True)
    channel = models.CharField(max_length=20, choices=NotificationChannel.choices)
    subject = models.CharField(max_length=200, blank=True)  # For email
    body = models.TextField()
    variables = models.JSONField(default=dict)  # Expected variables
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'notification_templates'

    def render(self, context: dict) -> tuple:
        """Render template with context variables"""
        from string import Template

        subject = Template(self.subject).safe_substitute(context) if self.subject else ""
        body = Template(self.body).safe_substitute(context)

        return subject, body

class UserNotificationPreference(models.Model):
    """User's notification preferences per channel"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    channel = models.CharField(max_length=20, choices=NotificationChannel.choices)
    enabled = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'user_notification_preferences'
        unique_together = ('user', 'channel')

class Notification(models.Model):
    """Notification record"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    channel = models.CharField(max_length=20, choices=NotificationChannel.choices)
    priority = models.CharField(
        max_length=20,
        choices=NotificationPriority.choices,
        default=NotificationPriority.MEDIUM
    )
    template = models.ForeignKey(
        NotificationTemplate,
        on_delete=models.SET_NULL,
        null=True,
        blank=True
    )
    subject = models.CharField(max_length=200, blank=True)
    body = models.TextField()
    metadata = models.JSONField(default=dict)
    status = models.CharField(
        max_length=20,
        choices=NotificationStatus.choices,
        default=NotificationStatus.PENDING
    )
    scheduled_at = models.DateTimeField(null=True, blank=True)
    sent_at = models.DateTimeField(null=True, blank=True)
    delivered_at = models.DateTimeField(null=True, blank=True)
    read_at = models.DateTimeField(null=True, blank=True)
    retry_count = models.IntegerField(default=0)
    max_retries = models.IntegerField(default=3)
    error_message = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'notifications'
        indexes = [
            models.Index(fields=['user', 'status']),
            models.Index(fields=['scheduled_at']),
        ]

class NotificationLog(models.Model):
    """Delivery attempt logs"""
    notification = models.ForeignKey(Notification, on_delete=models.CASCADE)
    status = models.CharField(max_length=20, choices=NotificationStatus.choices)
    provider_response = models.JSONField(default=dict)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'notification_logs'

# services.py
from django.utils import timezone
from typing import Optional, Dict, Any
import logging

logger = logging.getLogger(__name__)

class NotificationService:
    """Core notification service"""

    @staticmethod
    def create_notification(
        user: User,
        channel: str,
        subject: str,
        body: str,
        priority: str = 'medium',
        template_name: Optional[str] = None,
        template_context: Optional[Dict] = None,
        scheduled_at: Optional[timezone.datetime] = None,
        metadata: Optional[Dict] = None
    ) -> Notification:
        """
        Create notification (to be sent)
        """
        # Check user preferences
        if not NotificationService._is_channel_enabled(user, channel):
            logger.info(f"User {user.id} has disabled {channel} notifications")
            return None

        # Use template if provided
        if template_name:
            try:
                template = NotificationTemplate.objects.get(
                    name=template_name,
                    channel=channel,
                    is_active=True
                )
                subject, body = template.render(template_context or {})
            except NotificationTemplate.DoesNotExist:
                logger.error(f"Template {template_name} not found")
                raise

        # Create notification
        notification = Notification.objects.create(
            user=user,
            channel=channel,
            priority=priority,
            subject=subject,
            body=body,
            metadata=metadata or {},
            scheduled_at=scheduled_at,
            status=NotificationStatus.PENDING
        )

        # Queue for sending
        if not scheduled_at or scheduled_at <= timezone.now():
            NotificationService._queue_notification(notification)

        logger.info(f"Created notification {notification.id} for user {user.id}")
        return notification

    @staticmethod
    def _is_channel_enabled(user: User, channel: str) -> bool:
        """Check if user has enabled this channel"""
        pref, _ = UserNotificationPreference.objects.get_or_create(
            user=user,
            channel=channel,
            defaults={'enabled': True}
        )
        return pref.enabled

    @staticmethod
    def _queue_notification(notification: Notification):
        """Send notification to message queue"""
        from .tasks import send_notification_task

        # Update status
        notification.status = NotificationStatus.QUEUED
        notification.save(update_fields=['status'])

        # Queue with priority
        priority_map = {
            'low': 0,
            'medium': 5,
            'high': 7,
            'critical': 9
        }

        send_notification_task.apply_async(
            args=[notification.id],
            priority=priority_map.get(notification.priority, 5)
        )

# tasks.py (Celery)
from celery import shared_task
from django.core.mail import send_mail
from twilio.rest import Client
import firebase_admin
from firebase_admin import messaging

@shared_task(bind=True, max_retries=3)
def send_notification_task(self, notification_id: int):
    """
    Send notification via appropriate channel
    Retry on failure with exponential backoff
    """
    try:
        notification = Notification.objects.get(id=notification_id)

        # Update status
        notification.status = NotificationStatus.SENDING
        notification.save(update_fields=['status'])

        # Send via appropriate channel
        if notification.channel == NotificationChannel.EMAIL:
            success = send_email_notification(notification)
        elif notification.channel == NotificationChannel.SMS:
            success = send_sms_notification(notification)
        elif notification.channel == NotificationChannel.PUSH:
            success = send_push_notification(notification)
        elif notification.channel == NotificationChannel.IN_APP:
            success = send_in_app_notification(notification)
        else:
            raise ValueError(f"Unknown channel: {notification.channel}")

        if success:
            notification.status = NotificationStatus.SENT
            notification.sent_at = timezone.now()
            notification.save(update_fields=['status', 'sent_at'])

            # Log success
            NotificationLog.objects.create(
                notification=notification,
                status=NotificationStatus.SENT
            )

            logger.info(f"Sent notification {notification_id}")
        else:
            raise Exception("Provider returned failure")

    except Exception as exc:
        notification.retry_count += 1
        notification.error_message = str(exc)

        if notification.retry_count >= notification.max_retries:
            # Max retries reached
            notification.status = NotificationStatus.FAILED
            notification.save(update_fields=['status', 'retry_count', 'error_message'])

            logger.error(f"Notification {notification_id} failed after {notification.retry_count} retries")
        else:
            # Retry with exponential backoff
            notification.save(update_fields=['retry_count', 'error_message'])

            # Retry after 2^retry_count minutes
            countdown = 60 * (2 ** notification.retry_count)
            raise self.retry(exc=exc, countdown=countdown)

def send_email_notification(notification: Notification) -> bool:
    """Send email via SendGrid"""
    try:
        send_mail(
            subject=notification.subject,
            message=notification.body,
            from_email='noreply@example.com',
            recipient_list=[notification.user.email],
            fail_silently=False
        )
        return True
    except Exception as e:
        logger.error(f"Email send failed: {e}")
        return False

def send_sms_notification(notification: Notification) -> bool:
    """Send SMS via Twilio"""
    try:
        client = Client('account_sid', 'auth_token')

        message = client.messages.create(
            body=notification.body,
            from_='+1234567890',
            to=notification.user.phone_number
        )

        return message.status != 'failed'
    except Exception as e:
        logger.error(f"SMS send failed: {e}")
        return False

def send_push_notification(notification: Notification) -> bool:
    """Send push via Firebase Cloud Messaging"""
    try:
        # Get user's FCM token
        device_token = notification.user.profile.fcm_token

        message = messaging.Message(
            notification=messaging.Notification(
                title=notification.subject,
                body=notification.body
            ),
            token=device_token
        )

        response = messaging.send(message)
        return bool(response)
    except Exception as e:
        logger.error(f"Push send failed: {e}")
        return False

def send_in_app_notification(notification: Notification) -> bool:
    """Send in-app notification via WebSocket"""
    from channels.layers import get_channel_layer
    from asgiref.sync import async_to_sync

    try:
        channel_layer = get_channel_layer()

        async_to_sync(channel_layer.group_send)(
            f"user_{notification.user.id}",
            {
                'type': 'notification',
                'data': {
                    'id': notification.id,
                    'subject': notification.subject,
                    'body': notification.body,
                    'created_at': notification.created_at.isoformat()
                }
            }
        )

        return True
    except Exception as e:
        logger.error(f"In-app notification failed: {e}")
        return False

# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class NotificationViewSet(viewsets.ModelViewSet):
    """Notification API"""
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Notification.objects.filter(user=self.request.user)

    @action(detail=False, methods=['post'])
    def send(self, request):
        """Send notification to user"""
        user_id = request.data.get('user_id')
        channel = request.data.get('channel')
        subject = request.data.get('subject', '')
        body = request.data.get('body')
        priority = request.data.get('priority', 'medium')

        if not all([user_id, channel, body]):
            return Response(
                {'error': 'user_id, channel, and body required'},
                status=status.HTTP_400_BAD_REQUEST
            )

        try:
            user = User.objects.get(id=user_id)

            notification = NotificationService.create_notification(
                user=user,
                channel=channel,
                subject=subject,
                body=body,
                priority=priority
            )

            if notification:
                return Response({
                    'notification_id': notification.id,
                    'status': notification.status
                }, status=status.HTTP_201_CREATED)
            else:
                return Response({
                    'message': 'User has disabled this channel'
                }, status=status.HTTP_200_OK)

        except User.DoesNotExist:
            return Response(
                {'error': 'User not found'},
                status=status.HTTP_404_NOT_FOUND
            )

    @action(detail=True, methods=['post'])
    def mark_read(self, request, pk=None):
        """Mark notification as read"""
        notification = self.get_object()
        notification.read_at = timezone.now()
        notification.status = NotificationStatus.READ
        notification.save(update_fields=['read_at', 'status'])

        return Response({'status': 'read'})

    @action(detail=False, methods=['get'])
    def unread_count(self, request):
        """Get unread notification count"""
        count = Notification.objects.filter(
            user=request.user,
            read_at__isnull=True
        ).count()

        return Response({'unread_count': count})

class PreferenceViewSet(viewsets.ViewSet):
    """User notification preferences"""
    permission_classes = [IsAuthenticated]

    def list(self, request):
        """Get user's preferences"""
        prefs = UserNotificationPreference.objects.filter(user=request.user)

        return Response({
            pref.channel: pref.enabled
            for pref in prefs
        })

    @action(detail=False, methods=['post'])
    def update(self, request):
        """Update preferences"""
        channel = request.data.get('channel')
        enabled = request.data.get('enabled', True)

        pref, created = UserNotificationPreference.objects.update_or_create(
            user=request.user,
            channel=channel,
            defaults={'enabled': enabled}
        )

        return Response({
            'channel': channel,
            'enabled': enabled
        })
```

## 🚀 FastAPI Implementation

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import Optional, Dict, Any
from enum import Enum
from datetime import datetime

app = FastAPI()

class ChannelEnum(str, Enum):
    EMAIL = "email"
    SMS = "sms"
    PUSH = "push"
    IN_APP = "in_app"

class PriorityEnum(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class NotificationCreate(BaseModel):
    user_id: int
    channel: ChannelEnum
    subject: str = ""
    body: str
    priority: PriorityEnum = PriorityEnum.MEDIUM
    template_name: Optional[str] = None
    template_context: Optional[Dict[str, Any]] = None
    scheduled_at: Optional[datetime] = None

class NotificationResponse(BaseModel):
    id: int
    user_id: int
    channel: str
    status: str
    created_at: datetime

@app.post("/api/notifications", response_model=NotificationResponse)
async def create_notification(
    notification_data: NotificationCreate,
    background_tasks: BackgroundTasks
):
    """Create and send notification"""
    # Create notification
    notification = await NotificationService.create_notification(
        user_id=notification_data.user_id,
        channel=notification_data.channel,
        subject=notification_data.subject,
        body=notification_data.body,
        priority=notification_data.priority
    )

    # Queue for sending
    background_tasks.add_task(send_notification_async, notification.id)

    return notification

@app.get("/api/notifications/unread")
async def get_unread_count(current_user: User = Depends(get_current_user)):
    """Get unread notification count"""
    count = await get_unread_notification_count(current_user.id)
    return {"unread_count": count}

@app.post("/api/notifications/{notification_id}/read")
async def mark_as_read(
    notification_id: int,
    current_user: User = Depends(get_current_user)
):
    """Mark notification as read"""
    await mark_notification_read(notification_id, current_user.id)
    return {"status": "read"}
```

## 🔐 Rate Limiting & Anti-Spam

```python
from django.core.cache import cache
from datetime import timedelta

class NotificationRateLimiter:
    """Prevent notification spam"""

    @staticmethod
    def check_rate_limit(
        user_id: int,
        channel: str,
        limit: int = 10,
        window: int = 3600
    ) -> bool:
        """
        Check if user has exceeded notification limit
        Args:
            user_id: User ID
            channel: Notification channel
            limit: Max notifications per window
            window: Time window in seconds (default 1 hour)
        """
        cache_key = f"notif_rate:{user_id}:{channel}"
        count = cache.get(cache_key, 0)

        if count >= limit:
            return False

        # Increment counter
        cache.set(cache_key, count + 1, timeout=window)
        return True

    @staticmethod
    def should_batch(user_id: int, notification_type: str) -> bool:
        """
        Decide if notifications should be batched
        E.g., batch email digests instead of sending individually
        """
        # Get user's recent notifications
        recent_count = Notification.objects.filter(
            user_id=user_id,
            metadata__notification_type=notification_type,
            created_at__gte=timezone.now() - timedelta(hours=1)
        ).count()

        # Batch if > 5 notifications of same type in last hour
        return recent_count > 5
```

## ❓ Interview Questions

### Q1: How do you handle notification delivery failures?

**Answer:**

1. **Retry with exponential backoff**: Wait 1min, 2min, 4min between retries
2. **Dead letter queue**: Move permanently failed notifications
3. **Fallback channel**: If email fails, try SMS or push
4. **Circuit breaker**: Stop trying if provider is down
5. **Alert ops team**: Monitor failure rates

### Q2: How do you prevent duplicate notifications?

**Answer:**

1. **Idempotency key**: Check if notification with same key exists
2. **Database uniqueness**: Unique constraint on (user, type, reference_id)
3. **Deduplication window**: Prevent duplicates within X minutes
4. **Message queue deduplication**: RabbitMQ/SQS built-in

### Q3: How do you handle different time zones?

**Answer:**

1. **Store user timezone** in profile
2. **Convert scheduled time** to user's timezone
3. **Send at optimal time**: 9 AM in user's timezone, not server's
4. **Respect quiet hours**: Don't send between 10 PM - 8 AM user time

### Q4: How do you scale for 1M notifications/second?

**Answer:**

1. **Partition message queues** by priority and channel
2. **Horizontal scaling**: More workers for each channel
3. **Batch API calls**: Send 100 emails at once (provider batching)
4. **Database sharding**: Shard by user_id
5. **Async everything**: No blocking operations

### Q5: How do you track delivery and read status?

**Answer:**

1. **Email**: Tracking pixel, link tracking
2. **SMS**: Provider delivery webhooks (Twilio)
3. **Push**: FCM delivery receipts
4. **In-app**: Client sends read event via WebSocket
5. **Store in logs table**: Audit trail for compliance

---

## 📚 Summary

**Key Components:**

1. **Multi-channel support**: Email, SMS, push, in-app
2. **Priority queue**: RabbitMQ/Celery with priorities
3. **Template system**: Reusable templates with variables
4. **Retry mechanism**: Exponential backoff
5. **User preferences**: Per-channel opt-in/opt-out
6. **Delivery tracking**: Logs and status updates

**Scalability:**

- Message queue for async processing
- Database sharding by user_id
- Caching for preferences
- Provider batching for efficiency
- Auto-scaling workers

**Best Practices:**

- Idempotency for no duplicates
- Rate limiting to prevent spam
- Respect user preferences
- Monitor delivery rates
- Fallback channels for failures

This pattern applies to: Email services, SMS gateways, push notification systems, real-time alerts, webhook delivery.
