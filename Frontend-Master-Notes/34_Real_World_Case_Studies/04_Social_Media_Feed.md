# Social Media Feed Architecture

## Table of Contents

1. [System Overview](#system-overview)
2. [Feed Architecture](#feed-architecture)
3. [Infinite Scroll](#infinite-scroll)
4. [Optimistic Updates](#optimistic-updates)
5. [Reactions System](#reactions-system)
6. [Comments System](#comments-system)
7. [Real-Time Updates](#real-time-updates)
8. [Media Handling](#media-handling)
9. [Content Moderation](#content-moderation)
10. [Performance Optimization](#performance-optimization)
11. [Offline Support](#offline-support)
12. [Key Takeaways](#key-takeaways)

## System Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Frontend Layer                            │
│  ┌────────────┐  ┌────────────┐  ┌───────────┐             │
│  │    Feed    │  │  Post      │  │  Comment  │             │
│  │ Component  │  │ Component  │  │ Component │             │
│  └─────┬──────┘  └─────┬──────┘  └─────┬─────┘             │
└────────┼────────────────┼────────────────┼───────────────────┘
         │                │                │
┌────────▼────────────────▼────────────────▼───────────────────┐
│              State Management (React Query)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  - Infinite Query                                     │   │
│  │  - Optimistic Updates                                 │   │
│  │  - Cache Management                                   │   │
│  └──────────────────────────────────────────────────────┘   │
└────────┬──────────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────┐
│                      API Layer                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  REST    │  │  GraphQL │  │ WebSocket│  │   CDN    │    │
│  │   API    │  │   API    │  │  Events  │  │  (Media) │    │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘    │
└────────┼─────────────┼──────────────┼─────────────┼──────────┘
         │             │              │             │
┌────────▼─────────────▼──────────────▼─────────────▼──────────┐
│                   Backend Services                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  Feed    │  │  Post    │  │ Reaction │  │  Media   │    │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │    │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘    │
└────────┼─────────────┼──────────────┼─────────────┼──────────┘
         │             │              │             │
┌────────▼─────────────▼──────────────▼─────────────▼──────────┐
│                    Database Layer                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  Posts   │  │ Comments │  │ Reactions│  │  Redis   │    │
│  │   DB     │  │    DB    │  │    DB    │  │  Cache   │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└───────────────────────────────────────────────────────────────┘
```

## Feed Architecture

### Core Types

```typescript
interface Post {
  id: string;
  userId: string;
  user: User;
  content: string;
  mediaUrls: MediaItem[];
  visibility: PostVisibility;
  reactions: ReactionsSummary;
  commentsCount: number;
  sharesCount: number;
  createdAt: Date;
  updatedAt: Date;
  isEdited: boolean;
}

interface User {
  id: string;
  username: string;
  displayName: string;
  avatarUrl?: string;
  verified: boolean;
}

interface MediaItem {
  id: string;
  type: "image" | "video" | "gif";
  url: string;
  thumbnailUrl?: string;
  width: number;
  height: number;
  duration?: number; // for videos
  alt?: string;
}

type PostVisibility = "public" | "friends" | "private";

interface ReactionsSummary {
  total: number;
  byType: Record<ReactionType, number>;
  userReaction?: ReactionType;
}

type ReactionType = "like" | "love" | "haha" | "wow" | "sad" | "angry";

interface Comment {
  id: string;
  postId: string;
  userId: string;
  user: User;
  content: string;
  parentId?: string; // for nested comments
  reactions: ReactionsSummary;
  repliesCount: number;
  createdAt: Date;
  updatedAt: Date;
  isEdited: boolean;
}

interface FeedItem {
  id: string;
  type: "post" | "shared_post" | "ad";
  data: Post | SharedPost | Ad;
  priority: number;
}

interface SharedPost extends Post {
  originalPost: Post;
  shareComment?: string;
}
```

### Feed Service

```typescript
class FeedService {
  constructor(
    private apiClient: ApiClient,
    private cacheManager: CacheManager,
  ) {}

  async getFeed(
    userId: string,
    params: FeedParams,
  ): Promise<PaginatedResponse<FeedItem>> {
    const cacheKey = `feed:${userId}:${JSON.stringify(params)}`;

    // Try cache first
    const cached =
      await this.cacheManager.get<PaginatedResponse<FeedItem>>(cacheKey);
    if (cached && params.cursor === undefined) {
      return cached;
    }

    // Fetch from API
    const response = await this.apiClient.get<PaginatedResponse<FeedItem>>(
      "/feed",
      {
        params: {
          ...params,
          userId,
        },
      },
    );

    // Cache first page
    if (!params.cursor) {
      await this.cacheManager.set(cacheKey, response, 60); // 1 minute
    }

    return response;
  }

  async createPost(data: CreatePostData): Promise<Post> {
    const post = await this.apiClient.post<Post>("/posts", data);

    // Invalidate feed cache
    await this.cacheManager.invalidate(`feed:${data.userId}:*`);

    return post;
  }

  async updatePost(postId: string, data: UpdatePostData): Promise<Post> {
    const post = await this.apiClient.put<Post>(`/posts/${postId}`, data);

    // Invalidate post cache
    await this.cacheManager.invalidate(`post:${postId}`);

    return post;
  }

  async deletePost(postId: string): Promise<void> {
    await this.apiClient.delete(`/posts/${postId}`);

    // Invalidate caches
    await this.cacheManager.invalidate(`post:${postId}`);
  }

  async getPost(postId: string): Promise<Post> {
    const cacheKey = `post:${postId}`;

    const cached = await this.cacheManager.get<Post>(cacheKey);
    if (cached) return cached;

    const post = await this.apiClient.get<Post>(`/posts/${postId}`);
    await this.cacheManager.set(cacheKey, post, 300); // 5 minutes

    return post;
  }
}

interface FeedParams {
  cursor?: string;
  limit?: number;
  filter?: FeedFilter;
}

interface FeedFilter {
  type?: "all" | "friends" | "following";
  mediaOnly?: boolean;
}

interface CreatePostData {
  userId: string;
  content: string;
  mediaUrls?: string[];
  visibility: PostVisibility;
}

interface UpdatePostData {
  content?: string;
  visibility?: PostVisibility;
}
```

## Infinite Scroll

### Infinite Query Implementation

```typescript
import { useInfiniteQuery, QueryClient } from '@tanstack/react-query';
import { useIntersectionObserver } from '@/hooks/useIntersectionObserver';

interface UseFeedOptions {
  userId: string;
  filter?: FeedFilter;
  enabled?: boolean;
}

export function useFeed({ userId, filter, enabled = true }: UseFeedOptions) {
  const feedService = new FeedService(apiClient, cacheManager);

  return useInfiniteQuery({
    queryKey: ['feed', userId, filter],
    queryFn: ({ pageParam }) =>
      feedService.getFeed(userId, {
        cursor: pageParam,
        limit: 20,
        filter,
      }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    enabled,
    staleTime: 60 * 1000, // 1 minute
    cacheTime: 5 * 60 * 1000, // 5 minutes
  });
}

// Feed component with infinite scroll
export const Feed: React.FC<{ userId: string }> = ({ userId }) => {
  const [filter, setFilter] = useState<FeedFilter>({ type: 'all' });

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
    error,
  } = useFeed({ userId, filter });

  const { ref: loadMoreRef } = useIntersectionObserver({
    onIntersect: () => {
      if (hasNextPage && !isFetchingNextPage) {
        fetchNextPage();
      }
    },
    threshold: 0.5,
  });

  if (isLoading) {
    return <FeedSkeleton />;
  }

  if (isError) {
    return <ErrorDisplay error={error} />;
  }

  const posts = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <div className="feed">
      <FeedFilter value={filter} onChange={setFilter} />

      <CreatePostComposer userId={userId} />

      <div className="feed-items">
        {posts.map((item) => (
          <FeedItemComponent key={item.id} item={item} />
        ))}
      </div>

      {hasNextPage && (
        <div ref={loadMoreRef} className="load-more-trigger">
          {isFetchingNextPage && <Spinner />}
        </div>
      )}

      {!hasNextPage && posts.length > 0 && (
        <div className="end-of-feed">You're all caught up!</div>
      )}
    </div>
  );
};
```

### Virtual Scrolling for Performance

```typescript
import { useVirtual } from 'react-virtual';

interface VirtualFeedProps {
  items: FeedItem[];
  onLoadMore: () => void;
  hasMore: boolean;
}

export const VirtualFeed: React.FC<VirtualFeedProps> = ({
  items,
  onLoadMore,
  hasMore,
}) => {
  const parentRef = useRef<HTMLDivElement>(null);
  const [estimateSize, setEstimateSize] = useState(400);

  const rowVirtualizer = useVirtual({
    size: items.length,
    parentRef,
    estimateSize: useCallback(() => estimateSize, [estimateSize]),
    overscan: 5,
  });

  useEffect(() => {
    const [lastItem] = [...rowVirtualizer.virtualItems].reverse();

    if (!lastItem) return;

    if (lastItem.index >= items.length - 1 && hasMore) {
      onLoadMore();
    }
  }, [rowVirtualizer.virtualItems, items.length, hasMore, onLoadMore]);

  return (
    <div ref={parentRef} className="virtual-feed" style={{ height: '100vh', overflow: 'auto' }}>
      <div
        style={{
          height: `${rowVirtualizer.totalSize}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rowVirtualizer.virtualItems.map((virtualRow) => {
          const item = items[virtualRow.index];

          return (
            <div
              key={virtualRow.index}
              ref={virtualRow.measureRef}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <FeedItemComponent item={item} />
            </div>
          );
        })}
      </div>
    </div>
  );
};
```

## Optimistic Updates

### Optimistic UI Implementation

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface UseOptimisticPostOptions {
  userId: string;
  onSuccess?: (post: Post) => void;
  onError?: (error: Error) => void;
}

export function useOptimisticPost({ userId, onSuccess, onError }: UseOptimisticPostOptions) {
  const queryClient = useQueryClient();
  const feedService = new FeedService(apiClient, cacheManager);

  const createPostMutation = useMutation({
    mutationFn: (data: CreatePostData) => feedService.createPost(data),

    onMutate: async (newPost) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['feed', userId] });

      // Snapshot previous value
      const previousFeed = queryClient.getQueryData(['feed', userId]);

      // Optimistically update to the new value
      queryClient.setQueryData<InfiniteData<PaginatedResponse<FeedItem>>>(
        ['feed', userId],
        (old) => {
          if (!old) return old;

          const optimisticPost: Post = {
            id: `temp-${Date.now()}`,
            userId,
            user: getCurrentUser(),
            content: newPost.content,
            mediaUrls: [],
            visibility: newPost.visibility,
            reactions: {
              total: 0,
              byType: {},
            },
            commentsCount: 0,
            sharesCount: 0,
            createdAt: new Date(),
            updatedAt: new Date(),
            isEdited: false,
          };

          const firstPage = old.pages[0];
          const newPages = [
            {
              ...firstPage,
              items: [
                { id: optimisticPost.id, type: 'post' as const, data: optimisticPost, priority: 1 },
                ...firstPage.items,
              ],
            },
            ...old.pages.slice(1),
          ];

          return {
            ...old,
            pages: newPages,
          };
        }
      );

      return { previousFeed };
    },

    onError: (err, newPost, context) => {
      // Rollback on error
      if (context?.previousFeed) {
        queryClient.setQueryData(['feed', userId], context.previousFeed);
      }
      onError?.(err as Error);
    },

    onSuccess: (post) => {
      onSuccess?.(post);
    },

    onSettled: () => {
      // Refetch after mutation
      queryClient.invalidateQueries({ queryKey: ['feed', userId] });
    },
  });

  const deletePostMutation = useMutation({
    mutationFn: (postId: string) => feedService.deletePost(postId),

    onMutate: async (postId) => {
      await queryClient.cancelQueries({ queryKey: ['feed', userId] });

      const previousFeed = queryClient.getQueryData(['feed', userId]);

      // Optimistically remove post
      queryClient.setQueryData<InfiniteData<PaginatedResponse<FeedItem>>>(
        ['feed', userId],
        (old) => {
          if (!old) return old;

          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              items: page.items.filter((item) => item.id !== postId),
            })),
          };
        }
      );

      return { previousFeed };
    },

    onError: (err, postId, context) => {
      if (context?.previousFeed) {
        queryClient.setQueryData(['feed', userId], context.previousFeed);
      }
      onError?.(err as Error);
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['feed', userId] });
    },
  });

  return {
    createPost: createPostMutation.mutate,
    deletePost: deletePostMutation.mutate,
    isCreating: createPostMutation.isLoading,
    isDeleting: deletePostMutation.isLoading,
  };
}

// Usage in component
export const CreatePostComposer: React.FC<{ userId: string }> = ({ userId }) => {
  const [content, setContent] = useState('');
  const [visibility, setVisibility] = useState<PostVisibility>('public');
  const { createPost, isCreating } = useOptimisticPost({
    userId,
    onSuccess: () => {
      setContent('');
      toast.success('Post created!');
    },
    onError: (error) => {
      toast.error('Failed to create post');
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    if (!content.trim()) return;

    createPost({
      userId,
      content,
      visibility,
    });
  };

  return (
    <form onSubmit={handleSubmit} className="create-post-composer">
      <textarea
        value={content}
        onChange={(e) => setContent(e.target.value)}
        placeholder="What's on your mind?"
        disabled={isCreating}
      />

      <div className="composer-actions">
        <VisibilitySelector value={visibility} onChange={setVisibility} />
        <button type="submit" disabled={isCreating || !content.trim()}>
          {isCreating ? 'Posting...' : 'Post'}
        </button>
      </div>
    </form>
  );
};
```

## Reactions System

### Reaction Types and Service

```typescript
class ReactionService {
  constructor(private apiClient: ApiClient) {}

  async addReaction(
    targetType: "post" | "comment",
    targetId: string,
    reactionType: ReactionType,
  ): Promise<ReactionsSummary> {
    return await this.apiClient.post<ReactionsSummary>(`/reactions`, {
      targetType,
      targetId,
      reactionType,
    });
  }

  async removeReaction(
    targetType: "post" | "comment",
    targetId: string,
  ): Promise<ReactionsSummary> {
    return await this.apiClient.delete<ReactionsSummary>(
      `/reactions/${targetType}/${targetId}`,
    );
  }

  async getReactions(
    targetType: "post" | "comment",
    targetId: string,
  ): Promise<Reaction[]> {
    return await this.apiClient.get<Reaction[]>(
      `/reactions/${targetType}/${targetId}`,
    );
  }
}

interface Reaction {
  id: string;
  userId: string;
  user: User;
  targetType: "post" | "comment";
  targetId: string;
  reactionType: ReactionType;
  createdAt: Date;
}

// Optimistic reaction hook
export function useOptimisticReaction(
  targetType: "post" | "comment",
  targetId: string,
) {
  const queryClient = useQueryClient();
  const reactionService = new ReactionService(apiClient);

  const addReactionMutation = useMutation({
    mutationFn: (reactionType: ReactionType) =>
      reactionService.addReaction(targetType, targetId, reactionType),

    onMutate: async (reactionType) => {
      await queryClient.cancelQueries({ queryKey: ["feed"] });

      const previousData = queryClient.getQueryData(["feed"]);

      // Optimistically update reactions
      queryClient.setQueriesData<InfiniteData<PaginatedResponse<FeedItem>>>(
        { queryKey: ["feed"] },
        (old) => {
          if (!old) return old;

          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              items: page.items.map((item) => {
                if (item.type === "post" && item.data.id === targetId) {
                  const post = item.data as Post;
                  return {
                    ...item,
                    data: {
                      ...post,
                      reactions: {
                        ...post.reactions,
                        total: post.reactions.total + 1,
                        byType: {
                          ...post.reactions.byType,
                          [reactionType]:
                            (post.reactions.byType[reactionType] || 0) + 1,
                        },
                        userReaction: reactionType,
                      },
                    },
                  };
                }
                return item;
              }),
            })),
          };
        },
      );

      return { previousData };
    },

    onError: (err, reactionType, context) => {
      if (context?.previousData) {
        queryClient.setQueryData(["feed"], context.previousData);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["feed"] });
    },
  });

  const removeReactionMutation = useMutation({
    mutationFn: () => reactionService.removeReaction(targetType, targetId),

    onMutate: async () => {
      await queryClient.cancelQueries({ queryKey: ["feed"] });

      const previousData = queryClient.getQueryData(["feed"]);

      queryClient.setQueriesData<InfiniteData<PaginatedResponse<FeedItem>>>(
        { queryKey: ["feed"] },
        (old) => {
          if (!old) return old;

          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              items: page.items.map((item) => {
                if (item.type === "post" && item.data.id === targetId) {
                  const post = item.data as Post;
                  const userReaction = post.reactions.userReaction!;

                  return {
                    ...item,
                    data: {
                      ...post,
                      reactions: {
                        ...post.reactions,
                        total: Math.max(0, post.reactions.total - 1),
                        byType: {
                          ...post.reactions.byType,
                          [userReaction]: Math.max(
                            0,
                            (post.reactions.byType[userReaction] || 0) - 1,
                          ),
                        },
                        userReaction: undefined,
                      },
                    },
                  };
                }
                return item;
              }),
            })),
          };
        },
      );

      return { previousData };
    },

    onError: (err, _, context) => {
      if (context?.previousData) {
        queryClient.setQueryData(["feed"], context.previousData);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["feed"] });
    },
  });

  return {
    addReaction: addReactionMutation.mutate,
    removeReaction: removeReactionMutation.mutate,
    isUpdating:
      addReactionMutation.isLoading || removeReactionMutation.isLoading,
  };
}
```

### Reaction Picker Component

```typescript
export const ReactionPicker: React.FC<{
  targetType: 'post' | 'comment';
  targetId: string;
  currentReaction?: ReactionType;
}> = ({ targetType, targetId, currentReaction }) => {
  const [showPicker, setShowPicker] = useState(false);
  const { addReaction, removeReaction, isUpdating } = useOptimisticReaction(
    targetType,
    targetId
  );

  const reactions: Array<{ type: ReactionType; emoji: string; label: string }> = [
    { type: 'like', emoji: '👍', label: 'Like' },
    { type: 'love', emoji: '❤️', label: 'Love' },
    { type: 'haha', emoji: '😄', label: 'Haha' },
    { type: 'wow', emoji: '😮', label: 'Wow' },
    { type: 'sad', emoji: '😢', label: 'Sad' },
    { type: 'angry', emoji: '😠', label: 'Angry' },
  ];

  const handleReactionClick = (reactionType: ReactionType) => {
    if (currentReaction === reactionType) {
      removeReaction();
    } else {
      addReaction(reactionType);
    }
    setShowPicker(false);
  };

  return (
    <div className="reaction-picker-wrapper">
      <button
        className={`reaction-button ${currentReaction ? 'reacted' : ''}`}
        onMouseEnter={() => setShowPicker(true)}
        onMouseLeave={() => setShowPicker(false)}
        onClick={() => {
          if (currentReaction) {
            removeReaction();
          } else {
            addReaction('like');
          }
        }}
        disabled={isUpdating}
      >
        {currentReaction ? (
          <>
            <span className="reaction-emoji">
              {reactions.find((r) => r.type === currentReaction)?.emoji}
            </span>
            <span>{currentReaction}</span>
          </>
        ) : (
          <>
            <ThumbsUpIcon />
            <span>Like</span>
          </>
        )}
      </button>

      {showPicker && (
        <div
          className="reaction-picker"
          onMouseEnter={() => setShowPicker(true)}
          onMouseLeave={() => setShowPicker(false)}
        >
          {reactions.map((reaction) => (
            <button
              key={reaction.type}
              className="reaction-option"
              onClick={() => handleReactionClick(reaction.type)}
              title={reaction.label}
            >
              <span className="reaction-emoji">{reaction.emoji}</span>
            </button>
          ))}
        </div>
      )}
    </div>
  );
};
```

## Comments System

### Comment Service

```typescript
class CommentService {
  constructor(private apiClient: ApiClient) {}

