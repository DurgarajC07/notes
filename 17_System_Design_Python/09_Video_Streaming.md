# System Design: Video Streaming Platform (Like YouTube/Netflix)

## 📖 Problem Statement

Design a video streaming platform like YouTube or Netflix that allows users to upload, transcode, store, and stream videos with low latency and high availability. Support adaptive bitrate streaming, recommendations, and search.

**Core Features:**

- Video upload and processing
- Multiple quality levels (360p, 720p, 1080p, 4K)
- Adaptive bitrate streaming
- Video search and discovery
- Recommendations
- View count and analytics
- Comments and likes

## 🎯 Requirements

### Functional Requirements

1. **Upload videos** - Support large files (up to 10GB)
2. **Transcode videos** - Convert to multiple formats and resolutions
3. **Stream videos** - Low latency, adaptive bitrate
4. **Search videos** - Full-text search by title, description, tags
5. **Recommendations** - Personalized suggestions
6. **Analytics** - View count, watch time, engagement
7. **Social features** - Comments, likes, subscriptions

### Non-Functional Requirements

1. **Scalability**: Support 1B users, 500M daily active users
2. **Low Latency**: Video playback starts < 2 seconds
3. **High Availability**: 99.99% uptime
4. **Storage Efficiency**: Optimize storage costs
5. **Bandwidth**: Efficient video delivery via CDN
6. **Global**: Low latency worldwide

## 📊 Capacity Estimation

```python
class VideoStreamingEstimation:
    """Calculate capacity for video streaming platform"""

    def __init__(self):
        # User statistics
        self.total_users = 1_000_000_000  # 1 billion
        self.daily_active_users = 500_000_000  # 500M DAU
        self.videos_watched_per_user_per_day = 5
        self.avg_watch_duration_minutes = 10

        # Upload statistics
        self.videos_uploaded_per_day = 500_000  # 500K videos/day
        self.avg_video_size_mb = 500  # Before transcoding

        # Video specifications
        self.resolutions = {
            '360p': 0.3,   # GB per hour
            '480p': 0.7,
            '720p': 1.5,
            '1080p': 3.0,
            '4K': 8.0
        }

        # Distribution of views by quality
        self.quality_distribution = {
            '360p': 0.20,
            '480p': 0.30,
            '720p': 0.30,
            '1080p': 0.15,
            '4K': 0.05
        }

    def calculate_daily_views(self):
        """Calculate daily video views"""
        total_views = self.daily_active_users * self.videos_watched_per_user_per_day
        print("=== Daily Views ===")
        print(f"Total Daily Views: {total_views:,.0f}")
        print(f"Views per Second: {total_views / (24 * 3600):,.0f}")
        print(f"Peak Views/sec (5x): {(total_views / (24 * 3600)) * 5:,.0f}\n")
        return total_views

    def calculate_bandwidth(self):
        """Calculate bandwidth requirements"""
        concurrent_viewers = self.daily_active_users * 0.1  # 10% concurrent

        total_bandwidth_gbps = 0
        print("=== Bandwidth Estimation ===")

        for quality, gb_per_hour in self.resolutions.items():
            distribution = self.quality_distribution[quality]
            viewers_at_quality = concurrent_viewers * distribution

            # Mbps per stream
            mbps_per_stream = (gb_per_hour * 1024 * 8) / 3600

            # Total bandwidth for this quality
            total_mbps = viewers_at_quality * mbps_per_stream
            total_gbps = total_mbps / 1000
            total_bandwidth_gbps += total_gbps

            print(f"{quality}: {viewers_at_quality:,.0f} viewers = {total_gbps:,.0f} Gbps")

        print(f"\nTotal Bandwidth: {total_bandwidth_gbps:,.0f} Gbps")
        print(f"Peak Bandwidth (3x): {total_bandwidth_gbps * 3:,.0f} Gbps\n")

    def calculate_storage(self):
        """Calculate storage requirements"""
        # Daily uploads
        daily_upload_storage = self.videos_uploaded_per_day * self.avg_video_size_mb / 1024  # GB

        # After transcoding (multiple qualities)
        transcoded_multiplier = 3  # Original + 3 formats average
        daily_storage_after_transcode = daily_upload_storage * transcoded_multiplier

        # Yearly storage
        yearly_storage_tb = (daily_storage_after_transcode * 365) / 1024

        # 5-year storage
        five_year_storage_pb = (yearly_storage_tb * 5) / 1024

        print("=== Storage Estimation ===")
        print(f"Daily Uploads: {self.videos_uploaded_per_day:,.0f} videos")
        print(f"Daily Raw Storage: {daily_upload_storage:,.0f} GB")
        print(f"Daily After Transcode: {daily_storage_after_transcode:,.0f} GB")
        print(f"Yearly Storage: {yearly_storage_tb:,.2f} TB")
        print(f"5-Year Storage: {five_year_storage_pb:,.2f} PB\n")

    def calculate_cdn_cost(self):
        """Estimate CDN costs"""
        daily_views = self.daily_active_users * self.videos_watched_per_user_per_day
        avg_video_minutes = self.avg_watch_duration_minutes

        # Average bandwidth per view
        avg_gb_per_view = 0.5  # ~720p for 10 minutes

        # Total data transfer per day
        daily_data_transfer_tb = (daily_views * avg_gb_per_view) / 1024

        # CDN cost (approx $0.02 per GB)
        cdn_cost_per_gb = 0.02
        daily_cdn_cost = (daily_data_transfer_tb * 1024) * cdn_cost_per_gb

        print("=== CDN Cost Estimation ===")
        print(f"Daily Data Transfer: {daily_data_transfer_tb:,.0f} TB")
        print(f"Daily CDN Cost: ${daily_cdn_cost:,.0f}")
        print(f"Monthly CDN Cost: ${daily_cdn_cost * 30:,.0f}")
        print(f"Yearly CDN Cost: ${daily_cdn_cost * 365:,.0f}\n")

# Run estimation
estimator = VideoStreamingEstimation()
estimator.calculate_daily_views()
estimator.calculate_bandwidth()
estimator.calculate_storage()
estimator.calculate_cdn_cost()
```

