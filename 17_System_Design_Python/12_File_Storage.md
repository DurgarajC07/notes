# System Design: Distributed File Storage (Like Dropbox/Google Drive)

## 📖 Problem Statement

Design a distributed file storage and synchronization system like Dropbox or Google Drive that allows users to upload, store, sync, and share files across multiple devices with automatic conflict resolution.

**Core Features:**

- Upload and download files
- Sync across devices
- File sharing and permissions
- Version history
- Offline support
- Conflict resolution

## 🎯 Requirements

### Functional Requirements

1. **Upload/Download files** - Support files up to 50GB
2. **Sync across devices** - Real-time synchronization
3. **File sharing** - Share with users or public links
4. **Version history** - Track file changes, rollback
5. **Offline support** - Work offline, sync when online
6. **Conflict resolution** - Handle concurrent edits
7. **Search** - Find files by name, content

### Non-Functional Requirements

1. **Reliability**: 99.99% uptime, no data loss
2. **Consistency**: Strong consistency for metadata, eventual for files
3. **Scalability**: Support 500M users, 100M daily active
4. **Performance**: Fast sync (< 1 second for small files)
5. **Storage Efficiency**: Deduplication, compression
6. **Security**: Encryption at rest and in transit

## 📊 Capacity Estimation

```python
class FileStorageEstimation:
    """Calculate capacity for file storage system"""

    def __init__(self):
        self.total_users = 500_000_000  # 500M users
        self.daily_active_users = 100_000_000  # 100M DAU
        self.avg_files_per_user = 200
        self.avg_file_size_mb = 2  # Average file size
        self.files_synced_per_user_per_day = 10
        self.connections_per_user = 3  # Desktop, phone, tablet

    def calculate_storage(self):
        """Calculate total storage needed"""
        total_files = self.total_users * self.avg_files_per_user
        total_storage_tb = (total_files * self.avg_file_size_mb) / (1024 * 1024)

        # With deduplication (40% savings)
        deduplicated_storage_tb = total_storage_tb * 0.6

        print("=== Storage Estimation ===")
        print(f"Total Files: {total_files:,.0f}")
        print(f"Raw Storage: {total_storage_tb:,.2f} TB")
        print(f"After Deduplication: {deduplicated_storage_tb:,.2f} TB")
        print(f"Storage in PB: {deduplicated_storage_tb / 1024:.2f} PB\n")

    def calculate_bandwidth(self):
        """Calculate bandwidth requirements"""
        # Files synced per day
        daily_syncs = self.daily_active_users * self.files_synced_per_user_per_day
        sync_qps = daily_syncs / (24 * 3600)

        # Bandwidth (assuming average 2MB per sync)
        bandwidth_mbps = (sync_qps * self.avg_file_size_mb * 8)
        bandwidth_gbps = bandwidth_mbps / 1000

        print("=== Bandwidth Estimation ===")
        print(f"Daily Syncs: {daily_syncs:,.0f}")
        print(f"Sync QPS: {sync_qps:,.0f}")
        print(f"Bandwidth: {bandwidth_gbps:,.2f} Gbps")
        print(f"Peak Bandwidth (3x): {bandwidth_gbps * 3:,.2f} Gbps\n")

    def calculate_metadata_db(self):
        """Calculate metadata database size"""
        # Metadata per file: ~1KB (path, size, hash, timestamps)
        total_files = self.total_users * self.avg_files_per_user
        metadata_size_gb = (total_files * 1) / (1024 * 1024)

        print("=== Metadata Database ===")
        print(f"Total Metadata: {metadata_size_gb:,.2f} GB")
        print(f"Recommended DB: Sharded PostgreSQL or MongoDB\n")

    def calculate_sync_connections(self):
        """Calculate concurrent connections"""
        concurrent_users = self.daily_active_users * 0.2  # 20% concurrent
        total_connections = concurrent_users * self.connections_per_user

        print("=== Sync Connections ===")
        print(f"Concurrent Users: {concurrent_users:,.0f}")
        print(f"Total Connections: {total_connections:,.0f}")
        print(f"WebSocket servers needed: {total_connections / 50000:.0f} (50K connections/server)\n")

# Run estimation
estimator = FileStorageEstimation()
estimator.calculate_storage()
estimator.calculate_bandwidth()
estimator.calculate_metadata_db()
estimator.calculate_sync_connections()
```

