# Views and ViewSets Best Practices

## Overview
Views and ViewSets in Django REST Framework handle HTTP requests and provide actions for interacting with your models. This document covers best practices for implementing efficient and maintainable views.

> **🏗️ Architecture Best Practice:** Keep views thin by delegating business logic to service/selector layers. See `01-API-Design-and-Architecture.md` for the complete Service/Selector pattern implementation.

## ViewSets vs Views: Choosing the Right Approach

### Detailed Comparison

#### Approach 1: ViewSets (Recommended for Standard CRUD)

**ModelViewSet - Full CRUD:**
```python
from rest_framework import viewsets

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated]
```

**Pros:**
- Minimal code (5 lines for full CRUD)
- Automatic URL routing
- Consistent patterns
- Built-in actions
- Easy to maintain

**Cons:**
- Less flexibility for custom logic
- May provide more than needed
- Harder to customize individual actions
- Learning curve for beginners

**Use When:**
- Building standard CRUD APIs
- Need all or most CRUD operations
- Want consistent patterns
- Rapid development is priority
- Team follows DRF conventions

#### Approach 2: Generic Views (More Control)

```python
from rest_framework import generics

class ArticleListCreate(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

class ArticleRetrieveUpdateDestroy(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

**Pros:**
- More explicit control
- Easier to customize per endpoint
- Clear what each view does
- Good for learning DRF

**Cons:**
- More code than ViewSets
- Manual URL configuration
- Potential for inconsistency
- More files to manage

**Use When:**
- Need different logic per endpoint
- Want explicit control
- Building non-standard APIs
- Learning DRF fundamentals
- Different permissions per action

#### Approach 3: APIView (Maximum Flexibility)

```python
from rest_framework.views import APIView
from rest_framework.response import Response

class ArticleList(APIView):
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)
```

**Pros:**
- Complete control over logic
- Easy to understand
- No "magic" behavior
- Custom response formats
- Complex business logic

**Cons:**
- Most verbose (30+ lines vs 5)
- Must implement everything
- Easy to introduce inconsistencies
- More testing needed
- Repetitive code

**Use When:**
- Complex custom logic required
- Non-standard response formats
- Learning HTTP/REST fundamentals
- Existing codebase uses APIView
- Need complete control

#### Approach 4: Function-Based Views (Simplest)

```python
from rest_framework.decorators import api_view

@api_view(['GET', 'POST'])
def article_list(request):
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)
```

**Pros:**
- Simplest to understand
- No classes needed
- Good for simple endpoints
- Easy to test
- Quick prototyping

**Cons:**
- Doesn't scale well
- Hard to reuse logic
- No inheritance benefits
- Becomes messy with complexity
- Limited DRF features

**Use When:**
- Simple, one-off endpoints
- Prototyping
- Webhooks or callbacks
- Learning basics
- Very simple APIs

### Decision Matrix

| Criteria | ViewSets | Generic Views | APIView | Function-Based |
|----------|----------|---------------|---------|----------------|
| **Code Volume** | Minimal | Medium | High | Medium |
| **Flexibility** | Medium | High | Highest | Medium |
| **Learning Curve** | Medium | Low | Low | Lowest |
| **Best For** | CRUD APIs | Mixed operations | Custom logic | Simple endpoints |
| **URL Routing** | Automatic | Manual | Manual | Manual |
| **Reusability** | High | Medium | Low | Lowest |
| **Maintenance** | Easy | Medium | Hard | Medium |
| **Team Size** | Large teams | Medium teams | Any | Small teams |

### When to Use Each Approach

**Use ViewSets when:**
```
✓ Building standard REST APIs
✓ Need CRUD operations
✓ Want automatic URL routing
✓ Team follows DRF patterns
✓ Rapid development needed
✓ Consistency is important
```

**Use Generic Views when:**
```
✓ Need mix of operations
✓ Different logic per endpoint
✓ Want explicit control
✓ Learning DRF
✓ Customizing specific actions
```

**Use APIView when:**
```
✓ Complex custom logic
✓ Non-standard responses
✓ Full control required
✓ Unique business rules
✓ Integration with legacy code
```

**Use Function-Based Views when:**
```
✓ Very simple endpoints
✓ Prototyping quickly
✓ Webhooks/callbacks
✓ One-off utilities
✓ Learning basics
```

### Hybrid Approach (Best Practice)

Combine approaches based on needs:

```python
# Standard CRUD - Use ViewSet
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