  async getComments(
    postId: string,
    params: CommentParams,
  ): Promise<PaginatedResponse<Comment>> {
    return await this.apiClient.get<PaginatedResponse<Comment>>(
      `/posts/${postId}/comments`,
      { params },
    );
  }

  async createComment(data: CreateCommentData): Promise<Comment> {
    return await this.apiClient.post<Comment>(
      `/posts/${data.postId}/comments`,
      data,
    );
  }

  async updateComment(commentId: string, content: string): Promise<Comment> {
    return await this.apiClient.put<Comment>(`/comments/${commentId}`, {
      content,
    });
  }

  async deleteComment(commentId: string): Promise<void> {
    await this.apiClient.delete(`/comments/${commentId}`);
  }

  async getReplies(
    commentId: string,
    params: CommentParams,
  ): Promise<PaginatedResponse<Comment>> {
    return await this.apiClient.get<PaginatedResponse<Comment>>(
      `/comments/${commentId}/replies`,
      { params },
    );
  }
}

interface CommentParams {
  cursor?: string;
  limit?: number;
  sort?: "newest" | "oldest" | "top";
}

interface CreateCommentData {
  postId: string;
  content: string;
  parentId?: string;
}

// Comments hook
export function useComments(
  postId: string,
  sort: "newest" | "oldest" | "top" = "newest",
) {
  const commentService = new CommentService(apiClient);

  return useInfiniteQuery({
    queryKey: ["comments", postId, sort],
    queryFn: ({ pageParam }) =>
      commentService.getComments(postId, {
        cursor: pageParam,
        limit: 20,
        sort,
      }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });
}

export function useOptimisticComment(postId: string) {
  const queryClient = useQueryClient();
  const commentService = new CommentService(apiClient);

  const createCommentMutation = useMutation({
    mutationFn: (data: CreateCommentData) => commentService.createComment(data),

    onMutate: async (newComment) => {
      await queryClient.cancelQueries({ queryKey: ["comments", postId] });

      const previousComments = queryClient.getQueryData(["comments", postId]);

      queryClient.setQueryData<InfiniteData<PaginatedResponse<Comment>>>(
        ["comments", postId],
        (old) => {
          if (!old) return old;

          const optimisticComment: Comment = {
            id: `temp-${Date.now()}`,
            postId,
            userId: getCurrentUser().id,
            user: getCurrentUser(),
            content: newComment.content,
            parentId: newComment.parentId,
            reactions: {
              total: 0,
              byType: {},
            },
            repliesCount: 0,
            createdAt: new Date(),
            updatedAt: new Date(),
            isEdited: false,
          };

          const firstPage = old.pages[0];
          return {
            ...old,
            pages: [
              {
                ...firstPage,
                items: [optimisticComment, ...firstPage.items],
              },
              ...old.pages.slice(1),
            ],
          };
        },
      );

      // Update comment count on post
      queryClient.setQueriesData<InfiniteData<PaginatedResponse<FeedItem>>>(
        { queryKey: ["feed"] },
        (old) => {
          if (!old) return old;

          return {
            ...old,
            pages: old.pages.map((page) => ({
              ...page,
              items: page.items.map((item) => {
                if (item.type === "post" && item.data.id === postId) {
                  const post = item.data as Post;
                  return {
                    ...item,
                    data: {
                      ...post,
                      commentsCount: post.commentsCount + 1,
                    },
                  };
                }
                return item;
              }),
            })),
          };
        },
      );

      return { previousComments };
    },

    onError: (err, newComment, context) => {
      if (context?.previousComments) {
        queryClient.setQueryData(
          ["comments", postId],
          context.previousComments,
        );
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["comments", postId] });
      queryClient.invalidateQueries({ queryKey: ["feed"] });
    },
  });

  return {
    createComment: createCommentMutation.mutate,
    isCreating: createCommentMutation.isLoading,
  };
}
```

### Comment Component

```typescript
export const CommentSection: React.FC<{ postId: string }> = ({ postId }) => {
  const [sortBy, setSortBy] = useState<'newest' | 'oldest' | 'top'>('newest');
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useComments(
    postId,
    sortBy
  );

  const comments = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <div className="comment-section">
      <div className="comment-section-header">
        <h3>Comments ({comments.length})</h3>
        <SortSelector value={sortBy} onChange={setSortBy} />
      </div>

      <CommentComposer postId={postId} />

      <div className="comments-list">
        {comments.map((comment) => (
          <CommentItem key={comment.id} comment={comment} />
        ))}
      </div>

      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? 'Loading...' : 'Load More Comments'}
        </button>
      )}
    </div>
  );
};

