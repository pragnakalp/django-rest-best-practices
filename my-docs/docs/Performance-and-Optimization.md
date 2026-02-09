# Performance and Optimization Best Practices

## Overview
This document covers best practices for building high-performance Django REST Framework APIs that scale efficiently and provide fast response times.

## Database Query Optimization

### The N+1 Query Problem

#### Understanding the Problem
The N+1 query problem occurs when you fetch a list of objects and then access related objects for each item, causing N additional queries.

**Example of N+1 Problem:**
```python
# This causes N+1 queries
class ArticleSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.username')
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'author_name']

# View
articles = Article.objects.all()  # 1 query
serializer = ArticleSerializer(articles, many=True)
# For each article, accessing author.username causes 1 query = N queries
# Total: 1 + N queries
```

### Comparing N+1 Solutions: select_related() vs prefetch_related()

#### Approach 1: select_related() (SQL JOIN)

**Best For:** ForeignKey and OneToOne relationships

```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        # Single query with SQL JOIN
        return Article.objects.select_related('author', 'category')
```

**How It Works:**
- Creates SQL JOIN in single query
- Fetches related objects in one database hit
- Returns combined result set

**SQL Generated:**
```sql
SELECT article.*, author.*, category.*
FROM article
INNER JOIN user AS author ON article.author_id = author.id
INNER JOIN category ON article.category_id = category.id
```

**Pros:**
- Single database query
- Fastest for small to medium datasets
- Simple to understand
- Minimal memory overhead
- Works with filtering on related fields

**Cons:**
- Only works with ForeignKey/OneToOne
- Can create large result sets with multiple JOINs
- Cartesian product with multiple relationships
- Not suitable for ManyToMany

**Use When:**
- ForeignKey relationships (author, category)
- OneToOneField relationships (profile)
- Forward relationships only
- Need to filter on related fields
- Small to medium number of related objects

**Performance:**
```python
# Without select_related: 101 queries
articles = Article.objects.all()  # 1 query
for article in articles:  # 100 articles
    print(article.author.username)  # 100 queries
# Total: 101 queries, ~500ms

# With select_related: 1 query
articles = Article.objects.select_related('author').all()
for article in articles:
    print(article.author.username)  # No queries
# Total: 1 query, ~5ms
```

#### Approach 2: prefetch_related() (Separate Queries)

**Best For:** ManyToMany and reverse ForeignKey relationships

```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        # 2 separate queries, joined in Python
        return Article.objects.prefetch_related('tags', 'comments')
```

**How It Works:**
- Executes separate queries for each relationship
- Joins results in Python memory
- Caches related objects

**SQL Generated:**
```sql
-- Query 1: Get articles
SELECT * FROM article

-- Query 2: Get all related tags
SELECT * FROM tag
INNER JOIN article_tags ON tag.id = article_tags.tag_id
WHERE article_tags.article_id IN (1, 2, 3, ...)
```

**Pros:**
- Works with ManyToMany relationships
- Works with reverse ForeignKey
- No cartesian product issues
- Can prefetch multiple relationships efficiently
- Supports custom querysets (filtering)

**Cons:**
- Multiple database queries (but still efficient)
- More memory usage (caches in Python)
- Cannot filter on prefetched fields in database
- Slightly slower than select_related for simple cases

**Use When:**
- ManyToManyField relationships (tags, categories)
- Reverse ForeignKey relationships (comments, ratings)
- GenericRelation
- Need to prefetch filtered related objects
- Multiple related objects per parent

**Performance:**
```python
# Without prefetch_related: 101 queries
articles = Article.objects.all()  # 1 query
for article in articles:  # 100 articles
    print(article.tags.all())  # 100 queries
# Total: 101 queries, ~500ms

# With prefetch_related: 2 queries
articles = Article.objects.prefetch_related('tags').all()
for article in articles:
    print(article.tags.all())  # No queries
# Total: 2 queries, ~10ms
```

#### Decision Matrix: select_related vs prefetch_related

| Criteria | select_related | prefetch_related |
|----------|----------------|------------------|
| **Relationship Type** | ForeignKey, OneToOne | ManyToMany, Reverse FK |
| **Number of Queries** | 1 (JOIN) | 2+ (separate) |
| **Memory Usage** | Low | Medium |
| **Speed** | Fastest | Fast |
| **Filtering** | Can filter in DB | Filter before prefetch |
| **Complexity** | Simple | Medium |
| **Cartesian Product** | Possible | No |