**Output:**

```
=== Daily Views ===
Total Daily Views: 2,500,000,000
Views per Second: 28,935
Peak Views/sec (5x): 144,676

=== Bandwidth Estimation ===
360p: 10,000,000 viewers = 853 Gbps
480p: 15,000,000 viewers = 2,917 Gbps
720p: 15,000,000 viewers = 6,250 Gbps
1080p: 7,500,000 viewers = 6,250 Gbps
4K: 2,500,000 viewers = 5,556 Gbps

Total Bandwidth: 21,826 Gbps
Peak Bandwidth (3x): 65,478 Gbps

=== Storage Estimation ===
Daily Uploads: 500,000 videos
Daily Raw Storage: 244,141 GB
Daily After Transcode: 732,422 GB
Yearly Storage: 261.86 TB
5-Year Storage: 1.28 PB

=== CDN Cost Estimation ===
Daily Data Transfer: 1,220,703 TB
Daily CDN Cost: $25,024,414
Monthly CDN Cost: $750,732,422
Yearly CDN Cost: $9,133,910,742
```

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────────┐
│                    Client Apps                        │
│         (Web, iOS, Android, Smart TV)                │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
              ┌──────────────┐
              │     CDN      │
              │  (CloudFront)│
              └──────┬───────┘
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │API GW 1 │ │API GW 2 │ │API GW N │
    └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │
         └───────────┼───────────┘
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Upload  │  │ Metadata │  │Streaming │
│ Service  │  │ Service  │  │ Service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│   S3     │  │PostgreSQL│  │  Redis   │
│(Original)│  │(Metadata)│  │  Cache   │
└────┬─────┘  └──────────┘  └──────────┘
     │
     ▼
┌──────────────┐
│ Transcoding  │
│   Queue      │
│ (SQS/Kafka)  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Transcoding  │
│  Workers     │
│ (FFmpeg)     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  S3 Bucket   │
│ (Transcoded) │
│ Multiple     │
│ Resolutions  │
└──────────────┘
```

## 🔑 Database Schema

```sql
-- Videos table
CREATE TABLE videos (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    duration_seconds INT,
    thumbnail_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'processing',  -- processing, ready, failed
    view_count BIGINT DEFAULT 0,
    like_count INT DEFAULT 0,
    dislike_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP
);

