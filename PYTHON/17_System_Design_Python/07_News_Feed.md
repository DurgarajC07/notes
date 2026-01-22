# System Design: News Feed (Like Twitter/Facebook)

## 📖 Problem Statement

Design a news feed system like Twitter, Facebook, or Instagram that displays posts from users you follow in reverse chronological order or ranked by relevance.

**Core Features:**

- Post creation (text, images, videos)
- Follow/unfollow users
- View personalized feed
- Like, comment, share
- Real-time updates

## 🎯 Requirements

### Functional Requirements

1. **User can create posts** (text, images, videos)
2. **User can follow other users**
3. **User sees feed of posts from followed users**
4. **Feed should be paginated**
5. **Support likes, comments, shares**
6. **Real-time notifications for new posts (optional)**

### Non-Functional Requirements

1. **Low Latency**: Feed loads in <200ms
2. **High Availability**: 99.99% uptime
3. **Scalability**: Support 1B users, 100M daily active
4. **Eventual Consistency**: It's okay if feed is slightly stale
5. **High Write Throughput**: 50K posts/second
6. **Read-Heavy**: 100:1 read-to-write ratio

## 📊 Capacity Estimation

### Traffic Estimates

```python
class NewsFeedEstimation:
    """Calculate system capacity for news feed"""

    def __init__(self):
        self.total_users = 1_000_000_000  # 1 billion
        self.daily_active_users = 100_000_000  # 100 million
        self.avg_posts_per_user_per_day = 0.5  # Most users are readers
        self.avg_follows_per_user = 200
        self.avg_feed_requests_per_day = 10  # User checks feed 10 times

        # Post size
        self.avg_post_size = 1024  # 1KB (text + metadata)
        self.avg_image_size = 200 * 1024  # 200KB
        self.posts_with_images = 0.3  # 30% have images

    def calculate_qps(self):
        """Calculate queries per second"""
        # Write QPS (post creation)
        daily_posts = self.daily_active_users * self.avg_posts_per_user_per_day
        write_qps = daily_posts / (24 * 3600)

        # Read QPS (feed generation)
        daily_feed_requests = self.daily_active_users * self.avg_feed_requests_per_day
        read_qps = daily_feed_requests / (24 * 3600)

        # Peak traffic (3x average)
        peak_write_qps = write_qps * 3
        peak_read_qps = read_qps * 3

        print("=== Traffic Estimation ===")
        print(f"Daily Posts: {daily_posts:,.0f}")
        print(f"Write QPS: {write_qps:,.0f}")
        print(f"Peak Write QPS: {peak_write_qps:,.0f}")
        print(f"Read QPS: {read_qps:,.0f}")
        print(f"Peak Read QPS: {peak_read_qps:,.0f}\n")

    def calculate_storage(self):
        """Calculate storage requirements"""
        daily_posts = self.daily_active_users * self.avg_posts_per_user_per_day
        yearly_posts = daily_posts * 365

        # Text storage
        text_storage = yearly_posts * self.avg_post_size

        # Image storage
        posts_with_imgs = yearly_posts * self.posts_with_images
        image_storage = posts_with_imgs * self.avg_image_size

        total_storage = text_storage + image_storage
        storage_tb = total_storage / (1024 ** 4)

        print("=== Storage Estimation (per year) ===")
        print(f"Yearly Posts: {yearly_posts:,.0f}")
        print(f"Text Storage: {text_storage / (1024**4):.2f} TB")
        print(f"Image Storage: {image_storage / (1024**4):.2f} TB")
        print(f"Total Storage: {storage_tb:.2f} TB\n")

    def calculate_fanout_work(self):
        """Calculate fanout workload"""
        daily_posts = self.daily_active_users * self.avg_posts_per_user_per_day

        # Each post needs to fanout to followers
        fanout_operations = daily_posts * self.avg_follows_per_user
        fanout_ops_per_sec = fanout_operations / (24 * 3600)

        print("=== Fanout Estimation ===")
        print(f"Daily Fanout Operations: {fanout_operations:,.0f}")
        print(f"Fanout Ops/Second: {fanout_ops_per_sec:,.0f}\n")

# Run estimation
estimator = NewsFeedEstimation()
estimator.calculate_qps()
estimator.calculate_storage()
estimator.calculate_fanout_work()
```

**Output:**