#### When to Use Each

**Use select_related() when:**
```
✓ ForeignKey relationships (article.author)
✓ OneToOneField relationships (user.profile)
✓ Forward relationships only
✓ Need to filter on related fields
✓ Want single query
✓ Small number of JOINs (< 3)
```

**Use prefetch_related() when:**
```
✓ ManyToManyField (article.tags)
✓ Reverse ForeignKey (article.comments)
✓ Multiple related objects
✓ Need filtered related objects
✓ Avoiding cartesian products
✓ GenericRelation
```

#### Combining Both (Best Practice)

```python
class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Article.objects.select_related(
            'author',           # ForeignKey - use select_related
            'category'          # ForeignKey - use select_related
        ).prefetch_related(
            'tags',             # ManyToMany - use prefetch_related
            'comments',         # Reverse FK - use prefetch_related
            'comments__author'  # Nested: prefetch comments, select author
        )

# Result: 3 queries instead of 100+
# Query 1: Articles with author and category (JOIN)
# Query 2: All tags for these articles
# Query 3: All comments with authors for these articles
```

#### Advanced: Prefetch with Custom Querysets

```python
from django.db.models import Prefetch

class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Only prefetch approved comments, ordered by date
        approved_comments = Comment.objects.filter(
            approved=True
        ).select_related('author').order_by('-created_at')
        
        return Article.objects.select_related(
            'author'
        ).prefetch_related(
            Prefetch('comments', queryset=approved_comments),
            'tags'
        )
```

**Why This is Better:**
- Filters comments in database (not Python)
- Orders comments efficiently
- Combines select_related within prefetch
- Total queries: 3 (articles+author, comments+author, tags)

#### Common Mistakes to Avoid

**Mistake 1: Using select_related on ManyToMany**
```python
# WRONG - select_related doesn't work with ManyToMany
Article.objects.select_related('tags')  # Ignored!

# CORRECT
Article.objects.prefetch_related('tags')
```

**Mistake 2: Accessing Related Objects Without Optimization**
```python
# BAD - N+1 queries
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # Query per article
    print(article.tags.count())  # Query per article

# GOOD - 3 queries total
articles = Article.objects.select_related('author').prefetch_related('tags')
for article in articles:
    print(article.author.name)  # No query
    print(article.tags.count())  # No query
```

**Mistake 3: Over-optimizing**
```python
# BAD - Fetching data you don't use
Article.objects.select_related(
    'author', 'category', 'publisher', 'editor'
).prefetch_related(
    'tags', 'comments', 'ratings', 'bookmarks'
)  # If you only need author and tags, this is wasteful

# GOOD - Only fetch what you need
Article.objects.select_related('author').prefetch_related('tags')
```

### Combining select_related() and prefetch_related()

```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        return Article.objects.select_related(
            'author',           # ForeignKey
            'category'          # ForeignKey
        ).prefetch_related(
            'tags',             # ManyToMany
            'comments',         # Reverse ForeignKey
            'comments__author'  # Nested relationship
        )
```

### Prefetch with Custom Querysets

```python
from django.db.models import Prefetch

class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Only prefetch approved comments
        approved_comments = Comment.objects.filter(approved=True)
        
        return Article.objects.prefetch_related(
            Prefetch('comments', queryset=approved_comments)
        ).select_related('author')
```

## Serializer Optimization

### Avoid Heavy Operations in SerializerMethodField

**Bad Practice:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    comment_count = serializers.SerializerMethodField()
    
    def get_comment_count(self, obj):
        # Database query for EACH article
        return obj.comments.count()  # N queries!
```

**Good Practice:**
```python
# In ViewSet
class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Annotate in queryset - single query
        return Article.objects.annotate(
            comment_count=Count('comments')
        )

# In Serializer
class ArticleSerializer(serializers.ModelSerializer):
    comment_count = serializers.IntegerField(read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'comment_count']
