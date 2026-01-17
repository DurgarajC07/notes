# Django Views - Function-Based vs Class-Based

## 📖 Concept Explanation

Views are Python functions or classes that receive web requests and return web responses. Django supports two approaches: Function-Based Views (FBV) and Class-Based Views (CBV).

### View Flow

```
URL → View → Template/Response
```

**Process**:

1. URL dispatcher routes request to view
2. View processes request (database queries, business logic)
3. Returns HttpResponse (rendered template, JSON, redirect, etc.)

### Function-Based View (FBV)

```python
from django.shortcuts import render, get_object_or_404
from django.http import HttpResponse
from .models import Post

def post_list(request):
    """Simple FBV"""
    posts = Post.objects.all()
    return render(request, 'blog/post_list.html', {'posts': posts})

def post_detail(request, pk):
    """FBV with parameter"""
    post = get_object_or_404(Post, pk=pk)
    return render(request, 'blog/post_detail.html', {'post': post})
```

### Class-Based View (CBV)

```python
from django.views import View
from django.views.generic import ListView, DetailView
from .models import Post

# Generic CBV
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'

class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
```

## 🧠 Why It Matters

**Advantages of CBV**:

- Code reusability through inheritance
- Built-in generic views for common patterns
- Mixins for cross-cutting concerns
- Cleaner separation of HTTP methods

**Advantages of FBV**:

- Simpler for beginners
- More explicit control flow
- Easier to debug
- Better for unique/complex logic

## 🏗️ Function-Based Views (FBV)

### 1. Basic CRUD Operations

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.http import JsonResponse
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods
from .models import Post
from .forms import PostForm

# LIST
@login_required
def post_list(request):
    posts = Post.objects.select_related('author').all()
    return render(request, 'blog/post_list.html', {'posts': posts})

# CREATE
@login_required
@require_http_methods(['GET', 'POST'])
def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm()

    return render(request, 'blog/post_form.html', {'form': form})

# UPDATE
@login_required
def post_update(request, pk):
    post = get_object_or_404(Post, pk=pk)

    # Authorization check
    if post.author != request.user:
        return HttpResponseForbidden("You don't have permission to edit this post")

    if request.method == 'POST':
        form = PostForm(request.POST, instance=post)
        if form.is_valid():
            form.save()
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm(instance=post)

    return render(request, 'blog/post_form.html', {'form': form, 'post': post})

# DELETE
@login_required
@require_http_methods(['POST'])
def post_delete(request, pk):
    post = get_object_or_404(Post, pk=pk)

    if post.author != request.user:
        return HttpResponseForbidden("You don't have permission to delete this post")

    post.delete()
    return redirect('post_list')

# DETAIL
def post_detail(request, pk):
    post = get_object_or_404(Post.objects.select_related('author'), pk=pk)
    comments = post.comments.select_related('author').all()

    return render(request, 'blog/post_detail.html', {
        'post': post,
        'comments': comments
    })
```

### 2. API Endpoint (FBV)

```python
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods
import json

@csrf_exempt  # Use JWT/Token auth in production
@require_http_methods(['GET', 'POST'])
def api_posts(request):
    if request.method == 'GET':
        # List posts
        posts = Post.objects.values('id', 'title', 'content', 'created_at')
        return JsonResponse({'posts': list(posts)})

    elif request.method == 'POST':
        # Create post
        try:
            data = json.loads(request.body)
            post = Post.objects.create(
                title=data['title'],
                content=data['content'],
                author=request.user
            )
            return JsonResponse({
                'id': post.id,
                'title': post.title,
                'content': post.content
            }, status=201)
        except (KeyError, json.JSONDecodeError) as e:
            return JsonResponse({'error': str(e)}, status=400)
```

## 🏗️ Class-Based Views (CBV)

### 1. Generic Views

```python
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView
)
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.urls import reverse_lazy
from .models import Post