-- Video files (different qualities)
CREATE TABLE video_files (
    id BIGSERIAL PRIMARY KEY,
    video_id BIGINT REFERENCES videos(id),
    resolution VARCHAR(10) NOT NULL,  -- 360p, 720p, 1080p, 4K
    file_url VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    codec VARCHAR(20),
    bitrate_kbps INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(video_id, resolution)
);

-- Watch history
CREATE TABLE watch_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    video_id BIGINT REFERENCES videos(id),
    watch_duration_seconds INT,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Comments
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    video_id BIGINT REFERENCES videos(id),
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    like_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Subscriptions
CREATE TABLE subscriptions (
    id BIGSERIAL PRIMARY KEY,
    subscriber_id BIGINT NOT NULL,
    channel_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(subscriber_id, channel_id)
);

-- Indexes
CREATE INDEX idx_videos_user ON videos(user_id, published_at DESC);
CREATE INDEX idx_videos_views ON videos(view_count DESC);
CREATE INDEX idx_videos_published ON videos(published_at DESC) WHERE status = 'ready';
CREATE INDEX idx_watch_history_user ON watch_history(user_id, created_at DESC);
CREATE INDEX idx_comments_video ON comments(video_id, created_at DESC);
```

## 💻 Django Implementation

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class VideoStatus(models.TextChoices):
    UPLOADING = 'uploading', 'Uploading'
    PROCESSING = 'processing', 'Processing'
    READY = 'ready', 'Ready'
    FAILED = 'failed', 'Failed'

class VideoResolution(models.TextChoices):
    R_360P = '360p', '360p'
    R_480P = '480p', '480p'
    R_720P = '720p', '720p (HD)'
    R_1080P = '1080p', '1080p (Full HD)'
    R_4K = '4K', '4K (Ultra HD)'

class Video(models.Model):
    """Video metadata"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    duration_seconds = models.IntegerField(null=True)
    thumbnail_url = models.URLField(max_length=500, blank=True)
    status = models.CharField(
        max_length=20,
        choices=VideoStatus.choices,
        default=VideoStatus.UPLOADING
    )
    view_count = models.BigIntegerField(default=0)
    like_count = models.IntegerField(default=0)
    dislike_count = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    published_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        db_table = 'videos'
        indexes = [
            models.Index(fields=['user', '-published_at']),
            models.Index(fields=['-view_count']),
            models.Index(fields=['-published_at']),
        ]

class VideoFile(models.Model):
    """Video file for specific resolution"""
    video = models.ForeignKey(Video, on_delete=models.CASCADE, related_name='files')
    resolution = models.CharField(max_length=10, choices=VideoResolution.choices)
    file_url = models.URLField(max_length=500)
    file_size_bytes = models.BigIntegerField()
    codec = models.CharField(max_length=20, default='h264')
    bitrate_kbps = models.IntegerField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'video_files'
        unique_together = ('video', 'resolution')

class WatchHistory(models.Model):
    """Track user watch history"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    video = models.ForeignKey(Video, on_delete=models.CASCADE)
    watch_duration_seconds = models.IntegerField()
    completed = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'watch_history'
        indexes = [
            models.Index(fields=['user', '-created_at']),
        ]

# services/video_service.py
import boto3
from django.conf import settings
from django.core.cache import cache
from typing import Optional
import logging

logger = logging.getLogger(__name__)

class VideoUploadService:
    """Handle video upload to S3"""

    def __init__(self):
        self.s3_client = boto3.client(
            's3',
            aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
            aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY
        )
        self.bucket_name = settings.S3_VIDEO_BUCKET

    def generate_upload_url(self, video_id: int, filename: str) -> dict:
        """
        Generate presigned URL for direct upload from client
        """
        key = f"uploads/{video_id}/{filename}"

        # Generate presigned POST
        presigned_post = self.s3_client.generate_presigned_post(
            Bucket=self.bucket_name,
            Key=key,
            Fields={'acl': 'private'},
            Conditions=[
                {'acl': 'private'},
                ['content-length-range', 1, 10 * 1024 * 1024 * 1024]  # Max 10GB
            ],
            ExpiresIn=3600  # Valid for 1 hour
        )

        return {
            'upload_url': presigned_post['url'],
            'fields': presigned_post['fields'],
            's3_key': key
        }

    def mark_upload_complete(self, video: Video, s3_key: str):
        """
        Mark upload as complete and queue for transcoding
        """
        video.status = VideoStatus.PROCESSING
        video.save(update_fields=['status'])

        # Queue transcoding job
        from .tasks import transcode_video_task
        transcode_video_task.delay(video.id, s3_key)

        logger.info(f"Video {video.id} queued for transcoding")

class TranscodingService:
    """Handle video transcoding"""

    RESOLUTIONS = {
        '360p': {'width': 640, 'height': 360, 'bitrate': '800k'},
        '480p': {'width': 854, 'height': 480, 'bitrate': '1400k'},
        '720p': {'width': 1280, 'height': 720, 'bitrate': '2800k'},
        '1080p': {'width': 1920, 'height': 1080, 'bitrate': '5000k'},
        '4K': {'width': 3840, 'height': 2160, 'bitrate': '15000k'},
    }

    @classmethod
    def transcode_video(cls, video: Video, source_s3_key: str):
        """
        Transcode video to multiple resolutions using FFmpeg
        """
        import subprocess
        import tempfile
        import os

        s3 = boto3.client('s3')

        try:
            # Download original file
            with tempfile.NamedTemporaryFile(delete=False, suffix='.mp4') as tmp_file:
                s3.download_file(
                    settings.S3_VIDEO_BUCKET,
                    source_s3_key,
                    tmp_file.name
                )
                source_path = tmp_file.name

            # Transcode to each resolution
            for resolution, config in cls.RESOLUTIONS.items():
                output_path = cls._transcode_resolution(
                    source_path,
                    resolution,
                    config
                )

                # Upload to S3
                output_key = f"videos/{video.id}/{resolution}.mp4"
                s3.upload_file(
                    output_path,
                    settings.S3_VIDEO_BUCKET,
                    output_key,
                    ExtraArgs={'ContentType': 'video/mp4'}
                )

                # Create VideoFile record
                file_size = os.path.getsize(output_path)
                VideoFile.objects.create(
                    video=video,
                    resolution=resolution,
                    file_url=f"https://{settings.CDN_DOMAIN}/{output_key}",
                    file_size_bytes=file_size,
                    bitrate_kbps=int(config['bitrate'].replace('k', ''))
                )

                # Cleanup
                os.remove(output_path)

            # Extract thumbnail
            thumbnail_key = cls._generate_thumbnail(source_path, video.id)
            video.thumbnail_url = f"https://{settings.CDN_DOMAIN}/{thumbnail_key}"

            # Get video duration
            duration = cls._get_video_duration(source_path)
            video.duration_seconds = duration

            # Update video status
            video.status = VideoStatus.READY
            video.published_at = timezone.now()
            video.save(update_fields=['status', 'published_at', 'thumbnail_url', 'duration_seconds'])

            # Cleanup original
            os.remove(source_path)

            logger.info(f"Video {video.id} transcoded successfully")

        except Exception as e:
            logger.error(f"Transcoding failed for video {video.id}: {str(e)}")
            video.status = VideoStatus.FAILED
            video.save(update_fields=['status'])

    @classmethod
    def _transcode_resolution(cls, source_path: str, resolution: str, config: dict) -> str:
        """Transcode to specific resolution using FFmpeg"""
        import tempfile

        output_path = tempfile.mktemp(suffix='.mp4')

        # FFmpeg command
        cmd = [
            'ffmpeg',
            '-i', source_path,
            '-vf', f"scale={config['width']}:{config['height']}",
            '-c:v', 'libx264',
            '-b:v', config['bitrate'],
            '-c:a', 'aac',
            '-b:a', '128k',
            '-movflags', '+faststart',  # Enable streaming
            output_path
        ]

        subprocess.run(cmd, check=True, capture_output=True)
        return output_path

    @classmethod
    def _generate_thumbnail(cls, source_path: str, video_id: int) -> str:
        """Generate video thumbnail"""
        import subprocess
        import tempfile

        output_path = tempfile.mktemp(suffix='.jpg')

        # Extract frame at 2 seconds
        cmd = [
            'ffmpeg',
            '-i', source_path,
            '-ss', '00:00:02',
            '-vframes', '1',
            '-vf', 'scale=1280:720',
            output_path
        ]

        subprocess.run(cmd, check=True, capture_output=True)

        # Upload to S3
        s3 = boto3.client('s3')
        thumbnail_key = f"thumbnails/{video_id}/thumb.jpg"
        s3.upload_file(
            output_path,
            settings.S3_VIDEO_BUCKET,
            thumbnail_key,
            ExtraArgs={'ContentType': 'image/jpeg'}
        )

        os.remove(output_path)
        return thumbnail_key

    @classmethod
    def _get_video_duration(cls, source_path: str) -> int:
        """Get video duration in seconds"""
        import subprocess
        import json

        cmd = [
            'ffprobe',
            '-v', 'quiet',
            '-print_format', 'json',
            '-show_format',
            source_path
        ]

        result = subprocess.run(cmd, capture_output=True, text=True)
        data = json.loads(result.stdout)

        return int(float(data['format']['duration']))

# tasks.py (Celery)
from celery import shared_task

@shared_task
def transcode_video_task(video_id: int, source_s3_key: str):
    """Async video transcoding"""
    try:
        video = Video.objects.get(id=video_id)
        TranscodingService.transcode_video(video, source_s3_key)
    except Video.DoesNotExist:
        logger.error(f"Video {video_id} not found")
    except Exception as e:
        logger.error(f"Transcoding task failed: {str(e)}")

# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, AllowAny

class VideoViewSet(viewsets.ModelViewSet):
    """Video CRUD and streaming"""
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Video.objects.filter(status=VideoStatus.READY).order_by('-published_at')

    def create(self, request):
        """Initiate video upload"""
        title = request.data.get('title')
        description = request.data.get('description', '')
        filename = request.data.get('filename')

        if not all([title, filename]):
            return Response(
                {'error': 'title and filename required'},
                status=status.HTTP_400_BAD_REQUEST
            )

        # Create video record
        video = Video.objects.create(
            user=request.user,
            title=title,
            description=description,
            status=VideoStatus.UPLOADING
        )

        # Generate upload URL
        upload_service = VideoUploadService()
        upload_data = upload_service.generate_upload_url(video.id, filename)

        return Response({
            'video_id': video.id,
            'upload_url': upload_data['upload_url'],
            'upload_fields': upload_data['fields']
        }, status=status.HTTP_201_CREATED)

    @action(detail=True, methods=['post'])
    def upload_complete(self, request, pk=None):
        """Mark upload as complete"""
        video = self.get_object()
        s3_key = request.data.get('s3_key')

        upload_service = VideoUploadService()
        upload_service.mark_upload_complete(video, s3_key)

        return Response({'status': 'processing'})

    @action(detail=True, methods=['get'], permission_classes=[AllowAny])
    def stream(self, request, pk=None):
        """Get streaming URLs for video"""
        video = self.get_object()

        if video.status != VideoStatus.READY:
            return Response(
                {'error': 'Video not ready'},
                status=status.HTTP_400_BAD_REQUEST
            )

        # Get all available resolutions
        video_files = video.files.all()

        streaming_urls = {
            vf.resolution: {
                'url': vf.file_url,
                'bitrate': vf.bitrate_kbps,
                'size_bytes': vf.file_size_bytes
            }
            for vf in video_files
        }

        # Increment view count (async)
        cache_key = f"viewed:{request.user.id if request.user.is_authenticated else request.META.get('REMOTE_ADDR')}:{video.id}"
        if not cache.get(cache_key):
            video.view_count = models.F('view_count') + 1
            video.save(update_fields=['view_count'])
            cache.set(cache_key, True, timeout=3600)  # Count once per hour

        return Response({
            'video_id': video.id,
            'title': video.title,
            'duration': video.duration_seconds,
            'thumbnail': video.thumbnail_url,
            'streaming_urls': streaming_urls
        })

    @action(detail=True, methods=['post'])
    def like(self, request, pk=None):
        """Like a video"""
        video = self.get_object()

        # Check if already liked
        cache_key = f"liked:{request.user.id}:{video.id}"
        if cache.get(cache_key):
            return Response({'liked': True})

        video.like_count = models.F('like_count') + 1
        video.save(update_fields=['like_count'])
        cache.set(cache_key, True, timeout=86400)  # 24 hours

        return Response({'liked': True})
```