const CommentItem: React.FC<{ comment: Comment }> = ({ comment }) => {
  const [showReplies, setShowReplies] = useState(false);
  const [isReplying, setIsReplying] = useState(false);

  return (
    <div className="comment-item">
      <div className="comment-avatar">
        <img src={comment.user.avatarUrl} alt={comment.user.displayName} />
      </div>

      <div className="comment-content">
        <div className="comment-header">
          <span className="comment-author">{comment.user.displayName}</span>
          <span className="comment-timestamp">
            {formatTimeAgo(comment.createdAt)}
          </span>
        </div>

        <p className="comment-text">{comment.content}</p>

        <div className="comment-actions">
          <ReactionPicker
            targetType="comment"
            targetId={comment.id}
            currentReaction={comment.reactions.userReaction}
          />
          <button onClick={() => setIsReplying(!isReplying)}>Reply</button>
          {comment.repliesCount > 0 && (
            <button onClick={() => setShowReplies(!showReplies)}>
              {showReplies ? 'Hide' : 'View'} {comment.repliesCount} replies
            </button>
          )}
        </div>

        {isReplying && (
          <CommentComposer
            postId={comment.postId}
            parentId={comment.id}
            onSuccess={() => setIsReplying(false)}
          />
        )}

        {showReplies && <CommentReplies commentId={comment.id} />}
      </div>
    </div>
  );
};