# LIST VIEW
class PostListView(ListView):
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 20
    ordering = ['-created_at']

    def get_queryset(self):
        """Optimize with select_related"""
        return Post.objects.select_related('author').all()

    def get_context_data(self, **kwargs):
        """Add extra context"""
        context = super().get_context_data(**kwargs)
        context['total_posts'] = Post.objects.count()
        return context

# DETAIL VIEW
class PostDetailView(DetailView):
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_queryset(self):
        return Post.objects.select_related('author').prefetch_related('comments__author')

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['comments'] = self.object.comments.select_related('author').all()
        return context

# CREATE VIEW
class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'
    success_url = reverse_lazy('post_list')

    def form_valid(self, form):
        """Set author before saving"""
        form.instance.author = self.request.user
        return super().form_valid(form)

# UPDATE VIEW
class PostUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'

    def test_func(self):
        """Authorization: only author can edit"""
        post = self.get_object()
        return self.request.user == post.author

    def get_success_url(self):
        return reverse_lazy('post_detail', kwargs={'pk': self.object.pk})

# DELETE VIEW
class PostDeleteView(LoginRequiredMixin, UserPassesTestMixin, DeleteView):
    model = Post
    success_url = reverse_lazy('post_list')

    def test_func(self):
        post = self.get_object()
        return self.request.user == post.author
```

### 2. Custom View Class

```python
from django.views import View
from django.shortcuts import render, redirect
from django.http import JsonResponse

class PostView(View):
    """Handle multiple HTTP methods"""

    def get(self, request, pk=None):
        if pk:
            # Detail view
            post = get_object_or_404(Post, pk=pk)
            return render(request, 'blog/post_detail.html', {'post': post})
        else:
            # List view
            posts = Post.objects.all()
            return render(request, 'blog/post_list.html', {'posts': posts})

    def post(self, request):
        # Create post
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('post_detail', pk=post.pk)
        return render(request, 'blog/post_form.html', {'form': form})

    def put(self, request, pk):
        # Update post (usually for API)
        post = get_object_or_404(Post, pk=pk)
        # Process PUT data...
        return JsonResponse({'status': 'updated'})

    def delete(self, request, pk):
        # Delete post
        post = get_object_or_404(Post, pk=pk)
        if post.author == request.user:
            post.delete()
            return JsonResponse({'status': 'deleted'})
        return JsonResponse({'error': 'forbidden'}, status=403)
```

## 🎯 Mixins for Reusability

### 1. Custom Mixins

```python
from django.http import JsonResponse
from django.contrib.auth.mixins import AccessMixin

class JSONResponseMixin:
    """Mixin to add JSON response capability"""

    def render_to_json_response(self, context, **response_kwargs):
        return JsonResponse(context, **response_kwargs)

class AjaxRequiredMixin:
    """Mixin to require AJAX requests"""

    def dispatch(self, request, *args, **kwargs):
        if not request.headers.get('X-Requested-With') == 'XMLHttpRequest':
            return JsonResponse({'error': 'AJAX required'}, status=400)
        return super().dispatch(request, *args, **kwargs)

class AuthorRequiredMixin(AccessMixin):
    """Mixin to verify user is the author"""

    def dispatch(self, request, *args, **kwargs):
        obj = self.get_object()
        if obj.author != request.user:
            return self.handle_no_permission()
        return super().dispatch(request, *args, **kwargs)