```

### Use only() and defer()

Fetch only the fields you need.

```python
# only() - Fetch only specified fields
class ArticleListView(generics.ListAPIView):
    def get_queryset(self):
        # Only fetch id, title, created_at
        return Article.objects.only('id', 'title', 'created_at')

# defer() - Fetch all fields except specified
class ArticleListView(generics.ListAPIView):
    def get_queryset(self):
        # Fetch all fields except content (large text field)
        return Article.objects.defer('content')
```

**Use Case:**
- List views with minimal data
- Avoiding large text/blob fields
- Reducing database load

### Different Serializers for Different Actions

```python
class ArticleListSerializer(serializers.ModelSerializer):
    """Minimal serializer for list view"""
    class Meta:
        model = Article
        fields = ['id', 'title', 'author', 'created_at']

class ArticleDetailSerializer(serializers.ModelSerializer):
    """Full serializer for detail view"""
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'author', 'comments', 'created_at', 'updated_at']

class ArticleViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.action == 'list':
            return ArticleListSerializer
        return ArticleDetailSerializer
    
    def get_queryset(self):
        if self.action == 'list':
            return Article.objects.select_related('author').only(
                'id', 'title', 'author__username', 'created_at'
            )
        return Article.objects.select_related('author').prefetch_related('comments')
```

## Pagination

### Why Pagination is Critical
Without pagination, large datasets can:
- Cause memory issues
- Slow response times
- Enable DoS attacks
- Overwhelm clients

### Configure Default Pagination

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20
}
```

### Custom Pagination Classes

#### Page Number Pagination
```python
from rest_framework.pagination import PageNumberPagination

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    pagination_class = StandardResultsSetPagination
```

**Usage:**
```
GET /articles/?page=2
GET /articles/?page=2&page_size=50
```

#### Limit/Offset Pagination
```python
from rest_framework.pagination import LimitOffsetPagination

class CustomLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    max_limit = 100

class ArticleViewSet(viewsets.ModelViewSet):
    pagination_class = CustomLimitOffsetPagination
```

**Usage:**
```
GET /articles/?limit=20&offset=40
```

#### Cursor Pagination (Best for Large Datasets)
```python
from rest_framework.pagination import CursorPagination

class ArticleCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must have ordering

class ArticleViewSet(viewsets.ModelViewSet):
    pagination_class = ArticleCursorPagination
```

**Benefits:**
- Consistent pagination even with new data
- Better performance for large datasets
- Prevents duplicate/missing items

## Caching Strategies: Choosing the Right Approach

### Comparing Caching Approaches

#### Approach 1: View-Level Caching (Simplest)

**Best For:** Static or semi-static content, public endpoints

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page

class ArticleViewSet(viewsets.ModelViewSet):
    @method_decorator(cache_page(60 * 15))  # Cache for 15 minutes
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
    
    @method_decorator(cache_page(60 * 60))  # Cache for 1 hour
    def retrieve(self, request, *args, **kwargs):
        return super().retrieve(request, *args, **kwargs)
```

**Pros:**
- Extremely simple (one decorator)
- Caches entire HTTP response
- Includes headers
- Works with Django's cache framework
- Minimal code changes

**Cons:**
- All-or-nothing caching
- Same cache for all users (unless varied)
- Hard to invalidate selectively
- Can't cache parts of response
- Query parameters affect cache key

**Use When:**
- Public, read-only endpoints
- Content changes infrequently
- Same response for all users
- Simple caching needs
- Quick wins needed

**Performance Impact:**
```python
# Without cache: 50ms per request
# With cache: 2ms per request (25x faster)
```

#### Approach 2: Low-Level Caching (More Control)

**Best For:** Selective caching, complex invalidation logic

```python
from django.core.cache import cache

class ArticleViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        # Build cache key based on filters
        cache_key = f'article_list_{request.query_params.get("category", "all")}'
        cached_data = cache.get(cache_key)
        
        if cached_data is not None:
            return Response(cached_data)
        
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        
        cache.set(cache_key, serializer.data, 60 * 15)
        return Response(serializer.data)
