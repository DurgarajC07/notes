# Video Platform Architecture

## Table of Contents

1. [System Overview](#system-overview)
2. [Video Upload Pipeline](#video-upload-pipeline)
3. [Transcoding Service](#transcoding-service)
4. [CDN Integration](#cdn-integration)
5. [Video Player](#video-player)
6. [Playlist Management](#playlist-management)
7. [Live Streaming](#live-streaming)
8. [Analytics & Metrics](#analytics--metrics)
9. [DRM & Security](#drm--security)
10. [Adaptive Bitrate Streaming](#adaptive-bitrate-streaming)
11. [Performance Optimization](#performance-optimization)
12. [Key Takeaways](#key-takeaways)

## System Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Client Layer                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Video    │  │  Playlist  │  │  Upload    │            │
│  │   Player   │  │  Manager   │  │  Manager   │            │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘            │
└────────┼────────────────┼────────────────┼───────────────────┘
         │                │                │
┌────────▼────────────────▼────────────────▼───────────────────┐
│                   API Gateway                                 │
│         (Authentication, Rate Limiting, Routing)              │
└────────┬──────────────────────────────────────────────────────┘
         │
    ┌────┴────┬──────────────┬──────────────┬──────────────┐
    │         │              │              │              │
┌───▼────┐┌──▼─────┐┌───────▼─────┐┌───────▼─────┐┌──────▼────┐
│ Video  ││Transcode││   Storage   ││    CDN      ││Analytics  │
│Service ││ Queue  ││   Service   ││   Service   ││  Service  │
└───┬────┘└──┬─────┘└──────┬──────┘└──────┬──────┘└──────┬────┘
    │        │             │               │              │
┌───▼────────▼─────────────▼───────────────▼──────────────▼────┐
│                      Storage Layer                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  Video   │  │ Metadata │  │   CDN    │  │ Analytics│    │
│  │ Storage  │  │Database  │  │ Storage  │  │    DB    │    │
│  │  (S3)    │  │(Postgres)│  │          │  │          │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└───────────────────────────────────────────────────────────────┘
```

## Video Upload Pipeline

### Core Types

```typescript
interface Video {
  id: string;
  title: string;
  description: string;
  userId: string;
  user: User;
  duration: number;
  status: VideoStatus;
  visibility: VideoVisibility;
  thumbnailUrl: string;
  sources: VideoSource[];
  subtitles: Subtitle[];
  metadata: VideoMetadata;
  analytics: VideoAnalytics;
  createdAt: Date;
  updatedAt: Date;
}

type VideoStatus =
  | "uploading"
  | "processing"
  | "transcoding"
  | "ready"
  | "failed"
  | "deleted";

type VideoVisibility = "public" | "unlisted" | "private";

interface VideoSource {
  quality: VideoQuality;
  url: string;
  format: string;
  size: number;
  bitrate: number;
  codec: string;
}

type VideoQuality =
  | "144p"
  | "240p"
  | "360p"
  | "480p"
  | "720p"
  | "1080p"
  | "1440p"
  | "4k";

interface Subtitle {
  id: string;
  language: string;
  label: string;
  url: string;
  default: boolean;
}

interface VideoMetadata {
  width: number;
  height: number;
  aspectRatio: string;
  frameRate: number;
  codec: string;
  bitrate: number;
  size: number;
}

interface VideoAnalytics {
  views: number;
  likes: number;
  dislikes: number;
  averageWatchTime: number;
  completionRate: number;
}

interface UploadProgress {
  videoId: string;
  progress: number;
  stage: UploadStage;
  eta?: number;
  error?: string;
}

type UploadStage =
  | "preparing"
  | "uploading"
  | "processing"
  | "transcoding"
  | "complete";
```

### Upload Service

```typescript
class VideoUploadService {
  private chunkSize = 5 * 1024 * 1024; // 5MB chunks

  constructor(
    private apiClient: ApiClient,
    private storageClient: StorageClient,
  ) {}

  async uploadVideo(
    file: File,
    metadata: Partial<Video>,
    onProgress: (progress: UploadProgress) => void,
  ): Promise<Video> {
    try {
      // Stage 1: Create video record
      onProgress({
        videoId: "",
        progress: 0,
        stage: "preparing",
      });

      const video = await this.createVideoRecord(metadata);

      // Stage 2: Upload file in chunks
      onProgress({
        videoId: video.id,
        progress: 0,
        stage: "uploading",
      });

      await this.uploadInChunks(video.id, file, (uploadProgress) => {
        onProgress({
          videoId: video.id,
          progress: uploadProgress,
          stage: "uploading",
        });
      });

      // Stage 3: Trigger processing
      onProgress({
        videoId: video.id,
        progress: 100,
        stage: "processing",
      });

      await this.triggerProcessing(video.id);

      return video;
    } catch (error) {
      throw new Error(`Upload failed: ${(error as Error).message}`);
    }
  }

  private async createVideoRecord(metadata: Partial<Video>): Promise<Video> {
    return await this.apiClient.post<Video>("/videos", {
      ...metadata,
      status: "uploading",
    });
  }

  private async uploadInChunks(
    videoId: string,
    file: File,
    onProgress: (progress: number) => void,
  ): Promise<void> {
    const totalChunks = Math.ceil(file.size / this.chunkSize);
    let uploadedChunks = 0;

    // Get upload URL
    const { uploadId, uploadUrls } = await this.apiClient.post<{
      uploadId: string;
      uploadUrls: string[];
    }>(`/videos/${videoId}/upload-urls`, {
      chunks: totalChunks,
    });

    // Upload chunks in parallel (max 3 concurrent)
    const uploadPromises: Promise<void>[] = [];

    for (let i = 0; i < totalChunks; i++) {
      const start = i * this.chunkSize;
      const end = Math.min(start + this.chunkSize, file.size);
      const chunk = file.slice(start, end);

      const uploadPromise = this.uploadChunk(uploadUrls[i], chunk, i).then(
        () => {
          uploadedChunks++;
          const progress = (uploadedChunks / totalChunks) * 100;
          onProgress(progress);
        },
      );

      uploadPromises.push(uploadPromise);

      // Limit concurrent uploads
      if (uploadPromises.length >= 3 || i === totalChunks - 1) {
        await Promise.all(uploadPromises);
        uploadPromises.length = 0;
      }
    }

    // Complete multipart upload
    await this.apiClient.post(`/videos/${videoId}/upload-complete`, {
      uploadId,
    });
  }

  private async uploadChunk(
    url: string,
    chunk: Blob,
    chunkIndex: number,
  ): Promise<void> {
    const response = await fetch(url, {
      method: "PUT",
      body: chunk,
      headers: {
        "Content-Type": "application/octet-stream",
      },
    });

    if (!response.ok) {
      throw new Error(`Chunk ${chunkIndex} upload failed`);
    }
  }

  private async triggerProcessing(videoId: string): Promise<void> {
    await this.apiClient.post(`/videos/${videoId}/process`);
  }

  async cancelUpload(videoId: string): Promise<void> {
    await this.apiClient.post(`/videos/${videoId}/cancel-upload`);
  }

  async resumeUpload(
    videoId: string,
    file: File,
    onProgress: (progress: number) => void,
  ): Promise<void> {
    // Get uploaded chunks
    const { uploadedChunks } = await this.apiClient.get<{
      uploadedChunks: number[];
    }>(`/videos/${videoId}/upload-status`);

    const totalChunks = Math.ceil(file.size / this.chunkSize);
    const remainingChunks = Array.from(
      { length: totalChunks },
      (_, i) => i,
    ).filter((i) => !uploadedChunks.includes(i));

    // Continue upload from where it left off
    for (const chunkIndex of remainingChunks) {
      const start = chunkIndex * this.chunkSize;
      const end = Math.min(start + this.chunkSize, file.size);
      const chunk = file.slice(start, end);

      const { uploadUrl } = await this.apiClient.post<{ uploadUrl: string }>(
        `/videos/${videoId}/upload-url`,
        { chunkIndex },
      );

      await this.uploadChunk(uploadUrl, chunk, chunkIndex);

      const progress =
        ((uploadedChunks.length + remainingChunks.indexOf(chunkIndex) + 1) /
          totalChunks) *
        100;
      onProgress(progress);
    }
  }
}
```

### Upload Component

```typescript
export const VideoUploader: React.FC = () => {
  const [file, setFile] = useState<File | null>(null);
  const [metadata, setMetadata] = useState<Partial<Video>>({
    title: '',
    description: '',
    visibility: 'public',
  });
  const [uploadProgress, setUploadProgress] = useState<UploadProgress | null>(
    null
  );
  const [videoId, setVideoId] = useState<string | null>(null);

  const uploadService = new VideoUploadService(apiClient, storageClient);

  const handleFileSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
    const selectedFile = e.target.files?.[0];
    if (selectedFile && selectedFile.type.startsWith('video/')) {
      setFile(selectedFile);
      // Generate thumbnail preview
      generateThumbnail(selectedFile);
    }
  };

  const handleUpload = async () => {
    if (!file) return;

    try {
      const video = await uploadService.uploadVideo(
        file,
        metadata,
        (progress) => {
          setUploadProgress(progress);
          setVideoId(progress.videoId);
        }
      );

      toast.success('Video uploaded successfully!');
      // Navigate to video page or continue editing
    } catch (error) {
      toast.error('Upload failed');
      console.error(error);
    }
  };

  const handleCancel = async () => {
    if (videoId) {
      await uploadService.cancelUpload(videoId);
      setUploadProgress(null);
      setVideoId(null);
      setFile(null);
    }
  };

  return (
    <div className="video-uploader">
      {!file && (
        <div className="upload-dropzone">
          <input
            type="file"
            accept="video/*"
            onChange={handleFileSelect}
            id="video-file"
            style={{ display: 'none' }}
          />
          <label htmlFor="video-file">
            <UploadIcon />
            <p>Click to select video or drag and drop</p>
            <span>MP4, WebM, or AVI • Max 2GB</span>
          </label>
        </div>
      )}

      {file && !uploadProgress && (
        <div className="upload-form">
          <div className="video-preview">
            <video src={URL.createObjectURL(file)} controls />
          </div>

          <div className="metadata-form">
            <input
              type="text"
              placeholder="Title"
              value={metadata.title}
              onChange={(e) =>
                setMetadata({ ...metadata, title: e.target.value })
              }
            />

            <textarea
              placeholder="Description"
              value={metadata.description}
              onChange={(e) =>
                setMetadata({ ...metadata, description: e.target.value })
              }
            />

            <select
              value={metadata.visibility}
              onChange={(e) =>
                setMetadata({
                  ...metadata,
                  visibility: e.target.value as VideoVisibility,
                })
              }
            >
              <option value="public">Public</option>
              <option value="unlisted">Unlisted</option>
              <option value="private">Private</option>
            </select>

            <div className="upload-actions">
              <button onClick={() => setFile(null)}>Cancel</button>
              <button onClick={handleUpload} className="btn-primary">
                Upload Video
              </button>
            </div>
          </div>
        </div>
      )}

      {uploadProgress && (
        <div className="upload-progress">
          <h3>Uploading Video</h3>

          <div className="progress-bar">
            <div
              className="progress-fill"
              style={{ width: `${uploadProgress.progress}%` }}
            />
          </div>

          <div className="progress-info">
            <span>{uploadProgress.stage}</span>
            <span>{Math.round(uploadProgress.progress)}%</span>
          </div>

          {uploadProgress.stage === 'uploading' && (
            <button onClick={handleCancel}>Cancel Upload</button>
          )}

          {uploadProgress.stage === 'transcoding' && (
            <p>Your video is being processed. This may take a few minutes.</p>
          )}
        </div>
      )}
    </div>
  );
};
```

## Transcoding Service

### Transcoding Configuration

```typescript
interface TranscodingProfile {
  name: string;
  quality: VideoQuality;
  width: number;
  height: number;
  bitrate: number;
  codec: string;
  format: string;
}

const TRANSCODING_PROFILES: TranscodingProfile[] = [
  {
    name: "4K",
    quality: "4k",
    width: 3840,
    height: 2160,
    bitrate: 20000,
    codec: "h264",
    format: "mp4",
  },
  {
    name: "1080p",
    quality: "1080p",
    width: 1920,
    height: 1080,
    bitrate: 8000,
    codec: "h264",
    format: "mp4",
  },
  {
    name: "720p",
    quality: "720p",
    width: 1280,
    height: 720,
    bitrate: 5000,
    codec: "h264",
    format: "mp4",
  },
  {
    name: "480p",
    quality: "480p",
    width: 854,
    height: 480,
    bitrate: 2500,
    codec: "h264",
    format: "mp4",
  },
  {
    name: "360p",
    quality: "360p",
    width: 640,
    height: 360,
    bitrate: 1000,
    codec: "h264",
    format: "mp4",
  },
];

class TranscodingService {
  constructor(
    private queue: QueueClient,
    private storage: StorageClient,
  ) {}

  async transcodeVideo(videoId: string, sourceUrl: string): Promise<void> {
    // Add transcoding jobs to queue for each profile
    const jobs = TRANSCODING_PROFILES.map((profile) => ({
      videoId,
      sourceUrl,
      profile,
      priority: this.getJobPriority(profile.quality),
    }));

    await this.queue.addBatch("video-transcoding", jobs);
  }

  private getJobPriority(quality: VideoQuality): number {
    const priorityMap: Record<VideoQuality, number> = {
      "4k": 1,
      "1440p": 2,
      "1080p": 3,
      "720p": 4,
      "480p": 5,
      "360p": 6,
      "240p": 7,
      "144p": 8,
    };
    return priorityMap[quality] || 5;
  }

  async processTranscodingJob(job: TranscodingJob): Promise<void> {
    const { videoId, sourceUrl, profile } = job;

    try {
      // Download source video
      const sourceFile = await this.storage.download(sourceUrl);

      // Transcode video using FFmpeg
      const transcodedFile = await this.transcode(sourceFile, profile);

      // Upload transcoded video
      const url = await this.storage.upload(
        `videos/${videoId}/${profile.quality}.${profile.format}`,
        transcodedFile,
      );

      // Update video record
      await this.addVideoSource(videoId, {
        quality: profile.quality,
        url,
        format: profile.format,
        size: transcodedFile.size,
        bitrate: profile.bitrate,
        codec: profile.codec,
      });

      // Generate thumbnail if 720p
      if (profile.quality === "720p") {
        await this.generateThumbnail(videoId, transcodedFile);
      }
    } catch (error) {
      console.error(`Transcoding failed for ${profile.quality}:`, error);
      throw error;
    }
  }

  private async transcode(
    sourceFile: Buffer,
    profile: TranscodingProfile,
  ): Promise<Buffer> {
    // Use FFmpeg to transcode
    const command = [
      "-i",
      "pipe:0",
      "-c:v",
      profile.codec,
      "-b:v",
      `${profile.bitrate}k`,
      "-vf",
      `scale=${profile.width}:${profile.height}`,
      "-c:a",
      "aac",
      "-b:a",
      "128k",
      "-f",
      profile.format,
      "pipe:1",
    ];

    return await this.executeFFmpeg(command, sourceFile);
  }

  private async executeFFmpeg(
    command: string[],
    input: Buffer,
  ): Promise<Buffer> {
    // Execute FFmpeg command and return output
    // Implementation depends on platform (Node.js, serverless, etc.)
    return Buffer.from([]);
  }

  private async generateThumbnail(
    videoId: string,
    videoFile: Buffer,
  ): Promise<void> {
    // Generate thumbnail at 5 seconds
    const command = [
      "-i",
      "pipe:0",
      "-ss",
      "00:00:05",
      "-vframes",
      "1",
      "-vf",
      "scale=1280:720",
      "-f",
      "image2",
      "pipe:1",
    ];

    const thumbnailBuffer = await this.executeFFmpeg(command, videoFile);

    const thumbnailUrl = await this.storage.upload(
      `videos/${videoId}/thumbnail.jpg`,
      thumbnailBuffer,
    );

    await this.updateVideoThumbnail(videoId, thumbnailUrl);
  }

  private async addVideoSource(
    videoId: string,
    source: VideoSource,
  ): Promise<void> {
    await apiClient.post(`/videos/${videoId}/sources`, source);
  }

  private async updateVideoThumbnail(
    videoId: string,
    thumbnailUrl: string,
  ): Promise<void> {
    await apiClient.patch(`/videos/${videoId}`, { thumbnailUrl });
  }
}

interface TranscodingJob {
  videoId: string;
  sourceUrl: string;
  profile: TranscodingProfile;
  priority: number;
}
```

## CDN Integration

### CDN Service

```typescript
class CDNService {
  constructor(
    private cdnProvider: "cloudflare" | "cloudfront" | "fastly",
    private cdnConfig: CDNConfig,
  ) {}

  getVideoUrl(videoId: string, quality: VideoQuality): string {
    const baseUrl = this.cdnConfig.baseUrl;
    return `${baseUrl}/videos/${videoId}/${quality}.mp4`;
  }

  getHLSManifestUrl(videoId: string): string {
    const baseUrl = this.cdnConfig.baseUrl;
    return `${baseUrl}/videos/${videoId}/master.m3u8`;
  }

  getDASHManifestUrl(videoId: string): string {
    const baseUrl = this.cdnConfig.baseUrl;
    return `${baseUrl}/videos/${videoId}/manifest.mpd`;
  }

  async invalidateCache(videoId: string): Promise<void> {
    const paths = [`/videos/${videoId}/*`, `/thumbnails/${videoId}/*`];

    switch (this.cdnProvider) {
      case "cloudflare":
        await this.invalidateCloudflare(paths);
        break;
      case "cloudfront":
        await this.invalidateCloudFront(paths);
        break;
      case "fastly":
        await this.invalidateFastly(paths);
        break;
    }
  }

  private async invalidateCloudflare(paths: string[]): Promise<void> {
    // Cloudflare purge cache API
    await fetch(
      `https://api.cloudflare.com/client/v4/zones/${this.cdnConfig.zoneId}/purge_cache`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.cdnConfig.apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ files: paths }),
      },
    );
  }

  private async invalidateCloudFront(paths: string[]): Promise<void> {
    // AWS CloudFront invalidation
    // Implementation using AWS SDK
  }

  private async invalidateFastly(paths: string[]): Promise<void> {
    // Fastly purge API
    // Implementation using Fastly SDK
  }

  getSignedUrl(
    videoId: string,
    quality: VideoQuality,
    expiresIn: number = 3600,
  ): string {
    const url = this.getVideoUrl(videoId, quality);
    const expires = Math.floor(Date.now() / 1000) + expiresIn;
    const signature = this.generateSignature(url, expires);

    return `${url}?expires=${expires}&signature=${signature}`;
  }

  private generateSignature(url: string, expires: number): string {
    const crypto = require("crypto");
    const message = `${url}${expires}`;
    return crypto
      .createHmac("sha256", this.cdnConfig.secretKey)
      .update(message)
      .digest("hex");
  }
}

interface CDNConfig {
  baseUrl: string;
  zoneId?: string;
  apiKey?: string;
  secretKey: string;
}
```

## Video Player

### Custom Video Player Component

```typescript
import React, { useRef, useState, useEffect } from 'react';
import Hls from 'hls.js';
import dashjs from 'dashjs';

interface VideoPlayerProps {
  video: Video;
  autoplay?: boolean;
  controls?: boolean;
  onTimeUpdate?: (currentTime: number) => void;
  onEnded?: () => void;
}

export const VideoPlayer: React.FC<VideoPlayerProps> = ({
  video,
  autoplay = false,
  controls = true,
  onTimeUpdate,
  onEnded,
}) => {
  const videoRef = useRef<HTMLVideoElement>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [currentTime, setCurrentTime] = useState(0);
  const [duration, setDuration] = useState(0);
  const [volume, setVolume] = useState(1);
  const [isMuted, setIsMuted] = useState(false);
  const [selectedQuality, setSelectedQuality] = useState<VideoQuality>('720p');
  const [selectedSubtitle, setSelectedSubtitle] = useState<string | null>(null);
  const [isFullscreen, setIsFullscreen] = useState(false);
  const [buffered, setBuffered] = useState(0);

  const hlsRef = useRef<Hls | null>(null);
  const dashPlayerRef = useRef<dashjs.MediaPlayerClass | null>(null);

  useEffect(() => {
    if (!videoRef.current) return;

    const videoElement = videoRef.current;

    // Initialize HLS or DASH player
    if (video.sources.length > 0 && video.sources[0].format === 'm3u8') {
      initializeHLS();
    } else if (video.sources.length > 0 && video.sources[0].format === 'mpd') {
      initializeDASH();
    } else {
      // Use native video element
      videoElement.src = video.sources.find(s => s.quality === selectedQuality)?.url || '';
    }

    return () => {
      cleanupPlayers();
    };
  }, [video]);

  const initializeHLS = () => {
    if (!videoRef.current || !Hls.isSupported()) return;

    const hls = new Hls({
      enableWorker: true,
      lowLatencyMode: true,
      backBufferLength: 90,
    });

    hlsRef.current = hls;

    const manifestUrl = getCDNService().getHLSManifestUrl(video.id);
    hls.loadSource(manifestUrl);
    hls.attachMedia(videoRef.current);

    hls.on(Hls.Events.MANIFEST_PARSED, () => {
      if (autoplay) {
        videoRef.current?.play();
      }
    });

    hls.on(Hls.Events.ERROR, (event, data) => {
      console.error('HLS error:', data);
      if (data.fatal) {
        switch (data.type) {
          case Hls.ErrorTypes.NETWORK_ERROR:
            hls.startLoad();
            break;
          case Hls.ErrorTypes.MEDIA_ERROR:
            hls.recoverMediaError();
            break;
          default:
            hls.destroy();
            break;
        }
      }
    });
  };

  const initializeDASH = () => {
    if (!videoRef.current) return;

    const player = dashjs.MediaPlayer().create();
    dashPlayerRef.current = player;

    player.initialize(
      videoRef.current,
      getCDNService().getDASHManifestUrl(video.id),
      autoplay
    );

    player.updateSettings({
      streaming: {
        buffer: {
          bufferTimeAtTopQuality: 30,
          bufferTimeAtTopQualityLongForm: 60,
        },
      },
    });
  };

  const cleanupPlayers = () => {
    if (hlsRef.current) {
      hlsRef.current.destroy();
      hlsRef.current = null;
    }
    if (dashPlayerRef.current) {
      dashPlayerRef.current.destroy();
      dashPlayerRef.current = null;
    }
  };

  const togglePlay = () => {
    if (!videoRef.current) return;

    if (isPlaying) {
      videoRef.current.pause();
    } else {
      videoRef.current.play();
    }
  };

  const handleTimeUpdate = () => {
    if (!videoRef.current) return;

    const time = videoRef.current.currentTime;
    setCurrentTime(time);
    onTimeUpdate?.(time);

    // Update buffered
    if (videoRef.current.buffered.length > 0) {
      const bufferedEnd = videoRef.current.buffered.end(
        videoRef.current.buffered.length - 1
      );
      setBuffered((bufferedEnd / duration) * 100);
    }
  };

  const handleSeek = (time: number) => {
    if (!videoRef.current) return;
    videoRef.current.currentTime = time;
  };

  const handleVolumeChange = (newVolume: number) => {
    if (!videoRef.current) return;
    videoRef.current.volume = newVolume;
    setVolume(newVolume);
    setIsMuted(newVolume === 0);
  };

  const toggleMute = () => {
    if (!videoRef.current) return;
    videoRef.current.muted = !isMuted;
    setIsMuted(!isMuted);
  };

  const toggleFullscreen = () => {
    if (!document.fullscreenElement) {
      videoRef.current?.requestFullscreen();
      setIsFullscreen(true);
    } else {
      document.exitFullscreen();
      setIsFullscreen(false);
    }
  };

  const changeQuality = (quality: VideoQuality) => {
    if (!videoRef.current) return;

    const currentTime = videoRef.current.currentTime;
    const wasPlaying = !videoRef.current.paused;

    setSelectedQuality(quality);

    if (hlsRef.current) {
      // HLS will handle quality switching
      const levels = hlsRef.current.levels;
      const levelIndex = levels.findIndex(l =>
        l.height === parseInt(quality.replace('p', ''))
      );
      if (levelIndex !== -1) {
        hlsRef.current.currentLevel = levelIndex;
      }
    } else {
      // Manual quality switch
      const source = video.sources.find(s => s.quality === quality);
      if (source) {
        videoRef.current.src = source.url;
        videoRef.current.currentTime = currentTime;
        if (wasPlaying) {
          videoRef.current.play();
        }
      }
    }
  };

  const changeSubtitle = (subtitleId: string | null) => {
    if (!videoRef.current) return;

    const tracks = videoRef.current.textTracks;

    for (let i = 0; i < tracks.length; i++) {
      tracks[i].mode = tracks[i].id === subtitleId ? 'showing' : 'hidden';
    }

    setSelectedSubtitle(subtitleId);
  };

  const skip = (seconds: number) => {
    if (!videoRef.current) return;
    videoRef.current.currentTime += seconds;
  };

  return (
    <div className={`video-player ${isFullscreen ? 'fullscreen' : ''}`}>
      <video
        ref={videoRef}
        className="video-element"
        poster={video.thumbnailUrl}
        onPlay={() => setIsPlaying(true)}
        onPause={() => setIsPlaying(false)}
        onTimeUpdate={handleTimeUpdate}
        onDurationChange={(e) => setDuration(e.currentTarget.duration)}
        onEnded={() => {
          setIsPlaying(false);
          onEnded?.();
        }}
        onClick={togglePlay}
      >
        {video.subtitles.map((subtitle) => (
          <track
            key={subtitle.id}
            id={subtitle.id}
            kind="subtitles"
            src={subtitle.url}
            srcLang={subtitle.language}
            label={subtitle.label}
            default={subtitle.default}
          />
        ))}
      </video>

      {controls && (
        <div className="video-controls">
          <div className="progress-bar" onClick={(e) => {
            const rect = e.currentTarget.getBoundingClientRect();
            const percent = (e.clientX - rect.left) / rect.width;
            handleSeek(percent * duration);
          }}>
            <div className="progress-buffered" style={{ width: `${buffered}%` }} />
            <div className="progress-played" style={{ width: `${(currentTime / duration) * 100}%` }} />
          </div>

          <div className="controls-bottom">
            <button onClick={togglePlay}>
              {isPlaying ? <PauseIcon /> : <PlayIcon />}
            </button>

            <button onClick={() => skip(-10)}>
              <Rewind10Icon />
            </button>

            <button onClick={() => skip(10)}>
              <Forward10Icon />
            </button>

            <span className="time-display">
              {formatTime(currentTime)} / {formatTime(duration)}
            </span>

            <div className="volume-control">
              <button onClick={toggleMute}>
                {isMuted ? <MutedIcon /> : <VolumeIcon />}
              </button>
              <input
                type="range"
                min="0"
                max="1"
                step="0.1"
                value={volume}
                onChange={(e) => handleVolumeChange(parseFloat(e.target.value))}
              />
            </div>

            <select value={selectedQuality} onChange={(e) => changeQuality(e.target.value as VideoQuality)}>
              {video.sources.map((source) => (
                <option key={source.quality} value={source.quality}>
                  {source.quality}
                </option>
              ))}
            </select>

            <select value={selectedSubtitle || ''} onChange={(e) => changeSubtitle(e.target.value || null)}>
              <option value="">No Subtitles</option>
              {video.subtitles.map((subtitle) => (
                <option key={subtitle.id} value={subtitle.id}>
                  {subtitle.label}
                </option>
              ))}
            </select>

            <button onClick={toggleFullscreen}>
              {isFullscreen ? <ExitFullscreenIcon /> : <FullscreenIcon />}
            </button>
          </div>
        </div>
      )}
    </div>
  );
};

function formatTime(seconds: number): string {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);

  if (hours > 0) {
    return `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  }
  return `${minutes}:${secs.toString().padStart(2, '0')}`;
}
```

## Playlist Management

### Playlist Types and Service

```typescript
interface Playlist {
  id: string;
  title: string;
  description: string;
  userId: string;
  user: User;
  visibility: PlaylistVisibility;
  videos: PlaylistVideo[];
  thumbnailUrl: string;
  videosCount: number;
  totalDuration: number;
  createdAt: Date;
  updatedAt: Date;
}

type PlaylistVisibility = "public" | "unlisted" | "private";

interface PlaylistVideo {
  videoId: string;
  video: Video;
  order: number;
  addedAt: Date;
}

class PlaylistService {
  constructor(private apiClient: ApiClient) {}

  async createPlaylist(data: CreatePlaylistData): Promise<Playlist> {
    return await this.apiClient.post<Playlist>("/playlists", data);
  }

  async getPlaylist(playlistId: string): Promise<Playlist> {
    return await this.apiClient.get<Playlist>(`/playlists/${playlistId}`);
  }

  async updatePlaylist(
    playlistId: string,
    data: Partial<Playlist>,
  ): Promise<Playlist> {
    return await this.apiClient.put<Playlist>(`/playlists/${playlistId}`, data);
  }

  async deletePlaylist(playlistId: string): Promise<void> {
    await this.apiClient.delete(`/playlists/${playlistId}`);
  }

  async addVideo(playlistId: string, videoId: string): Promise<Playlist> {
    return await this.apiClient.post<Playlist>(
      `/playlists/${playlistId}/videos`,
      { videoId },
    );
  }

  async removeVideo(playlistId: string, videoId: string): Promise<Playlist> {
    return await this.apiClient.delete<Playlist>(
      `/playlists/${playlistId}/videos/${videoId}`,
    );
  }

  async reorderVideos(
    playlistId: string,
    videoOrders: Array<{ videoId: string; order: number }>,
  ): Promise<Playlist> {
    return await this.apiClient.put<Playlist>(
      `/playlists/${playlistId}/reorder`,
      { videoOrders },
    );
  }

  async getUserPlaylists(userId: string): Promise<Playlist[]> {
    return await this.apiClient.get<Playlist[]>(`/users/${userId}/playlists`);
  }
}

interface CreatePlaylistData {
  title: string;
  description?: string;
  visibility: PlaylistVisibility;
}
```

### Playlist Component

```typescript
export const PlaylistPlayer: React.FC<{ playlistId: string }> = ({
  playlistId,
}) => {
  const [playlist, setPlaylist] = useState<Playlist | null>(null);
  const [currentVideoIndex, setCurrentVideoIndex] = useState(0);
  const [autoplayNext, setAutoplayNext] = useState(true);

  useEffect(() => {
    loadPlaylist();
  }, [playlistId]);

  const loadPlaylist = async () => {
    const playlistService = new PlaylistService(apiClient);
    const data = await playlistService.getPlaylist(playlistId);
    setPlaylist(data);
  };

  const handleVideoEnd = () => {
    if (autoplayNext && currentVideoIndex < playlist!.videos.length - 1) {
      setCurrentVideoIndex(currentVideoIndex + 1);
    }
  };

  const playVideo = (index: number) => {
    setCurrentVideoIndex(index);
  };

  if (!playlist) {
    return <LoadingSpinner />;
  }

  const currentVideo = playlist.videos[currentVideoIndex];

  return (
    <div className="playlist-player">
      <div className="player-section">
        <VideoPlayer
          video={currentVideo.video}
          onEnded={handleVideoEnd}
          autoplay
        />

        <div className="video-info">
          <h2>{currentVideo.video.title}</h2>
          <p>{currentVideo.video.description}</p>
        </div>
      </div>

      <div className="playlist-sidebar">
        <div className="playlist-header">
          <h3>{playlist.title}</h3>
          <p>
            {currentVideoIndex + 1} / {playlist.videos.length}
          </p>
        </div>

        <div className="autoplay-toggle">
          <label>
            <input
              type="checkbox"
              checked={autoplayNext}
              onChange={(e) => setAutoplayNext(e.target.checked)}
            />
            Autoplay
          </label>
        </div>

        <div className="playlist-videos">
          {playlist.videos.map((playlistVideo, index) => (
            <div
              key={playlistVideo.videoId}
              className={`playlist-video-item ${
                index === currentVideoIndex ? 'active' : ''
              }`}
              onClick={() => playVideo(index)}
            >
              <div className="video-thumbnail">
                <img
                  src={playlistVideo.video.thumbnailUrl}
                  alt={playlistVideo.video.title}
                />
                <span className="video-duration">
                  {formatDuration(playlistVideo.video.duration)}
                </span>
              </div>

              <div className="video-details">
                <h4>{playlistVideo.video.title}</h4>
                <span className="video-author">
                  {playlistVideo.video.user.displayName}
                </span>
              </div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

function formatDuration(seconds: number): string {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);

  if (hours > 0) {
    return `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  }
  return `${minutes}:${secs.toString().padStart(2, '0')}`;
}
```

## Analytics & Metrics

### Analytics Service

```typescript
interface VideoView {
  videoId: string;
  userId?: string;
  sessionId: string;
  watchTime: number;
  completionRate: number;
  quality: VideoQuality;
  device: string;
  browser: string;
  timestamp: Date;
}

interface VideoEngagement {
  videoId: string;
  likes: number;
  dislikes: number;
  shares: number;
  comments: number;
  averageWatchTime: number;
  completionRate: number;
  retentionCurve: number[];
}

class VideoAnalyticsService {
  constructor(private apiClient: ApiClient) {}

  async trackView(data: Partial<VideoView>): Promise<void> {
    await this.apiClient.post("/analytics/views", data);
  }

  async trackWatchTime(
    videoId: string,
    sessionId: string,
    watchTime: number,
  ): Promise<void> {
    await this.apiClient.post("/analytics/watch-time", {
      videoId,
      sessionId,
      watchTime,
      timestamp: new Date(),
    });
  }

  async getVideoAnalytics(videoId: string): Promise<VideoEngagement> {
    return await this.apiClient.get<VideoEngagement>(
      `/analytics/videos/${videoId}`,
    );
  }

  async getChannelAnalytics(
    userId: string,
    timeRange: TimeRange,
  ): Promise<ChannelAnalytics> {
    return await this.apiClient.get<ChannelAnalytics>(
      `/analytics/channels/${userId}`,
      { params: { timeRange } },
    );
  }
}

interface ChannelAnalytics {
  totalViews: number;
  totalWatchTime: number;
  subscribers: number;
  averageViewDuration: number;
  topVideos: Array<{ video: Video; views: number }>;
  viewsOverTime: Array<{ date: Date; views: number }>;
}

// Hook for tracking watch time
export function useVideoTracking(videoId: string) {
  const sessionId = useRef(generateSessionId());
  const analyticsService = new VideoAnalyticsService(apiClient);
  const lastUpdateTime = useRef(0);

  const trackWatchTime = useCallback(
    (currentTime: number) => {
      // Only track every 10 seconds
      if (currentTime - lastUpdateTime.current < 10) return;

      analyticsService.trackWatchTime(videoId, sessionId.current, currentTime);

      lastUpdateTime.current = currentTime;
    },
    [videoId],
  );

  useEffect(() => {
    // Track initial view
    analyticsService.trackView({
      videoId,
      sessionId: sessionId.current,
      device: getDeviceType(),
      browser: getBrowserType(),
    });
  }, [videoId]);

  return { trackWatchTime };
}

function generateSessionId(): string {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

function getDeviceType(): string {
  const ua = navigator.userAgent;
  if (/mobile/i.test(ua)) return "mobile";
  if (/tablet/i.test(ua)) return "tablet";
  return "desktop";
}

function getBrowserType(): string {
  const ua = navigator.userAgent;
  if (/chrome/i.test(ua)) return "chrome";
  if (/firefox/i.test(ua)) return "firefox";
  if (/safari/i.test(ua)) return "safari";
  if (/edge/i.test(ua)) return "edge";
  return "other";
}
```

## Key Takeaways

### 1. **Chunked Upload**

Implement chunked upload for large video files. Support resume capability for failed uploads. Use multipart upload with S3 or similar storage.

### 2. **Adaptive Bitrate Streaming**

Use HLS or DASH for automatic quality switching. Generate multiple quality versions during transcoding. Let client select best quality based on bandwidth.

### 3. **CDN Optimization**

Serve video content through CDN for global reach. Use signed URLs for private content. Implement cache invalidation strategies.

### 4. **Progressive Transcoding**

Prioritize common qualities (720p, 1080p) first. Transcode higher/lower qualities in background. Use job queues for scalable transcoding.

### 5. **Player Features**

Implement custom controls for branding. Support subtitles and captions. Add keyboard shortcuts for better UX. Remember playback position.

### 6. **Playlist Management**

Support creating and managing playlists. Enable autoplay for continuous viewing. Allow reordering videos with drag-and-drop.

### 7. **Analytics Tracking**

Track views, watch time, and engagement. Monitor completion rates and drop-off points. Provide creator analytics dashboard.

### 8. **Performance Optimization**

Preload video metadata only. Use progressive streaming. Implement video thumbnails preview on hover. Lazy load off-screen videos.

### 9. **Mobile Optimization**

Support mobile-optimized formats. Implement picture-in-picture mode. Optimize bandwidth usage for cellular. Support gestures for controls.

### 10. **Security & DRM**

Implement signed URLs for access control. Support DRM for premium content. Monitor for unauthorized sharing. Implement geo-restrictions if needed.

---

**Further Reading:**

- HLS.js Documentation
- DASH.js Documentation
- FFmpeg Video Processing Guide
- Video CDN Best Practices