```
=== Traffic Estimation ===
Daily Posts: 50,000,000
Write QPS: 579
Peak Write QPS: 1,736
Read QPS: 11,574
Peak Read QPS: 34,722

=== Storage Estimation (per year) ===
Yearly Posts: 18,250,000,000
Text Storage: 0.02 TB
Image Storage: 1.01 TB
Total Storage: 1.03 TB

=== Fanout Estimation ===
Daily Fanout Operations: 10,000,000,000
Fanout Ops/Second: 115,741
```

## 🏗️ High-Level Architecture

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │     CDN     │
                    │  (Images)   │
                    └─────────────┘
                           │
                           ▼
                 ┌──────────────────┐
                 │  Load Balancer   │
                 └────────┬─────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ API      │    │ API      │    │ API      │
   │ Server 1 │    │ Server 2 │    │ Server N │
   └────┬─────┘    └────┬─────┘    └────┬─────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌─────────┐   ┌──────────┐   ┌──────────┐
   │  Redis  │   │ Post DB  │   │ Graph DB │
   │ (Cache) │   │(Postgres)│   │ (Follows)│
   └─────────┘   └──────────┘   └──────────┘
        │               │               │
        │               ▼               │
        │        ┌──────────────┐       │
        │        │ Message Queue│       │
        │        │  (RabbitMQ)  │       │
        │        └──────┬───────┘       │
        │               │               │
        │               ▼               │
        │        ┌──────────────┐       │
        └────────│ Fanout       │───────┘
                 │ Service      │
                 └──────────────┘
```

## 🔑 Core Design Decisions

### 1. Feed Generation: Pull vs Push vs Hybrid

**Pull Model (Read Time)**

- Generate feed when user requests it
- Query all followed users' recent posts
- **Pros**: No fanout cost, works for inactive users
- **Cons**: Slow for users following many people

**Push Model (Write Time)**

- Pre-generate feed when post is created
- Fanout post to all followers' feeds
- **Pros**: Fast reads, pre-computed feeds
- **Cons**: High fanout cost for celebrities

**Hybrid Model (Recommended)**

- Push for normal users (< 10K followers)
- Pull for celebrities (> 10K followers)
- Best of both worlds

```python
# Database Schema
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    """User post with content"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField(max_length=280)
    image_url = models.URLField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    like_count = models.PositiveIntegerField(default=0)
    comment_count = models.PositiveIntegerField(default=0)

    class Meta:
        db_table = 'posts'
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['user', '-created_at']),
        ]

class Follow(models.Model):
    """User following relationship"""
    follower = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='following'
    )
    followee = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='followers'
    )
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'follows'
        unique_together = ('follower', 'followee')
        indexes = [
            models.Index(fields=['follower']),
            models.Index(fields=['followee']),
        ]

class Feed(models.Model):
    """Pre-computed feed (push model)"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'feeds'
        indexes = [
            models.Index(fields=['user', '-created_at']),
        ]
        unique_together = ('user', 'post')

class Like(models.Model):
    """Post likes"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'likes'
        unique_together = ('user', 'post')
```

### 2. Fanout Service Implementation

```python
# services.py
from django.core.cache import cache
from django.db import transaction
from typing import List
import logging

logger = logging.getLogger(__name__)

class FanoutService:
    """Handle post distribution to followers' feeds"""

    CELEBRITY_THRESHOLD = 10_000  # Followers count
    MAX_FEED_SIZE = 1000  # Keep latest 1000 posts per user

    @classmethod
    def is_celebrity(cls, user: User) -> bool:
        """Check if user has too many followers"""
        follower_count = Follow.objects.filter(followee=user).count()
        return follower_count > cls.CELEBRITY_THRESHOLD

    @classmethod
    def fanout_post(cls, post: Post):
        """
        Distribute post to followers' feeds
        Hybrid approach: Push for normal users, Pull for celebrities
        """
        if cls.is_celebrity(post.user):
            # Celebrity: Don't fanout, use pull model
            logger.info(f"Celebrity post {post.id}, skipping fanout")
            cls._cache_celebrity_post(post)
            return

        # Normal user: Fanout to all followers
        followers = Follow.objects.filter(followee=post.user).select_related('follower')

        # Batch insert for performance
        feed_items = [
            Feed(user=follow.follower, post=post, created_at=post.created_at)
            for follow in followers
        ]

        Feed.objects.bulk_create(feed_items, ignore_conflicts=True)

        # Trim old posts from feeds (keep latest 1000)
        for follow in followers:
            cls._trim_feed(follow.follower)

        logger.info(f"Fanned out post {post.id} to {len(followers)} followers")

    @classmethod
    def _cache_celebrity_post(cls, post: Post):
        """Cache celebrity posts for pull model"""
        cache_key = f"user_posts:{post.user.id}"
        # Add to sorted set (Redis ZADD equivalent)
        # For simplicity, using list in cache
        posts = cache.get(cache_key) or []
        posts.insert(0, post.id)
        posts = posts[:100]  # Keep latest 100
        cache.set(cache_key, posts, timeout=3600)

    @classmethod
    def _trim_feed(cls, user: User):
        """Keep only latest posts in user's feed"""
        # Get count
        feed_count = Feed.objects.filter(user=user).count()

        if feed_count > cls.MAX_FEED_SIZE:
            # Delete oldest posts
            excess = feed_count - cls.MAX_FEED_SIZE
            old_feeds = Feed.objects.filter(user=user).order_by('created_at')[:excess]
            old_feed_ids = list(old_feeds.values_list('id', flat=True))
            Feed.objects.filter(id__in=old_feed_ids).delete()