```

**Pros:**
- Fine-grained control
- Cache specific data (not full response)
- Custom cache keys
- Selective invalidation
- Can cache querysets, serialized data, or computed values

**Cons:**
- More code to write
- Must handle cache keys manually
- Must implement invalidation
- Easy to make mistakes
- More maintenance

**Use When:**
- Need custom cache keys
- Selective invalidation required
- Caching computed values
- Different cache times per data type
- Complex caching logic

**Performance Impact:**
```python
# Without cache: 100ms (complex query)
# With cache: 1ms (from Redis)
```

#### Approach 3: Query-Level Caching (Database)

**Best For:** Expensive database queries, aggregations

```python
from django.core.cache import cache
from django.db.models import Count

class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        cache_key = 'article_stats'
        stats = cache.get(cache_key)
        
        if stats is None:
            stats = Article.objects.aggregate(
                total=Count('id'),
                published=Count('id', filter=Q(published=True))
            )
            cache.set(cache_key, stats, 60 * 5)  # 5 minutes
        
        return Article.objects.all()
```

**Pros:**
- Caches expensive queries
- Reduces database load
- Can cache aggregations
- Transparent to views

**Cons:**
- Only caches query results
- Doesn't cache serialization
- Still processes data
- Complex invalidation

**Use When:**
- Expensive aggregations
- Complex queries
- Database is bottleneck
- Multiple views use same query

#### Approach 4: Redis with django-redis (Production)

**Best For:** High-traffic production systems

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'PARSER_CLASS': 'redis.connection.HiredisParser',
            'CONNECTION_POOL_CLASS_KWARGS': {
                'max_connections': 50,
            },
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 300,
    }
}

# Usage (same as low-level caching)
from django_redis import get_redis_connection

redis_conn = get_redis_connection("default")
redis_conn.setex('key', 300, 'value')
```

**Pros:**
- Fast (in-memory)
- Distributed (shared across servers)
- Persistent (survives restarts)
- Advanced features (pub/sub, atomic operations)
- Production-ready

**Cons:**
- External dependency (Redis)
- Infrastructure complexity
- Memory management needed
- Cost (if hosted)

**Use When:**
- Production environments
- Multi-server deployment
- High traffic (1000+ req/s)
- Need distributed cache
- Already using Redis

#### Approach 5: CDN/HTTP Caching (Edge Caching)

**Best For:** Static content, global distribution

```python
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        response = super().list(request, *args, **kwargs)
        
        # Set cache headers for CDN
        response['Cache-Control'] = 'public, max-age=3600'
        response['Vary'] = 'Accept-Language'
        
        return response
```

**Pros:**
- Caches at edge (closest to users)
- Reduces server load completely
- Global distribution
- Fastest possible response
- Scales infinitely

**Cons:**
- Only works for GET requests
- Public data only
- Invalidation is hard
- Requires CDN setup
- Cost

**Use When:**
- Public APIs
- Global user base
- Static content
- High traffic
- Budget allows

### Decision Matrix for Caching

| Criteria | View-Level | Low-Level | Query-Level | Redis | CDN |
|----------|-----------|-----------|-------------|-------|-----|
| **Complexity** | ⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Control** | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Performance** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalability** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Setup Time** | 5 min | 30 min | 20 min | 2 hours | 4 hours |
| **Cost** | Free | Free | Free | $ | $$$ |

### When to Use Each Caching Strategy

**Use View-Level Caching when:**
```
✓ Quick wins needed
✓ Public endpoints
✓ Simple requirements
✓ Same response for all users
✓ Content changes infrequently
```

**Use Low-Level Caching when:**
```
✓ Need fine-grained control
✓ Custom cache keys required
✓ Selective invalidation needed
✓ Caching computed values
✓ Complex caching logic
```

**Use Query-Level Caching when:**
```
✓ Expensive database queries
✓ Complex aggregations
✓ Database is bottleneck
✓ Same query used multiple times
```

**Use Redis when:**
```
✓ Production environment
✓ Multi-server deployment
✓ High traffic
✓ Need distributed cache
✓ Advanced features needed
```

**Use CDN when:**
```
✓ Global user base
✓ Public content
✓ Very high traffic
✓ Static or semi-static data
✓ Budget allows
```

### Cache Invalidation Strategies

#### Strategy 1: Time-Based (TTL)
```python
# Simple: Cache expires after time
cache.set('key', value, 60 * 15)  # 15 minutes
```

**Pros:** Simple, automatic
**Cons:** Stale data until expiration
**Use When:** Data changes predictably