**Output:**

```
=== Storage Estimation ===
Total Files: 100,000,000,000
Raw Storage: 190,734.86 TB
After Deduplication: 114,440.92 TB
Storage in PB: 111.76 PB

=== Bandwidth Estimation ===
Daily Syncs: 1,000,000,000
Sync QPS: 11,574
Bandwidth: 185.19 Gbps
Peak Bandwidth (3x): 555.56 Gbps

=== Metadata Database ===
Total Metadata: 95,367.43 GB
Recommended DB: Sharded PostgreSQL or MongoDB

=== Sync Connections ===
Concurrent Users: 20,000,000
Total Connections: 60,000,000
WebSocket servers needed: 1200 (50K connections/server)
```

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────┐
│          Client Applications                      │
│   (Desktop, Mobile, Web)                         │
└────────────────┬─────────────────────────────────┘
                 │
                 ▼
          ┌─────────────┐
          │ Load Balancer│
          └──────┬───────┘
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ API      │ │ Sync     │ │ Metadata │
│ Server   │ │ Server   │ │ Service  │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Metadata │ │ Block    │ │  Object  │
│ Database │ │ Service  │ │ Storage  │
│(Postgres)│ │          │ │   (S3)   │
└──────────┘ └────┬─────┘ └──────────┘
                  │
                  ▼
           ┌──────────────┐
           │ Message Queue│
           │  (Kafka)     │
           └──────┬───────┘
                  │
                  ▼
           ┌──────────────┐
           │ Notification │
           │   Service    │
           └──────────────┘
```

## 🔑 Core Design Decisions

### 1. Chunking Strategy

Split files into chunks (4MB each) for:

- **Efficient uploads**: Only upload changed chunks
- **Resume capability**: Continue interrupted uploads
- **Deduplication**: Same chunk across different files
- **Bandwidth optimization**: Parallel chunk uploads

```python
import hashlib
import os

class ChunkingService:
    """Handle file chunking and deduplication"""

    CHUNK_SIZE = 4 * 1024 * 1024  # 4MB chunks

    @staticmethod
    def split_file(file_path: str) -> list:
        """
        Split file into chunks and calculate hashes
        Returns list of (chunk_index, chunk_hash, chunk_size)
        """
        chunks = []
        chunk_index = 0

        with open(file_path, 'rb') as f:
            while True:
                chunk_data = f.read(ChunkingService.CHUNK_SIZE)
                if not chunk_data:
                    break

                # Calculate SHA256 hash
                chunk_hash = hashlib.sha256(chunk_data).hexdigest()

                chunks.append({
                    'index': chunk_index,
                    'hash': chunk_hash,
                    'size': len(chunk_data),
                    'data': chunk_data
                })

                chunk_index += 1

        return chunks

    @staticmethod
    def calculate_file_hash(chunks: list) -> str:
        """Calculate overall file hash from chunk hashes"""
        combined_hashes = ''.join([c['hash'] for c in chunks])
        return hashlib.sha256(combined_hashes.encode()).hexdigest()

    @staticmethod
    def identify_changed_chunks(old_chunks: list, new_chunks: list) -> list:
        """
        Compare chunk lists and identify what changed
        Returns list of chunks that need to be uploaded
        """
        old_hashes = {c['hash'] for c in old_chunks}

        changed_chunks = [
            chunk for chunk in new_chunks
            if chunk['hash'] not in old_hashes
        ]

        return changed_chunks
```

### 2. Database Schema

```sql
-- Users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    storage_quota_bytes BIGINT DEFAULT 16106127360,  -- 15GB
    storage_used_bytes BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Files table (metadata)