# Custom endpoint - Use APIView
class ArticleStatsView(APIView):
    def get(self, request):
        stats = calculate_complex_stats()
        return Response(stats)

# Simple utility - Use function
@api_view(['POST'])
def article_webhook(request):
    process_webhook(request.data)
    return Response({'status': 'received'})
```

### Basic ViewSet Example

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Tag
from .serializers import TagSerializer

class TagViewSet(viewsets.ModelViewSet):
    """
    ViewSet for viewing and editing Tag instances.
    Provides: list, create, retrieve, update, partial_update, destroy
    """
    queryset = Tag.objects.all()
    serializer_class = TagSerializer
    permission_classes = [IsAuthenticated]
```

### ViewSet Mixins
Combine mixins to create custom ViewSets with only the actions you need.

```python
from rest_framework import mixins, viewsets

class TagViewSet(mixins.ListModelMixin,
                 mixins.CreateModelMixin,
                 mixins.RetrieveModelMixin,
                 viewsets.GenericViewSet):
    """
    ViewSet that provides list, create, and retrieve actions only.
    No update or delete functionality.
    """
    queryset = Tag.objects.all()
    serializer_class = TagSerializer
```

**Available Mixins:**
- `ListModelMixin` - List a queryset
- `CreateModelMixin` - Create a model instance
- `RetrieveModelMixin` - Retrieve a model instance
- `UpdateModelMixin` - Update a model instance
- `DestroyModelMixin` - Delete a model instance

### ModelViewSet
Use `ModelViewSet` when you need all CRUD operations.

```python
from rest_framework import viewsets

class ArticleViewSet(viewsets.ModelViewSet):
    """
    Provides all CRUD operations:
    - list() - GET /articles/
    - create() - POST /articles/
    - retrieve() - GET /articles/{id}/
    - update() - PUT /articles/{id}/
    - partial_update() - PATCH /articles/{id}/
    - destroy() - DELETE /articles/{id}/
    """
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

## URL Configuration with Routers

### Using Routers
Routers automatically generate URL patterns for ViewSets, enforcing best practices in URL naming.

```python
# urls.py
from rest_framework.routers import DefaultRouter
from .views import TagViewSet, ArticleViewSet

router = DefaultRouter()
router.register(r'tags', TagViewSet, basename='tag')
router.register(r'articles', ArticleViewSet, basename='article')

urlpatterns = [
    path('api/v1/', include(router.urls)),
]
```

**Generated URLs:**
```
GET    /api/v1/tags/              -> list all tags
POST   /api/v1/tags/              -> create new tag
GET    /api/v1/tags/<id>/         -> retrieve specific tag
PUT    /api/v1/tags/<id>/         -> update tag
PATCH  /api/v1/tags/<id>/         -> partial update tag
DELETE /api/v1/tags/<id>/         -> delete tag
```

**Benefits:**
- Predictable URL structure
- Automatic URL naming
- Less boilerplate code
- RESTful conventions enforced

## Custom Actions

### Adding Custom Actions with @action Decorator
Extend ViewSets with custom endpoints using the `@action` decorator.

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import status

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Custom action to publish an article.
        URL: POST /articles/{id}/publish/
        
        Best Practice: Delegate business logic to service layer.
        """
        article = self.get_object()
        
        # ✅ RECOMMENDED: Use service layer for business logic
        # from myapp.services import article_publish
        # article = article_publish(article=article, user=request.user)
        
        # ⚠️ ACCEPTABLE: Simple logic can stay in view
        article.published = True
        article.published_date = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def recent(self, request):
        """
        Custom action to get recent articles.
        URL: GET /articles/recent/
        """
        recent_articles = self.queryset.order_by('-created_at')[:10]
        serializer = self.get_serializer(recent_articles, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'], url_path='comments')
    def get_comments(self, request, pk=None):
        """
        Custom action with custom URL path.
        URL: GET /articles/{id}/comments/
        """
        article = self.get_object()
        comments = article.comments.all()
        serializer = CommentSerializer(comments, many=True)
        return Response(serializer.data)
```