#### Strategy 2: Event-Based (Signals)
```python
from django.db.models.signals import post_save, post_delete

@receiver([post_save, post_delete], sender=Article)
def invalidate_cache(sender, instance, **kwargs):
    cache.delete('article_list')
    cache.delete(f'article_{instance.id}')
```

**Pros:** Immediate invalidation
**Cons:** More complex, can miss cases
**Use When:** Data changes unpredictably

#### Strategy 3: Cache Versioning
```python
CACHE_VERSION = 1

cache_key = f'article_list_v{CACHE_VERSION}'
cache.set(cache_key, data, timeout=None)  # Never expires

# To invalidate: increment CACHE_VERSION
```

**Pros:** No explicit invalidation needed
**Cons:** Old cache remains in memory
**Use When:** Deployment-based invalidation

#### Strategy 4: Cache Tags (Advanced)
```python
# Tag-based invalidation
cache.set('article_1', data, tags=['articles', 'user_123'])
cache.delete_many(tags=['articles'])  # Invalidate all articles
```

**Pros:** Flexible, group invalidation
**Cons:** Requires special cache backend
**Use When:** Complex invalidation patterns

### Layered Caching Strategy (Best Practice)

```python
# Layer 1: CDN (edge caching) - 1 hour
# Layer 2: Redis (application caching) - 15 minutes  
# Layer 3: Database query optimization

class ArticleViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        # Layer 2: Redis cache
        cache_key = 'article_list'
        cached_data = cache.get(cache_key)
        
        if cached_data:
            response = Response(cached_data)
        else:
            # Layer 3: Optimized query
            queryset = Article.objects.select_related('author').prefetch_related('tags')
            serializer = self.get_serializer(queryset, many=True)
            cache.set(cache_key, serializer.data, 60 * 15)
            response = Response(serializer.data)
        
        # Layer 1: CDN headers
        response['Cache-Control'] = 'public, max-age=3600'
        return response
```

**Why Layered Caching:**
- CDN serves most requests (fastest)
- Redis serves cache misses (fast)
- Database only hit on cache miss (rare)
- Different TTLs per layer

## Database Indexing

### Add Indexes to Frequently Queried Fields

```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True, db_index=True)
    author = models.ForeignKey(User, on_delete=models.CASCADE, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    published = models.BooleanField(default=False, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['author', 'published']),
            models.Index(fields=['-created_at']),
            models.Index(fields=['published', '-created_at']),
        ]
```

**When to Add Indexes:**
- Fields used in `filter()`
- Fields used in `order_by()`
- Foreign keys (automatic in Django)
- Fields used in `WHERE` clauses

**When NOT to Add Indexes:**
- Small tables
- Fields that are rarely queried
- Fields with low cardinality (few unique values)
- Write-heavy tables (indexes slow down writes)

## Aggregation and Annotation

### Use Database Aggregations

```python
from django.db.models import Count, Avg, Sum, Max, Min

class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Article.objects.annotate(
            comment_count=Count('comments'),
            average_rating=Avg('ratings__score'),
            total_views=Sum('views'),
            latest_comment=Max('comments__created_at')
        )

class ArticleSerializer(serializers.ModelSerializer):
    comment_count = serializers.IntegerField(read_only=True)
    average_rating = serializers.FloatField(read_only=True)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'comment_count', 'average_rating']
```

### Conditional Aggregation

```python
from django.db.models import Q, Count, Case, When, IntegerField

class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Article.objects.annotate(
            approved_comment_count=Count(
                'comments',
                filter=Q(comments__approved=True)
            ),
            pending_comment_count=Count(
                'comments',
                filter=Q(comments__approved=False)
            )
        )
```

## Bulk Operations

### Bulk Create

```python
# Bad - Multiple database hits
for item in data:
    Article.objects.create(**item)

# Good - Single database hit
articles = [Article(**item) for item in data]
Article.objects.bulk_create(articles, batch_size=100)
```

### Bulk Update

```python
from django.db.models import F

# Bad - Multiple queries
for article in articles:
    article.views += 1
    article.save()

# Good - Single query
Article.objects.filter(id__in=article_ids).update(views=F('views') + 1)

# Or with bulk_update
for article in articles:
    article.views += 1
Article.objects.bulk_update(articles, ['views'], batch_size=100)
```