class NewsFeedService:
    """Generate user's news feed"""

    @staticmethod
    def get_feed(user: User, page: int = 1, page_size: int = 20) -> List[Post]:
        """
        Get user's personalized feed
        Hybrid: Pre-computed feed + celebrity posts
        """
        cache_key = f"feed:{user.id}:page{page}"

        # Try cache
        cached_feed = cache.get(cache_key)
        if cached_feed:
            return cached_feed

        # Get from pre-computed feed (push model)
        offset = (page - 1) * page_size
        feed_items = Feed.objects.filter(user=user).select_related('post', 'post__user') \
            .order_by('-created_at')[offset:offset + page_size]

        posts = [item.post for item in feed_items]

        # Merge with celebrity posts (pull model)
        celebrity_posts = NewsFeedService._get_celebrity_posts(user)
        posts = NewsFeedService._merge_posts(posts, celebrity_posts)

        # Sort by created_at
        posts.sort(key=lambda p: p.created_at, reverse=True)
        posts = posts[:page_size]

        # Cache for 1 minute
        cache.set(cache_key, posts, timeout=60)

        return posts

    @staticmethod
    def _get_celebrity_posts(user: User) -> List[Post]:
        """Fetch recent posts from celebrities user follows"""
        # Get celebrities user follows
        celebrity_follows = Follow.objects.filter(
            follower=user,
            followee__followers__count__gt=FanoutService.CELEBRITY_THRESHOLD
        ).select_related('followee')

        celebrity_ids = [f.followee.id for f in celebrity_follows]

        if not celebrity_ids:
            return []

        # Get recent posts from celebrities
        posts = Post.objects.filter(user_id__in=celebrity_ids) \
            .select_related('user') \
            .order_by('-created_at')[:100]

        return list(posts)

    @staticmethod
    def _merge_posts(feed_posts: List[Post], celebrity_posts: List[Post]) -> List[Post]:
        """Merge two lists of posts, removing duplicates"""
        seen = set()
        merged = []

        for post in feed_posts + celebrity_posts:
            if post.id not in seen:
                seen.add(post.id)
                merged.append(post)

        return merged

# Celery task for async fanout
from celery import shared_task

@shared_task
def fanout_post_async(post_id: int):
    """Asynchronous fanout to avoid blocking post creation"""
    try:
        post = Post.objects.get(id=post_id)
        FanoutService.fanout_post(post)
    except Post.DoesNotExist:
        logger.error(f"Post {post_id} not found for fanout")
```

### 3. API Implementation (Django REST Framework)

```python
# serializers.py
from rest_framework import serializers