**@action Parameters:**
- `detail=True` - Action on single object (requires pk)
- `detail=False` - Action on collection
- `methods` - HTTP methods allowed (default: ['get'])
- `url_path` - Custom URL path (default: function name)
- `url_name` - Custom URL name for reverse lookups

## Overriding ViewSet Methods

### perform_create()
Override to customize object creation.

```python
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def perform_create(self, serializer):
        # Set author to current user
        serializer.save(author=self.request.user)
```

### perform_update()
Override to customize object updates.

```python
def perform_update(self, serializer):
    # Track who last modified the object
    serializer.save(
        modified_by=self.request.user,
        modified_at=timezone.now()
    )
```

### perform_destroy()
Override to customize deletion behavior.

```python
def perform_destroy(self, instance):
    # Soft delete instead of hard delete
    instance.is_deleted = True
    instance.deleted_at = timezone.now()
    instance.save()
```

### get_queryset()
Override to customize the queryset dynamically.

```python
def get_queryset(self):
    """
    Filter queryset based on user permissions.
    """
    user = self.request.user
    
    if user.is_staff:
        # Staff can see all articles
        return Article.objects.all()
    else:
        # Regular users see only published articles or their own
        return Article.objects.filter(
            Q(published=True) | Q(author=user)
        )
```

### get_serializer_class()
Override to use different serializers for different actions.

```python
def get_serializer_class(self):
    """
    Use different serializers for different actions.
    """
    if self.action == 'list':
        return ArticleListSerializer  # Minimal fields
    elif self.action == 'retrieve':
        return ArticleDetailSerializer  # All fields
    elif self.action in ['create', 'update']:
        return ArticleWriteSerializer  # Write-specific fields
    return ArticleSerializer  # Default
```

### get_permissions()
Override to set different permissions for different actions.

```python
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly, IsAdminUser

def get_permissions(self):
    """
    Set different permissions for different actions.
    """
    if self.action in ['list', 'retrieve']:
        permission_classes = [IsAuthenticatedOrReadOnly]
    elif self.action in ['create']:
        permission_classes = [IsAuthenticated]
    elif self.action in ['update', 'partial_update', 'destroy']:
        permission_classes = [IsAuthenticated, IsOwnerOrAdmin]
    else:
        permission_classes = [IsAdminUser]
    
    return [permission() for permission in permission_classes]
```

## Function-Based Views vs Class-Based Views

### When to Use Function-Based Views
Use function-based views for simple, one-off endpoints.

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def article_list(request):
    """
    List all articles or create a new article.
    """
    if request.method == 'GET':
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### When to Use Class-Based Views
Use class-based views for more complex logic and reusability.

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ArticleList(APIView):
    """
    List all articles or create a new article.
    """
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

## Generic Views

### Using Generic Views
DRF provides generic views for common patterns.

```python
from rest_framework import generics

class ArticleListCreate(generics.ListCreateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

class ArticleRetrieveUpdateDestroy(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrAdmin]
```

**Available Generic Views:**
- `CreateAPIView` - Create only
- `ListAPIView` - List only
- `RetrieveAPIView` - Retrieve only
- `DestroyAPIView` - Delete only
- `UpdateAPIView` - Update only
- `ListCreateAPIView` - List and create
- `RetrieveUpdateAPIView` - Retrieve and update
- `RetrieveDestroyAPIView` - Retrieve and delete
- `RetrieveUpdateDestroyAPIView` - Retrieve, update, and delete

## Filtering, Searching, and Ordering

### Using django-filter
```python
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import filters

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    
    # Filtering
    filterset_fields = ['author', 'category', 'published']
    
    # Searching
    search_fields = ['title', 'content', 'author__username']
    
    # Ordering
    ordering_fields = ['created_at', 'title', 'views']
    ordering = ['-created_at']  # Default ordering
```

**Usage:**
```
GET /articles/?author=1&published=true
GET /articles/?search=django
GET /articles/?ordering=-created_at
GET /articles/?author=1&search=rest&ordering=title
```