CREATE TABLE files (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    parent_id BIGINT REFERENCES files(id),  -- For folders
    name VARCHAR(255) NOT NULL,
    path TEXT NOT NULL,
    is_folder BOOLEAN DEFAULT FALSE,
    file_size_bytes BIGINT,
    file_hash VARCHAR(64),  -- SHA256
    mime_type VARCHAR(100),
    version INT DEFAULT 1,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP
);

-- File chunks
CREATE TABLE file_chunks (
    id BIGSERIAL PRIMARY KEY,
    file_id BIGINT REFERENCES files(id),
    chunk_index INT NOT NULL,
    chunk_hash VARCHAR(64) NOT NULL,
    chunk_size INT NOT NULL,
    storage_key VARCHAR(500) NOT NULL,  -- S3 key
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(file_id, chunk_index)
);

-- Chunk deduplication
CREATE TABLE chunks (
    chunk_hash VARCHAR(64) PRIMARY KEY,
    storage_key VARCHAR(500) NOT NULL,
    chunk_size INT NOT NULL,
    ref_count INT DEFAULT 0,  -- How many files reference this
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- File versions
CREATE TABLE file_versions (
    id BIGSERIAL PRIMARY KEY,
    file_id BIGINT REFERENCES files(id),
    version INT NOT NULL,
    file_size_bytes BIGINT,
    file_hash VARCHAR(64),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT REFERENCES users(id),
    UNIQUE(file_id, version)
);

-- File shares
CREATE TABLE file_shares (
    id BIGSERIAL PRIMARY KEY,
    file_id BIGINT REFERENCES files(id),
    shared_by BIGINT REFERENCES users(id),
    shared_with BIGINT REFERENCES users(id),  -- NULL for public links
    permission VARCHAR(20) DEFAULT 'read',  -- read, write, admin
    share_token VARCHAR(100) UNIQUE,  -- For public links
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Device sync state
CREATE TABLE device_sync (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    device_id VARCHAR(100) NOT NULL,
    last_sync_timestamp BIGINT,  -- Unix timestamp in milliseconds
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, device_id)
);

-- Indexes
CREATE INDEX idx_files_user ON files(user_id, is_deleted);
CREATE INDEX idx_files_parent ON files(parent_id) WHERE is_folder = FALSE;
CREATE INDEX idx_files_path ON files(path, user_id);
CREATE INDEX idx_chunks_hash ON chunks(chunk_hash);
CREATE INDEX idx_file_shares_token ON file_shares(share_token);
```

### 3. Django Implementation

```python
# models.py
from django.db import models
from django.contrib.auth.models import User
import hashlib

class File(models.Model):
    """File/folder metadata"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    parent = models.ForeignKey('self', on_delete=models.CASCADE, null=True, blank=True)
    name = models.CharField(max_length=255)
    path = models.TextField()  # Full path for easy querying
    is_folder = models.BooleanField(default=False)
    file_size_bytes = models.BigIntegerField(null=True)
    file_hash = models.CharField(max_length=64, blank=True)  # SHA256
    mime_type = models.CharField(max_length=100, blank=True)
    version = models.IntegerField(default=1)
    is_deleted = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        db_table = 'files'
        indexes = [
            models.Index(fields=['user', 'is_deleted']),
            models.Index(fields=['parent']),
        ]
        unique_together = ('user', 'path')

class Chunk(models.Model):
    """Deduplicated chunks"""
    chunk_hash = models.CharField(max_length=64, primary_key=True)
    storage_key = models.CharField(max_length=500)
    chunk_size = models.IntegerField()
    ref_count = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'chunks'

class FileChunk(models.Model):
    """File to chunk mapping"""
    file = models.ForeignKey(File, on_delete=models.CASCADE, related_name='chunks')
    chunk_index = models.IntegerField()
    chunk_hash = models.CharField(max_length=64)
    chunk_size = models.IntegerField()
    storage_key = models.CharField(max_length=500)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'file_chunks'
        unique_together = ('file', 'chunk_index')

class FileVersion(models.Model):
    """File version history"""
    file = models.ForeignKey(File, on_delete=models.CASCADE, related_name='versions')
    version = models.IntegerField()
    file_size_bytes = models.BigIntegerField()
    file_hash = models.CharField(max_length=64)
    created_at = models.DateTimeField(auto_now_add=True)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)

    class Meta:
        db_table = 'file_versions'
        unique_together = ('file', 'version')

class FileShare(models.Model):
    """File sharing"""
    file = models.ForeignKey(File, on_delete=models.CASCADE)
    shared_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name='shares_created')
    shared_with = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True, related_name='shares_received')
    permission = models.CharField(max_length=20, default='read')
    share_token = models.CharField(max_length=100, unique=True, blank=True)
    expires_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'file_shares'

class DeviceSync(models.Model):
    """Track sync state per device"""
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    device_id = models.CharField(max_length=100)
    last_sync_timestamp = models.BigIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'device_sync'
        unique_together = ('user', 'device_id')

# services/file_service.py
import boto3
from django.conf import settings
from django.db import transaction
from typing import List, Dict
import logging

logger = logging.getLogger(__name__)

class FileUploadService:
    """Handle file uploads with chunking"""

    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.bucket_name = settings.S3_BUCKET

    def initiate_upload(self, user: User, file_path: str, file_size: int, chunks: List[Dict]) -> File:
        """
        Initiate file upload
        Returns File object and list of chunks that need uploading
        """
        # Check existing file
        file_hash = ChunkingService.calculate_file_hash(chunks)

        # Check if file already exists (deduplication)
        existing_file = File.objects.filter(
            user=user,
            path=file_path,
            is_deleted=False
        ).first()

        if existing_file and existing_file.file_hash == file_hash:
            # File hasn't changed
            return existing_file, []

        # Create or update file record
        with transaction.atomic():
            file_obj, created = File.objects.update_or_create(
                user=user,
                path=file_path,
                defaults={
                    'name': os.path.basename(file_path),
                    'file_size_bytes': file_size,
                    'file_hash': file_hash,
                    'is_folder': False
                }
            )

            if not created:
                # New version
                file_obj.version += 1
                file_obj.save(update_fields=['version'])

        # Identify chunks that need uploading
        chunks_to_upload = []

        for chunk in chunks:
            chunk_hash = chunk['hash']

            # Check if chunk already exists (deduplication)
            existing_chunk = Chunk.objects.filter(chunk_hash=chunk_hash).first()

            if existing_chunk:
                # Chunk exists, just reference it
                FileChunk.objects.create(
                    file=file_obj,
                    chunk_index=chunk['index'],
                    chunk_hash=chunk_hash,
                    chunk_size=chunk['size'],
                    storage_key=existing_chunk.storage_key
                )

                # Increment ref count
                existing_chunk.ref_count += 1
                existing_chunk.save(update_fields=['ref_count'])
            else:
                # Need to upload this chunk
                chunks_to_upload.append(chunk)

        return file_obj, chunks_to_upload

    def upload_chunk(self, file_id: int, chunk: Dict) -> str:
        """Upload chunk to S3"""
        storage_key = f"chunks/{chunk['hash']}"

        # Upload to S3
        self.s3_client.put_object(
            Bucket=self.bucket_name,
            Key=storage_key,
            Body=chunk['data']
        )

        # Create Chunk record
        chunk_obj, created = Chunk.objects.get_or_create(
            chunk_hash=chunk['hash'],
            defaults={
                'storage_key': storage_key,
                'chunk_size': chunk['size'],
                'ref_count': 1
            }
        )

        if not created:
            chunk_obj.ref_count += 1
            chunk_obj.save(update_fields=['ref_count'])

        # Create FileChunk mapping
        FileChunk.objects.create(
            file_id=file_id,
            chunk_index=chunk['index'],
            chunk_hash=chunk['hash'],
            chunk_size=chunk['size'],
            storage_key=storage_key
        )

        return storage_key

    def complete_upload(self, file_id: int):
        """Mark upload as complete"""
        file_obj = File.objects.get(id=file_id)

        # Create version snapshot
        FileVersion.objects.create(
            file=file_obj,
            version=file_obj.version,
            file_size_bytes=file_obj.file_size_bytes,
            file_hash=file_obj.file_hash,
            created_by=file_obj.user
        )

        logger.info(f"Upload complete for file {file_id}")

class SyncService:
    """Handle file synchronization"""

    @staticmethod
    def get_changes_since(user: User, device_id: str, timestamp: int) -> List[Dict]:
        """
        Get all file changes since timestamp
        Returns list of changes for sync
        """
        device_sync = DeviceSync.objects.get_or_create(
            user=user,
            device_id=device_id
        )[0]

        # Get all files modified after last sync
        changes = []

        # Modified files
        modified_files = File.objects.filter(
            user=user,
            updated_at__gt=timezone.datetime.fromtimestamp(timestamp / 1000)
        ).exclude(is_deleted=True)

        for file in modified_files:
            changes.append({
                'action': 'update',
                'file_id': file.id,
                'path': file.path,
                'file_hash': file.file_hash,
                'version': file.version,
                'updated_at': file.updated_at.timestamp() * 1000
            })

        # Deleted files
        deleted_files = File.objects.filter(
            user=user,
            deleted_at__gt=timezone.datetime.fromtimestamp(timestamp / 1000),
            is_deleted=True
        )

        for file in deleted_files:
            changes.append({
                'action': 'delete',
                'file_id': file.id,
                'path': file.path,
                'deleted_at': file.deleted_at.timestamp() * 1000
            })

        # Update sync timestamp
        device_sync.last_sync_timestamp = int(timezone.now().timestamp() * 1000)
        device_sync.save(update_fields=['last_sync_timestamp', 'updated_at'])

        return changes

    @staticmethod
    def resolve_conflict(file_id: int, device_version: int) -> Dict:
        """
        Resolve sync conflict
        Strategy: Last-write-wins with version tracking
        """
        file_obj = File.objects.get(id=file_id)

        if file_obj.version > device_version:
            # Server version is newer
            return {
                'resolution': 'server_wins',
                'action': 'download',
                'server_version': file_obj.version
            }
        else:
            # Device version is newer or same
            return {
                'resolution': 'device_wins',
                'action': 'upload',
                'server_version': file_obj.version
            }

# views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class FileViewSet(viewsets.ModelViewSet):
    """File operations"""
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return File.objects.filter(user=self.request.user, is_deleted=False)

    @action(detail=False, methods=['post'])
    def upload_init(self, request):
        """Initialize file upload"""
        file_path = request.data.get('path')
        file_size = request.data.get('size')
        chunks = request.data.get('chunks')  # List of chunk hashes

        service = FileUploadService()
        file_obj, chunks_to_upload = service.initiate_upload(
            user=request.user,
            file_path=file_path,
            file_size=file_size,
            chunks=chunks
        )

        return Response({
            'file_id': file_obj.id,
            'chunks_needed': [c['index'] for c in chunks_to_upload]
        })

    @action(detail=True, methods=['post'])
    def upload_chunk(self, request, pk=None):
        """Upload single chunk"""
        file_obj = self.get_object()
        chunk_data = request.data

        service = FileUploadService()
        storage_key = service.upload_chunk(file_obj.id, chunk_data)

        return Response({'storage_key': storage_key})

    @action(detail=True, methods=['post'])
    def upload_complete(self, request, pk=None):
        """Mark upload as complete"""
        file_obj = self.get_object()

        service = FileUploadService()
        service.complete_upload(file_obj.id)

        return Response({'status': 'complete'})

    @action(detail=False, methods=['get'])
    def sync(self, request):
        """Get changes for sync"""
        device_id = request.query_params.get('device_id')
        timestamp = int(request.query_params.get('timestamp', 0))

        changes = SyncService.get_changes_since(
            user=request.user,
            device_id=device_id,
            timestamp=timestamp
        )

        return Response({
            'changes': changes,
            'timestamp': int(timezone.now().timestamp() * 1000)
        })
```

## 🔐 Conflict Resolution

```python
class ConflictResolver:
    """Handle sync conflicts"""

    @staticmethod
    def resolve(server_file: File, client_file_hash: str, client_timestamp: int) -> str:
        """
        Resolve conflict between server and client versions

        Strategies:
        1. Last-write-wins (simple)
        2. Rename conflicting file (Dropbox approach)
        3. Three-way merge (advanced)
        """
        server_timestamp = int(server_file.updated_at.timestamp() * 1000)

        if client_timestamp > server_timestamp:
            # Client is newer
            if server_file.file_hash == client_file_hash:
                return 'no_conflict'  # Same content
            else:
                return 'client_wins'
        else:
            # Server is newer or equal
            if server_file.file_hash == client_file_hash:
                return 'no_conflict'
            else:
                # Create conflict copy
                return 'create_conflict_copy'

    @staticmethod
    def create_conflict_copy(file_obj: File, device_id: str):
        """Create conflict copy with device identifier"""
        conflict_name = f"{file_obj.name} (Conflict from {device_id})"
        conflict_path = file_obj.path.replace(file_obj.name, conflict_name)

        # Duplicate file metadata
        conflict_file = File.objects.create(
            user=file_obj.user,
            parent=file_obj.parent,
            name=conflict_name,
            path=conflict_path,
            file_size_bytes=file_obj.file_size_bytes,
            file_hash=file_obj.file_hash,
            version=1
        )

        # Copy chunk references
        for chunk in file_obj.chunks.all():
            FileChunk.objects.create(
                file=conflict_file,
                chunk_index=chunk.chunk_index,
                chunk_hash=chunk.chunk_hash,
                chunk_size=chunk.chunk_size,
                storage_key=chunk.storage_key
            )

        return conflict_file
```

## ❓ Interview Questions

### Q1: How do you handle file deduplication?

**Answer:**

1. **Chunk-level deduplication**: Split into chunks, hash each chunk
2. **Global deduplication**: Same chunk across all users (save storage)
3. **Hash comparison**: SHA256 to identify duplicates
4. **Reference counting**: Track how many files use each chunk
5. **Storage savings**: ~40% typical reduction

### Q2: How do you sync files in real-time?

**Answer:**

1. **Long polling / WebSocket**: Maintain connection with sync server
2. **Change notifications**: Server pushes changes to clients
3. **Delta sync**: Only transfer changed parts
4. **Timestamp-based**: Client tracks last sync time
5. **Conflict detection**: Compare timestamps and hashes

### Q3: How do you handle offline changes?

**Answer:**

1. **Local queue**: Store changes locally while offline
2. **Sync on reconnect**: Upload queued changes when online
3. **Conflict resolution**: Handle conflicts with server state
4. **Version vectors**: Track changes across devices

### Q4: How do you optimize for large files (50GB)?

**Answer:**

1. **Chunking**: 4MB chunks for parallel upload/download
2. **Resume capability**: Track uploaded chunks, resume on failure
3. **Streaming**: Don't load entire file in memory
4. **Background sync**: Don't block user interface
5. **Compression**: Compress chunks before transfer

### Q5: How do you ensure data durability?

**Answer:**

1. **Replication**: S3 automatically replicates across AZs
2. **Versioning**: Keep file history, rollback on corruption
3. **Checksums**: Verify integrity with SHA256
4. **Backup**: Regular backups to separate region
5. **Delete protection**: Soft delete with recovery period

---

## 📚 Summary

**Key Components:**

1. **Chunking**: 4MB chunks for efficient sync
2. **Deduplication**: Global chunk-level deduplication
3. **Sync protocol**: Delta sync with conflict resolution
4. **Storage**: S3 for chunks, PostgreSQL for metadata
5. **Real-time**: WebSocket for change notifications
6. **Versioning**: Track file history, rollback capability

**Scalability:**

- Chunk-level parallelism
- S3 for unlimited storage
- Database sharding by user_id
- CDN for file downloads
- Queue-based processing

**Optimization:**

- Only sync changed chunks
- Compress chunks
- Batch operations
- Cache metadata
- Background sync

This pattern applies to: Dropbox, Google Drive, OneDrive, Box, iCloud Drive, file sync services.