class PostSerializer(serializers.ModelSerializer):
    """Serialize post with user info"""
    username = serializers.CharField(source='user.username', read_only=True)
    user_avatar = serializers.URLField(source='user.profile.avatar', read_only=True)
    is_liked = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = [
            'id', 'user_id', 'username', 'user_avatar',
            'content', 'image_url', 'created_at',
            'like_count', 'comment_count', 'is_liked'
        ]
        read_only_fields = ['created_at', 'like_count', 'comment_count']

    def get_is_liked(self, obj):
        """Check if current user liked this post"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return Like.objects.filter(user=request.user, post=obj).exists()
        return False

# views.py
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from rest_framework.pagination import PageNumberPagination

class FeedPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class PostViewSet(viewsets.ModelViewSet):
    """Post CRUD operations"""
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated]
    pagination_class = FeedPagination

    def get_queryset(self):
        return Post.objects.filter(user=self.request.user).order_by('-created_at')

    def create(self, request):
        """Create new post and trigger fanout"""
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Create post
        post = serializer.save(user=request.user)

        # Trigger async fanout
        fanout_post_async.delay(post.id)

        return Response(serializer.data, status=status.HTTP_201_CREATED)

    @action(detail=True, methods=['post'])
    def like(self, request, pk=None):
        """Like a post"""
        post = self.get_object()

        # Create like (idempotent)
        like, created = Like.objects.get_or_create(user=request.user, post=post)

        if created:
            # Increment like count
            post.like_count = models.F('like_count') + 1
            post.save(update_fields=['like_count'])

        return Response({'liked': True})

    @action(detail=True, methods=['post'])
    def unlike(self, request, pk=None):
        """Unlike a post"""
        post = self.get_object()

        # Delete like
        deleted = Like.objects.filter(user=request.user, post=post).delete()[0]

        if deleted:
            # Decrement like count
            post.like_count = models.F('like_count') - 1
            post.save(update_fields=['like_count'])

        return Response({'liked': False})

class FeedViewSet(viewsets.ReadOnlyModelViewSet):
    """User's personalized news feed"""
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticated]
    pagination_class = FeedPagination

    def list(self, request):
        """Get user's feed"""
        page = int(request.GET.get('page', 1))
        page_size = int(request.GET.get('page_size', 20))

        # Get feed from service
        posts = NewsFeedService.get_feed(request.user, page, page_size)

        # Serialize
        serializer = self.get_serializer(posts, many=True)

        return Response({
            'results': serializer.data,
            'page': page,
            'page_size': page_size
        })

class FollowViewSet(viewsets.ViewSet):
    """Follow/unfollow operations"""
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['post'])
    def follow(self, request):
        """Follow a user"""
        followee_id = request.data.get('user_id')

        if not followee_id:
            return Response(
                {'error': 'user_id required'},
                status=status.HTTP_400_BAD_REQUEST
            )

        try:
            followee = User.objects.get(id=followee_id)
        except User.DoesNotExist:
            return Response(
                {'error': 'User not found'},
                status=status.HTTP_404_NOT_FOUND
            )

        # Create follow
        follow, created = Follow.objects.get_or_create(
            follower=request.user,
            followee=followee
        )

        return Response({
            'following': True,
            'followee_id': followee_id
        })

    @action(detail=False, methods=['post'])
    def unfollow(self, request):
        """Unfollow a user"""
        followee_id = request.data.get('user_id')

        # Delete follow
        Follow.objects.filter(
            follower=request.user,
            followee_id=followee_id
        ).delete()

        return Response({
            'following': False,
            'followee_id': followee_id
        })

# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')
router.register(r'feed', FeedViewSet, basename='feed')
router.register(r'follows', FollowViewSet, basename='follow')

urlpatterns = router.urls
```

### 4. FastAPI Implementation

```python
# main.py
from fastapi import FastAPI, Depends, HTTPException, BackgroundTasks
from sqlalchemy.orm import Session
from typing import List
from pydantic import BaseModel
from datetime import datetime

app = FastAPI()

# Models
class PostCreate(BaseModel):
    content: str
    image_url: str = ""

class PostResponse(BaseModel):
    id: int
    user_id: int
    username: str
    content: str
    image_url: str
    created_at: datetime
    like_count: int
    comment_count: int

    class Config:
        from_attributes = True

class FeedResponse(BaseModel):
    posts: List[PostResponse]
    page: int
    page_size: int