const CommentComposer: React.FC<{
  postId: string;
  parentId?: string;
  onSuccess?: () => void;
}> = ({ postId, parentId, onSuccess }) => {
  const [content, setContent] = useState('');
  const { createComment, isCreating } = useOptimisticComment(postId);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    if (!content.trim()) return;

    createComment(
      { postId, content, parentId },
      {
        onSuccess: () => {
          setContent('');
          onSuccess?.();
        },
      }
    );
  };

  return (
    <form onSubmit={handleSubmit} className="comment-composer">
      <textarea
        value={content}
        onChange={(e) => setContent(e.target.value)}
        placeholder="Write a comment..."
        disabled={isCreating}
      />
      <button type="submit" disabled={isCreating || !content.trim()}>
        {isCreating ? 'Posting...' : 'Post'}
      </button>
    </form>
  );
};
```

## Real-Time Updates

### WebSocket Integration

```typescript
class FeedWebSocketManager {
  private ws: WebSocket | null = null;
  private listeners: Map<string, Set<(event: FeedEvent) => void>> = new Map();

  connect(userId: string): void {
    this.ws = new WebSocket(`${WS_URL}?userId=${userId}`);

    this.ws.onmessage = (message) => {
      const event: FeedEvent = JSON.parse(message.data);
      this.handleEvent(event);
    };

    this.ws.onclose = () => {
      setTimeout(() => this.connect(userId), 3000);
    };
  }