## Asynchronous Tasks

### Use Celery for Heavy Operations

```python
from celery import shared_task

@shared_task
def generate_report(article_id):
    """Heavy operation - run asynchronously"""
    article = Article.objects.get(id=article_id)
    # Generate report...
    return report_data

class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def generate_report(self, request, pk=None):
        article = self.get_object()
        
        # Queue task asynchronously
        task = generate_report.delay(article.id)
        
        return Response({
            'status': 'Report generation started',
            'task_id': task.id
        }, status=status.HTTP_202_ACCEPTED)
```

### Background Processing

```python
@shared_task
def send_notification_emails(article_id):
    """Send emails in background"""
    article = Article.objects.get(id=article_id)
    subscribers = article.author.followers.all()
    
    for subscriber in subscribers:
        send_email(subscriber.email, article)

class ArticleViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        article = serializer.save()
        # Don't wait for emails to be sent
        send_notification_emails.delay(article.id)
```

## Response Optimization

### Compress Responses

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.gzip.GZipMiddleware',  # Add this
    'django.middleware.common.CommonMiddleware',
    # ... other middleware
]
```

### Optimize JSON Rendering

```python
from rest_framework.renderers import JSONRenderer
import orjson

class OrjsonRenderer(JSONRenderer):
    """Faster JSON rendering with orjson"""
    def render(self, data, accepted_media_type=None, renderer_context=None):
        return orjson.dumps(data)

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'myapp.renderers.OrjsonRenderer',
    ],
}
```

## Connection Pooling

### Database Connection Pooling

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # Connection pooling
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}
```

## Monitoring and Profiling

### Django Debug Toolbar

```python
# settings.py (development only)
if DEBUG:
    INSTALLED_APPS += ['debug_toolbar']
    MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']
    INTERNAL_IPS = ['127.0.0.1']
```

### Query Counting Middleware

```python
class QueryCountDebugMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        from django.db import connection
        from django.db import reset_queries
        
        reset_queries()
        
        response = self.get_response(request)
        
        if settings.DEBUG:
            print(f"Total queries: {len(connection.queries)}")
            for query in connection.queries:
                print(f"{query['time']}: {query['sql']}")
        
        return response
```

### Performance Logging

```python
import time
import logging

logger = logging.getLogger(__name__)

class PerformanceLoggingMixin:
    def dispatch(self, request, *args, **kwargs):
        start_time = time.time()
        response = super().dispatch(request, *args, **kwargs)
        duration = time.time() - start_time
        
        logger.info(
            f"{request.method} {request.path} - {response.status_code} - {duration:.2f}s"
        )
        
        return response

class ArticleViewSet(PerformanceLoggingMixin, viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

## Performance Testing

### Load Testing with Locust

```python
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        # Login
        response = self.client.post("/api/auth/login/", {
            "username": "testuser",
            "password": "testpass"
        })
        self.token = response.json()['token']
    
    @task(3)
    def list_articles(self):
        self.client.get(
            "/api/articles/",
            headers={"Authorization": f"Token {self.token}"}
        )
    
    @task(1)
    def create_article(self):
        self.client.post(
            "/api/articles/",
            json={"title": "Test Article", "content": "Test content"},
            headers={"Authorization": f"Token {self.token}"}
        )
```

## Summary

Key performance optimization practices:

1. **Query Optimization:** Use `select_related()` and `prefetch_related()` to avoid N+1 queries
2. **Serializer Efficiency:** Avoid heavy operations in `SerializerMethodField`, use annotations
3. **Pagination:** Always paginate list endpoints, use cursor pagination for large datasets
4. **Caching:** Implement view-level and low-level caching with proper invalidation
5. **Indexing:** Add database indexes on frequently queried fields
6. **Aggregation:** Use database aggregations instead of Python loops
7. **Bulk Operations:** Use `bulk_create()` and `bulk_update()` for multiple records
8. **Async Tasks:** Move heavy operations to background tasks with Celery
9. **Response Compression:** Enable GZip compression
10. **Monitoring:** Use profiling tools and logging to identify bottlenecks

Following these practices ensures your API performs well under load and scales efficiently.