```

### 2. Using Mixins

```python
class PostUpdateView(LoginRequiredMixin, AuthorRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'
    success_url = reverse_lazy('post_list')

class PostAPIView(JSONResponseMixin, View):
    def get(self, request, pk):
        post = get_object_or_404(Post, pk=pk)
        data = {
            'id': post.id,
            'title': post.title,
            'content': post.content,
        }
        return self.render_to_json_response(data)
```

## ✅ Best Practices

### 1. When to Use FBV vs CBV

```python
# Use FBV for:
# - Simple, unique logic
# - Complex workflows
# - API endpoints with custom behavior
# - When clarity is more important than reusability

@login_required
def complex_report(request):
    # Complex multi-step processing
    step1 = process_data()
    step2 = calculate_metrics(step1)
    step3 = generate_charts(step2)
    return render(request, 'report.html', {'data': step3})

# Use CBV for:
# - Standard CRUD operations
# - When you need inheritance/mixins
# - DRY principle with similar views
# - Built-in generic views fit your needs

class ProductListView(ListView):
    model = Product
    template_name = 'products/list.html'
    paginate_by = 20
```

### 2. Proper Error Handling

```python
from django.http import Http404, HttpResponseBadRequest
from django.db import transaction

def post_create(request):
    if request.method != 'POST':
        return HttpResponseBadRequest("Only POST allowed")

    try:
        with transaction.atomic():
            # Create post and related objects
            post = Post.objects.create(
                title=request.POST['title'],
                content=request.POST['content'],
                author=request.user
            )
            # Create tags
            tag_names = request.POST.getlist('tags')
            for tag_name in tag_names:
                Tag.objects.create(post=post, name=tag_name)

            return redirect('post_detail', pk=post.pk)

    except KeyError as e:
        return HttpResponseBadRequest(f"Missing field: {e}")
    except Exception as e:
        logger.error(f"Error creating post: {e}")
        return HttpResponseServerError("Internal error")
```

## 🔐 Security Best Practices

### 1. Authorization in Views

```python
from django.core.exceptions import PermissionDenied

class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post

    def dispatch(self, request, *args, **kwargs):
        post = self.get_object()
        if post.author != request.user and not request.user.is_staff:
            raise PermissionDenied("You don't have permission")
        return super().dispatch(request, *args, **kwargs)
```

### 2. Input Validation

```python
def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            # Form validation passed - data is clean
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            return redirect('post_detail', pk=post.pk)
        else:
            # Form validation failed
            return render(request, 'blog/post_form.html', {
                'form': form,
                'errors': form.errors
            })
```

## ⚡ Performance Optimization

### 1. Query Optimization

```python
class PostListView(ListView):
    model = Post

    def get_queryset(self):
        return Post.objects.select_related('author') \
            .prefetch_related('tags') \
            .only('id', 'title', 'created_at', 'author__username') \
            .order_by('-created_at')
```

### 2. Caching

```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

# FBV caching
@cache_page(60 * 15)  # 15 minutes
def post_list(request):
    posts = Post.objects.all()
    return render(request, 'blog/post_list.html', {'posts': posts})

# CBV caching
@method_decorator(cache_page(60 * 15), name='dispatch')
class PostListView(ListView):
    model = Post
```

## ❓ Interview Questions

### Q1: FBV vs CBV - when to use each?

**Answer**:

- **FBV**: Simple logic, complex workflows, easy to understand
- **CBV**: Standard CRUD, code reuse, inheritance, mixins

Use FBV for unique logic, CBV for common patterns.

### Q2: What is the purpose of LoginRequiredMixin?

**Answer**:
Ensures user is authenticated before accessing the view. Redirects to login if not.

```python
class PostCreateView(LoginRequiredMixin, CreateView):
    login_url = '/login/'
    redirect_field_name = 'next'
```

### Q3: How do you pass extra context to a template in CBV?

**Answer**:
Override `get_context_data()`:

```python
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    context['extra_data'] = 'value'
    return context
```

## 📚 Summary

**Key Takeaways**:

1. FBV = simple, explicit; CBV = reusable, DRY
2. Use generic CBVs for standard CRUD
3. Mixins provide cross-cutting functionality
4. Always validate input and check permissions
5. Optimize queries with select_related/prefetch_related
6. Cache expensive views
7. Use method decorators for FBV features in CBV

Choose the right tool for the job - FBV or CBV!