  subscribe(
    eventType: FeedEventType,
    callback: (event: FeedEvent) => void,
  ): () => void {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, new Set());
    }

    this.listeners.get(eventType)!.add(callback);

    return () => {
      this.listeners.get(eventType)?.delete(callback);
    };
  }

  private handleEvent(event: FeedEvent): void {
    const callbacks = this.listeners.get(event.type);
    if (callbacks) {
      callbacks.forEach((cb) => cb(event));
    }
  }

  disconnect(): void {
    this.ws?.close();
    this.listeners.clear();
  }
}

type FeedEventType =
  | "new_post"
  | "post_updated"
  | "post_deleted"
  | "new_comment"
  | "new_reaction";

interface FeedEvent {
  type: FeedEventType;
  data: any;
  timestamp: Date;
}

// React hook for real-time updates
export function useFeedRealtime(userId: string) {
  const queryClient = useQueryClient();
  const wsManager = useRef(new FeedWebSocketManager());

  useEffect(() => {
    wsManager.current.connect(userId);

    const unsubscribeNewPost = wsManager.current.subscribe(
      "new_post",
      (event) => {
        // Show notification for new post
        toast.info("New post available");

        // Optionally prepend to feed
        queryClient.setQueryData<InfiniteData<PaginatedResponse<FeedItem>>>(
          ["feed", userId],
          (old) => {
            if (!old) return old;

            const firstPage = old.pages[0];
            return {
              ...old,
              pages: [
                {
                  ...firstPage,
                  items: [event.data, ...firstPage.items],
                },
                ...old.pages.slice(1),
              ],
            };
          },
        );
      },
    );

    const unsubscribePostDeleted = wsManager.current.subscribe(
      "post_deleted",
      (event) => {
        queryClient.setQueryData<InfiniteData<PaginatedResponse<FeedItem>>>(
          ["feed", userId],
          (old) => {
            if (!old) return old;

            return {
              ...old,
              pages: old.pages.map((page) => ({
                ...page,
                items: page.items.filter(
                  (item) => item.id !== event.data.postId,
                ),
              })),
            };
          },
        );
      },
    );

    return () => {
      unsubscribeNewPost();
      unsubscribePostDeleted();
      wsManager.current.disconnect();
    };
  }, [userId, queryClient]);
}
```

## Media Handling

### Image Upload and Processing

```typescript
class MediaService {
  constructor(
    private apiClient: ApiClient,
    private cdnUrl: string
  ) {}