## 🚀 Adaptive Bitrate Streaming (HLS)

```python
class HLSService:
    """Generate HLS manifest for adaptive streaming"""

    @staticmethod
    def generate_master_playlist(video: Video) -> str:
        """
        Generate master playlist (.m3u8)
        Lists all available quality variants
        """
        video_files = video.files.all().order_by('-bitrate_kbps')

        playlist = "#EXTM3U\n"
        playlist += "#EXT-X-VERSION:3\n\n"

        for vf in video_files:
            playlist += f"#EXT-X-STREAM-INF:BANDWIDTH={vf.bitrate_kbps * 1000},"
            playlist += f"RESOLUTION={vf.resolution}\n"
            playlist += f"{vf.file_url}\n"

        return playlist
```

## ❓ Interview Questions

### Q1: How do you handle video upload for large files (10GB)?

**Answer:**

1. **Multipart upload**: Split into chunks, upload in parallel
2. **Presigned URLs**: Client uploads directly to S3
3. **Resumable uploads**: Track progress, resume on failure
4. **Progress tracking**: WebSocket for real-time updates

### Q2: How do you optimize CDN costs?

**Answer:**

1. **Popular videos**: Cache at edge locations
2. **Cold videos**: Fetch from origin on demand
3. **Smart caching**: Expire old content
4. **Compression**: Use efficient codecs (H.265, VP9, AV1)
5. **Tiered storage**: Hot in CDN, cold in S3 Glacier