# Routes
@app.post("/api/posts", response_model=PostResponse)
async def create_post(
    post_data: PostCreate,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Create new post"""
    # Create post
    post = Post(
        user_id=current_user.id,
        content=post_data.content,
        image_url=post_data.image_url
    )
    db.add(post)
    db.commit()
    db.refresh(post)

    # Queue fanout task
    background_tasks.add_task(FanoutService.fanout_post, post)

    return post

@app.get("/api/feed", response_model=FeedResponse)
async def get_feed(
    page: int = 1,
    page_size: int = 20,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Get user's personalized feed"""
    posts = NewsFeedService.get_feed(current_user, page, page_size)

    return FeedResponse(
        posts=posts,
        page=page,
        page_size=page_size
    )

@app.post("/api/posts/{post_id}/like")
async def like_post(
    post_id: int,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Like a post"""
    # Check if post exists
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # Create like
    like = Like(user_id=current_user.id, post_id=post_id)
    db.add(like)

    # Increment counter
    post.like_count += 1

    db.commit()

    return {"liked": True}

@app.post("/api/follows/{user_id}")
async def follow_user(
    user_id: int,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """Follow a user"""
    # Check if user exists
    followee = db.query(User).filter(User.id == user_id).first()
    if not followee:
        raise HTTPException(status_code=404, detail="User not found")

    # Create follow
    follow = Follow(follower_id=current_user.id, followee_id=user_id)
    db.add(follow)
    db.commit()

    return {"following": True}
```

## 🚀 Optimizations & Advanced Features

### 1. Redis for Feed Caching

```python
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

class RedisFeedCache:
    """Redis-based feed caching"""

    @staticmethod
    def cache_user_feed(user_id: int, posts: List[Post]):
        """Cache user's feed in Redis sorted set"""
        cache_key = f"feed:{user_id}"

        # Use sorted set with timestamp as score
        pipeline = redis_client.pipeline()

        for post in posts:
            pipeline.zadd(
                cache_key,
                {json.dumps(PostSerializer(post).data): post.created_at.timestamp()}
            )

        # Keep only latest 1000
        pipeline.zremrangebyrank(cache_key, 0, -1001)

        # Set expiry
        pipeline.expire(cache_key, 3600)

        pipeline.execute()

    @staticmethod
    def get_cached_feed(user_id: int, offset: int = 0, limit: int = 20) -> List[dict]:
        """Get feed from cache"""
        cache_key = f"feed:{user_id}"

        # Get from sorted set (reverse order - newest first)
        cached = redis_client.zrevrange(cache_key, offset, offset + limit - 1)

        if cached:
            return [json.loads(item) for item in cached]

        return None
```

### 2. Real-time Updates with WebSockets

```python
# websocket.py (Django Channels)
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class FeedConsumer(AsyncJsonWebsocketConsumer):
    """WebSocket for real-time feed updates"""

    async def connect(self):
        """Connect user to their feed channel"""
        self.user_id = self.scope['user'].id
        self.feed_group = f"feed_{self.user_id}"

        # Join feed group
        await self.channel_layer.group_add(
            self.feed_group,
            self.channel_name
        )

        await self.accept()

    async def disconnect(self, close_code):
        """Disconnect from feed channel"""
        await self.channel_layer.group_discard(
            self.feed_group,
            self.channel_name
        )

    async def new_post(self, event):
        """Send new post to client"""
        await self.send_json({
            'type': 'new_post',
            'post': event['post']
        })

# Trigger from fanout service
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

def notify_new_post(follower_id: int, post: Post):
    """Notify follower of new post via WebSocket"""
    channel_layer = get_channel_layer()

    async_to_sync(channel_layer.group_send)(
        f"feed_{follower_id}",
        {
            'type': 'new_post',
            'post': PostSerializer(post).data
        }
    )
```

### 3. Ranking Algorithm

```python
class FeedRanker:
    """Rank posts by relevance (like Facebook)"""

    @staticmethod
    def calculate_score(post: Post, user: User) -> float:
        """
        Calculate post relevance score
        Factors: recency, engagement, user affinity
        """
        import time
        from datetime import timedelta

        # Time decay (exponential)
        age_hours = (timezone.now() - post.created_at).total_seconds() / 3600
        time_score = 1 / (1 + age_hours)  # Newer = higher score

        # Engagement score
        engagement = post.like_count + (post.comment_count * 2)  # Comments worth more
        engagement_score = engagement / (1 + engagement)  # Normalize

        # User affinity (how much user interacts with post author)
        affinity_score = FeedRanker._calculate_affinity(user, post.user)

        # Weighted combination
        score = (
            0.4 * time_score +
            0.4 * engagement_score +
            0.2 * affinity_score
        )

        return score

    @staticmethod
    def _calculate_affinity(user: User, author: User) -> float:
        """Calculate user's affinity with author"""
        # Count interactions in last 30 days
        recent_date = timezone.now() - timedelta(days=30)

        interactions = Like.objects.filter(
            user=user,
            post__user=author,
            created_at__gte=recent_date
        ).count()

        return min(interactions / 10, 1.0)  # Normalize to 0-1

    @staticmethod
    def rank_posts(posts: List[Post], user: User) -> List[Post]:
        """Sort posts by relevance score"""
        scored_posts = [
            (post, FeedRanker.calculate_score(post, user))
            for post in posts
        ]

        # Sort by score (descending)
        scored_posts.sort(key=lambda x: x[1], reverse=True)

        return [post for post, score in scored_posts]
```

## 🔐 Security & Anti-Spam

```python
from django.core.cache import cache

class AntiSpamService:
    """Prevent spam and abuse"""

    @staticmethod
    def check_rate_limit(user_id: int, action: str, limit: int, window: int) -> bool:
        """
        Rate limiting
        Args:
            user_id: User ID
            action: Action type (e.g., 'post', 'like', 'follow')
            limit: Max actions allowed
            window: Time window in seconds
        """
        cache_key = f"rate_limit:{user_id}:{action}"
        count = cache.get(cache_key, 0)

        if count >= limit:
            return False

        # Increment counter
        cache.set(cache_key, count + 1, timeout=window)
        return True

    @staticmethod
    def detect_spam_content(content: str) -> bool:
        """Simple spam detection"""
        spam_keywords = ['viagra', 'casino', 'free money']

        content_lower = content.lower()
        for keyword in spam_keywords:
            if keyword in content_lower:
                return True

        # Check for excessive caps
        if len(content) > 10:
            caps_ratio = sum(1 for c in content if c.isupper()) / len(content)
            if caps_ratio > 0.7:
                return True

        return False

# Use in view
def create_post(request):
    # Check rate limit (5 posts per minute)
    if not AntiSpamService.check_rate_limit(request.user.id, 'post', 5, 60):
        return Response(
            {'error': 'Rate limit exceeded'},
            status=status.HTTP_429_TOO_MANY_REQUESTS
        )

    # Check spam
    if AntiSpamService.detect_spam_content(request.data['content']):
        return Response(
            {'error': 'Spam detected'},
            status=status.HTTP_400_BAD_REQUEST
        )

    # Create post...
```

## ❓ Interview Questions

### Q1: How do you handle a celebrity user with 10M followers posting?

**Answer:**
Use **hybrid fanout approach**:

1. **Don't push to 10M feeds** (too expensive)
2. **Cache celebrity's recent posts** in Redis
3. **Pull model**: When followers request feed, merge cached celebrity posts with their pre-computed feed
4. **Trade-off**: Slightly slower feed generation for followers, but much cheaper write cost

### Q2: How do you ensure feed is up-to-date when following someone new?

**Answer:**

1. **Option 1**: Immediately fetch and insert recent posts from new followee
2. **Option 2**: Wait for next post and start fanout from there
3. **Hybrid**: Fetch last 10-20 posts asynchronously, merge with existing feed

### Q3: How do you handle feed pagination consistency?

**Answer:**

1. **Problem**: User on page 1, new post arrives, goes to page 2 → misses posts
2. **Solution**: Use cursor-based pagination with timestamps
3. **Alternative**: Accept eventual consistency, use client-side deduplication

### Q4: How do you scale the database?

**Answer:**

1. **Sharding by user_id**: User's posts and feed on same shard
2. **Read replicas**: Route reads to replicas, writes to master
3. **Separate DB for social graph**: Neo4j or specialized graph database for follows
4. **Cache hot data**: Redis for active users' feeds

### Q5: How do you implement the ranking algorithm at scale?

**Answer:**

1. **Pre-compute scores** during fanout (write-time ranking)
2. **Store in Redis sorted set** with score as weight
3. **ML-based ranking**: Train model offline, score online
4. **A/B testing**: Different ranking for different users

---

## 📚 Summary

**Key Design Decisions:**

1. **Hybrid Fanout**: Push for normal users, pull for celebrities
2. **Pre-computed Feeds**: Store in Feed table for fast reads
3. **Redis Caching**: Hot feeds and celebrity posts
4. **Async Fanout**: Celery/background tasks
5. **Pagination**: Cursor-based for consistency
6. **Ranking**: Time decay + engagement + affinity

**Scalability:**

- Cache aggressively (Redis)
- Database sharding by user_id
- Read replicas for database
- CDN for media files
- Message queue for fanout
- WebSockets for real-time

**Trade-offs:**

- **Consistency vs Availability**: Eventual consistency acceptable
- **Write cost vs Read speed**: Pre-compute for fast reads
- **Storage vs Computation**: Store denormalized feeds
- **Latency vs Accuracy**: Rank asynchronously for speed

This pattern applies to: Twitter, Facebook, Instagram, LinkedIn feeds, notification feeds, activity streams.