  async uploadImage(file: File): Promise<MediaItem> {
    // Validate file
    this.validateImage(file);

    // Compress image
    const compressedFile = await this.compressImage(file);

    // Upload to server
    const formData = new FormData();
    formData.append('file', compressedFile);

    const response = await this.apiClient.post<UploadResponse>(
      '/media/upload',
      formData,
      {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
        onUploadProgress: (progressEvent) => {
          const percentCompleted = Math.round(
            (progressEvent.loaded * 100) / (progressEvent.total || 1)
          );
          console.log('Upload progress:', percentCompleted);
        },
      }
    );

    return {
      id: response.id,
      type: 'image',
      url: `${this.cdnUrl}/${response.path}`,
      thumbnailUrl: `${this.cdnUrl}/${response.thumbnailPath}`,
      width: response.width,
      height: response.height,
    };
  }

  private validateImage(file: File): void {
    const maxSize = 10 * 1024 * 1024; // 10MB
    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/gif'];

    if (!allowedTypes.includes(file.type)) {
      throw new Error('Invalid file type');
    }

    if (file.size > maxSize) {
      throw new Error('File too large');
    }
  }

  private async compressImage(file: File): Promise<File> {
    const options = {
      maxSizeMB: 1,
      maxWidthOrHeight: 1920,
      useWebWorker: true,
    };

    return await imageCompression(file, options);
  }