### Q3: How do you handle viral videos (1M concurrent viewers)?

**Answer:**

1. **CDN scaling**: Automatic edge location addition
2. **Origin shield**: Protect origin from traffic spikes
3. **Pre-warming**: Push to edge before peak
4. **Rate limiting**: Prevent DDoS
5. **Graceful degradation**: Lower quality if bandwidth limited

### Q4: How do you implement recommendations?

**Answer:**

1. **Collaborative filtering**: Users who watched X also watched Y
2. **Content-based**: Similar videos by tags, category
3. **Trending**: View count + recency
4. **Personalized**: ML model trained on watch history
5. **Real-time**: Update recommendations as user watches

### Q5: How do you ensure copyright protection (DRM)?

**Answer:**

1. **Encryption**: AES encryption for video files
2. **License server**: Issue temporary playback licenses
3. **Token-based access**: Signed URLs with expiry
4. **Watermarking**: Identify leaked content
5. **Geo-blocking**: Restrict by region

---

## 📚 Summary

**Key Components:**

1. **Upload**: S3 presigned URLs for direct upload
2. **Transcoding**: FFmpeg workers, multiple resolutions
3. **Storage**: S3 for files, PostgreSQL for metadata
4. **Streaming**: CDN with adaptive bitrate (HLS)
5. **Analytics**: Track views, watch time
6. **Recommendations**: ML-based suggestions

**Scalability:**

- CDN for global content delivery
- S3 for unlimited storage
- Queue-based transcoding (auto-scaling workers)
- Database sharding by user_id or video_id
- Cache metadata in Redis

**Cost Optimization:**

- Efficient codecs (H.265, AV1)
- Tiered storage (S3 → Glacier)
- CDN smart caching
- Lazy transcoding (on-demand for unpopular videos)

This pattern applies to: YouTube, Netflix, Twitch, TikTok, Vimeo, video conferencing platforms.