### Custom Filtering
```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        queryset = Article.objects.all()
        
        # Custom filtering logic
        published = self.request.query_params.get('published')
        if published is not None:
            queryset = queryset.filter(published=published.lower() == 'true')
        
        author_id = self.request.query_params.get('author_id')
        if author_id is not None:
            queryset = queryset.filter(author_id=author_id)
        
        # Date range filtering
        start_date = self.request.query_params.get('start_date')
        end_date = self.request.query_params.get('end_date')
        if start_date and end_date:
            queryset = queryset.filter(
                created_at__range=[start_date, end_date]
            )
        
        return queryset
```

## Pagination

> **📖 Reference:** See `01-API-Design-and-Architecture.md` for comprehensive pagination strategy comparison (Page Number, Limit/Offset, Cursor) and `06-Performance-and-Optimization.md` for performance considerations.

### Configure Pagination in ViewSet
```python
from rest_framework.pagination import PageNumberPagination

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    pagination_class = StandardResultsSetPagination  # Always paginate list endpoints
```

**Usage:**
```
GET /articles/?page=2
GET /articles/?page=2&page_size=50
```

**Important:** Always implement pagination on list endpoints to prevent performance issues with large datasets.

## Performance Optimization

> **📖 Reference:** See `06-Performance-and-Optimization.md` for comprehensive performance strategies including query optimization, caching, indexing, and bulk operations.

### Select Related and Prefetch Related
```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        """
        Optimize queries with select_related and prefetch_related.
        See 06-Performance-and-Optimization.md for detailed N+1 query solutions.
        """
        return Article.objects.select_related(
            'author',  # ForeignKey/OneToOne
            'category'
        ).prefetch_related(
            'tags',  # ManyToMany
            'comments'  # Reverse ForeignKey
        )
```

### Query Optimization
```python
from django.db.models import Count, Avg, Q

class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        """
        Annotate queryset with aggregated data.
        """
        return Article.objects.annotate(
            comment_count=Count('comments'),
            average_rating=Avg('ratings__score')
        ).select_related('author')
```

## Error Handling in Views

> **📖 Reference:** See `09-Error-Handling-and-Status-Codes.md` for the complete error handling strategy and standard error format.

### Custom Exception Handling

All error responses should follow the standard format defined in `09-Error-Handling-and-Status-Codes.md`:

```python
from rest_framework.views import exception_handler
from rest_framework.response import Response

def custom_exception_handler(exc, context):
    """
    Custom exception handler following the standard error format.
    
    Standard error format:
    {
        "error": {
            "code": "ERROR_CODE",
            "message": "Human-readable message",
            "fields": {...} or null
        }
    }
    """
    # Call DRF's default exception handler first
    response = exception_handler(exc, context)
    
    if response is not None:
        # Use standard error format from 09-Error-Handling-and-Status-Codes.md
        error_code = getattr(exc, 'default_code', 'ERROR').upper()
        
        custom_response = {
            'error': {
                'code': error_code,
                'message': str(exc),
                'fields': response.data if isinstance(response.data, dict) else None
            }
        }
        response.data = custom_response
    
    return response

# In settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'myapp.utils.custom_exception_handler'
}
```

**Note:** Always use **400** for validation errors, never 422.

### Handling Specific Exceptions in Views
```python
from rest_framework.exceptions import ValidationError, NotFound

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def retrieve(self, request, pk=None):
        try:
            article = self.get_object()
        except Article.DoesNotExist:
            raise NotFound(detail="Article not found")
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        article = self.get_object()
        
        if article.published:
            raise ValidationError("Article is already published")
        
        article.published = True
        article.save()
        
        return Response({'status': 'article published'})
```

## Summary

Key ViewSet and View best practices:

1. **Use ViewSets:** For standard CRUD operations to reduce boilerplate
2. **Routers:** Use routers for automatic URL generation
3. **Custom Actions:** Extend functionality with `@action` decorator
4. **Override Methods:** Customize behavior by overriding `perform_*` and `get_*` methods
5. **Filtering:** Implement filtering, searching, and ordering for better UX
6. **Pagination:** Always paginate list endpoints to prevent performance issues
7. **Optimization:** Use `select_related` and `prefetch_related` to avoid N+1 queries
8. **Permissions:** Set appropriate permissions per action
9. **Serializers:** Use different serializers for different actions when needed
10. **Error Handling:** Implement proper error handling and return meaningful messages

Following these practices ensures your views are efficient, maintainable, and provide a great developer experience.