  async uploadVideo(file: File, onProgress?: (progress: number) => void): Promise<MediaItem> {
    // Similar to image upload but with video-specific handling
    const formData = new FormData();
    formData.append('file', file);

    const response = await this.apiClient.post<UploadResponse>(
      '/media/upload-video',
      formData,
      {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
        onUploadProgress: (progressEvent) => {
          const percentCompleted = Math.round(
            (progressEvent.loaded * 100) / (progressEvent.total || 1)
          );
          onProgress?.(percentCompleted);
        },
      }
    );

    return {
      id: response.id,
      type: 'video',
      url: `${this.cdnUrl}/${response.path}`,
      thumbnailUrl: `${this.cdnUrl}/${response.thumbnailPath}`,
      width: response.width,
      height: response.height,
      duration: response.duration,
    };
  }
}

interface UploadResponse {
  id: string;
  path: string;
  thumbnailPath?: string;
  width: number;
  height: number;
  duration?: number;
}

// Media upload component
export const MediaUploader: React.FC<{
  onUpload: (media: MediaItem) => void;
}> = ({ onUpload }) => {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  const mediaService = new MediaService(apiClient, CDN_URL);

  const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    setUploading(true);
    setProgress(0);

    try {
      let media: MediaItem;

      if (file.type.startsWith('image/')) {
        media = await mediaService.uploadImage(file);
      } else if (file.type.startsWith('video/')) {
        media = await mediaService.uploadVideo(file, setProgress);
      } else {
        throw new Error('Unsupported file type');
      }

      onUpload(media);
      toast.success('Media uploaded successfully');
    } catch (error) {
      toast.error('Failed to upload media');
      console.error(error);
    } finally {
      setUploading(false);
      setProgress(0);
    }
  };

  return (
    <div className="media-uploader">
      <input
        type="file"
        accept="image/*,video/*"
        onChange={handleFileChange}
        disabled={uploading}
        style={{ display: 'none' }}
        id="media-upload"
      />
      <label htmlFor="media-upload" className="upload-button">
        <PhotoIcon />
        {uploading ? `Uploading... ${progress}%` : 'Add Photo/Video'}
      </label>
    </div>
  );
};
```

## Performance Optimization

### Code Splitting and Lazy Loading

```typescript
// Lazy load heavy components
const VideoPlayer = lazy(() => import('./VideoPlayer'));
const ImageGallery = lazy(() => import('./ImageGallery'));
const CommentSection = lazy(() => import('./CommentSection'));

export const FeedItemComponent: React.FC<{ item: FeedItem }> = ({ item }) => {
  const [showComments, setShowComments] = useState(false);

  if (item.type !== 'post') return null;

  const post = item.data as Post;

  return (
    <article className="feed-item">
      <PostHeader post={post} />
      <PostContent content={post.content} />

      {post.mediaUrls.length > 0 && (
        <Suspense fallback={<MediaSkeleton />}>
          {post.mediaUrls[0].type === 'video' ? (
            <VideoPlayer media={post.mediaUrls[0]} />
          ) : (
            <ImageGallery media={post.mediaUrls} />
          )}
        </Suspense>
      )}

      <PostActions post={post} onCommentClick={() => setShowComments(!showComments)} />

      {showComments && (
        <Suspense fallback={<CommentsSkeleton />}>
          <CommentSection postId={post.id} />
        </Suspense>
      )}
    </article>
  );
};
```

### Image Lazy Loading

```typescript
export const LazyImage: React.FC<{
  src: string;
  alt: string;
  aspectRatio?: number;
}> = ({ src, alt, aspectRatio = 16 / 9 }) => {
  const [loaded, setLoaded] = useState(false);
  const [inView, setInView] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);

  useEffect(() => {
    if (!imgRef.current) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setInView(true);
          observer.disconnect();
        }
      },
      { rootMargin: '50px' }
    );

    observer.observe(imgRef.current);

    return () => observer.disconnect();
  }, []);

  return (
    <div
      className="lazy-image-wrapper"
      style={{ paddingTop: `${(1 / aspectRatio) * 100}%` }}
    >
      {inView && (
        <>
          {!loaded && <ImageSkeleton />}
          <img
            ref={imgRef}
            src={src}
            alt={alt}
            onLoad={() => setLoaded(true)}
            className={loaded ? 'loaded' : 'loading'}
          />
        </>
      )}
    </div>
  );
};
```

## Key Takeaways

### 1. **Infinite Scroll Performance**

Use React Query's `useInfiniteQuery` for efficient pagination. Implement virtual scrolling for large feeds. Prefetch next page before user reaches end.

### 2. **Optimistic UI Updates**

Immediately reflect user actions in UI before server confirmation. Always have rollback strategy for failed operations. Show clear feedback for pending states.

### 3. **Efficient Reactions**

Batch reaction updates to reduce API calls. Use optimistic updates for instant feedback. Cache reaction summaries at post level.

### 4. **Smart Comment Loading**

Load comments on demand, not with initial feed. Support nested replies with lazy loading. Implement comment pagination for popular posts.

### 5. **Real-Time Capabilities**

Use WebSocket for live updates. Show notifications for new content. Allow users to manually refresh to see new posts.

### 6. **Media Optimization**

Compress images on client before upload. Generate multiple sizes on server. Use CDN for media delivery. Implement progressive image loading.

### 7. **Feed Algorithm**

Rank posts by relevance, recency, and engagement. Personalize based on user interests. Mix content types to maintain engagement.

### 8. **Caching Strategy**

Cache feed data with short TTL. Invalidate cache on user actions. Use optimistic updates to avoid cache staleness.

### 9. **Accessibility**

Support keyboard navigation for all interactions. Provide ARIA labels for screen readers. Ensure color contrast for reactions.

### 10. **Mobile Optimization**

Implement pull-to-refresh on mobile. Optimize for touch interactions. Reduce bundle size with code splitting. Support offline reading.

---

**Further Reading:**

- React Query Documentation
- WebSocket Best Practices
- Image Optimization Techniques
- Feed Ranking Algorithms
