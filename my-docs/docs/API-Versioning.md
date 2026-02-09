# API Versioning & Architecture Best Practices

## Overview
API versioning is not just a technical implementation detail but a strategic architectural decision that impacts your entire project structure, team organization, and long-term maintainability. This document covers architectural considerations and versioning strategies for evolving APIs in Django REST Framework.

## Project Architecture for Versioned APIs

### 1. Architectural Impact of Versioning

Before choosing a versioning strategy, consider how it affects your project architecture:

#### Code Organization
```
myproject/
├── apps/
│   ├── articles/
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── views/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── article_views.py
│   │   │   │   │   └── comment_views.py
│   │   │   │   ├── serializers/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── article_serializers.py
│   │   │   │   │   └── comment_serializers.py
│   │   │   │   └── urls.py
│   │   │   ├── v2/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── views/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── article_views.py
│   │   │   │   │   └── comment_views.py
│   │   │   │   ├── serializers/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── article_serializers.py
│   │   │   │   │   └── comment_serializers.py
│   │   │   │   └── urls.py
│   │   │   └── v3/
│   │   │       ├── __init__.py
│   │   │       ├── views/
│   │   │       │   ├── __init__.py
│   │   │       │   ├── article_views.py
│   │   │       │   └── comment_views.py
│   │   │       ├── serializers/
│   │   │       │   ├── __init__.py
│   │   │       │   ├── article_serializers.py
│   │   │       │   └── comment_serializers.py
│   │   │       └── urls.py
│   │   ├── models.py
│   │   └── services.py
│   └── users/
│       └── api/
│           └── v1/
│               ├── __init__.py
│               ├── views/
│               │   ├── __init__.py
│               │   └── user_views.py
│               ├── serializers/
│               │   ├── __init__.py
│               │   └── user_serializers.py
│               └── urls.py
├── config/
│   ├── urls.py
│   ├── settings/
│   │   ├── base.py
│   │   ├── v1.py
│   │   └── v2.py
└── tests/
    ├── api_v1/
    ├── api_v2/
    └── api_v3/
```

#### Team Structure Considerations
- **Version Teams**: Dedicated teams per API version for large organizations
- **Shared Services**: Common business logic in shared services layer
- **Backward Compatibility**: Team responsible for maintaining older versions
- **Migration Support**: Cross-team coordination for version transitions

#### Database Architecture
```python
# models.py - Shared across versions
class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    # Version-specific fields
    legacy_id = models.CharField(max_length=50, null=True, blank=True)  # v1 compatibility
    published_date = models.DateTimeField(null=True, blank=True)  # v2 feature
    
    class Meta:
        db_table = 'articles'

# services.py - Shared business logic
class ArticleService:
    @staticmethod
    def get_articles_for_user(user, version='v1'):
        queryset = Article.objects.all()
        
        if version == 'v1':
            return queryset.filter(author=user)
        elif version == 'v2':
            return queryset.filter(author=user, published_date__isnull=False)
        else:
            return queryset.filter(
                author=user,
                published_date__isnull=False
            ).select_related('author')
```

### 2. Architectural Patterns for Versioning

#### Pattern 1: Parallel Version Architecture
**Best for:** Large-scale applications with significant version differences

```python
# config/urls.py
urlpatterns = [
    # Version 1 - Legacy support
    path('api/v1/', include('apps.articles.api.v1.urls')),
    path('api/v1/', include('apps.users.api.v1.urls')),
    
    # Version 2 - Current stable
    path('api/v2/', include('apps.articles.api.v2.urls')),
    path('api/v2/', include('apps.users.api.v2.urls')),
    
    # Version 3 - Beta/Next
    path('api/v3/', include('apps.articles.api.v3.urls')),
    path('api/v3/', include('apps.users.api.v3.urls')),
]

# apps/articles/api/v1/views/article_views.py
class ArticleViewSetV1(viewsets.ModelViewSet):
    """Legacy version with basic CRUD"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializerV1
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        return Article.objects.filter(author=self.request.user)

# apps/articles/api/v2/views/article_views.py
class ArticleViewSetV2(viewsets.ModelViewSet):
    """Enhanced version with filtering and pagination"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializerV2
    permission_classes = [IsAuthenticated]
    filterset_fields = ['status', 'category']
    search_fields = ['title', 'content']
    
    def get_queryset(self):
        return Article.objects.filter(
            author=self.request.user,
            published_date__isnull=False
        )
```

#### Pattern 2: Shared Core Architecture
**Best for:** Applications with incremental changes between versions

```python
# apps/articles/api/views/base_article_views.py
class BaseArticleViewSet(viewsets.ModelViewSet):
    """Base viewset with shared logic"""
    queryset = Article.objects.all()
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        base_queryset = Article.objects.filter(author=self.request.user)
        return self.apply_version_filters(base_queryset)
    
    def apply_version_filters(self, queryset):
        version = getattr(self.request, 'version', 'v1')
        
        if version in ['v2', 'v3']:
            queryset = queryset.filter(published_date__isnull=False)
        
        if version == 'v3':
            queryset = queryset.select_related('author')
        
        return queryset

# apps/articles/api/v1/views/article_views.py
from ...views.base_article_views import BaseArticleViewSet

class ArticleViewSetV1(BaseArticleViewSet):
    serializer_class = ArticleSerializerV1

# apps/articles/api/v2/views/article_views.py
from ...views.base_article_views import BaseArticleViewSet

class ArticleViewSetV2(BaseArticleViewSet):
    serializer_class = ArticleSerializerV2
    filterset_fields = ['status', 'category']

# apps/articles/api/v3/views/article_views.py
from ...views.base_article_views import BaseArticleViewSet

class ArticleViewSetV3(BaseArticleViewSet):
    serializer_class = ArticleSerializerV3
    filterset_fields = ['status', 'category', 'tags']
    search_fields = ['title', 'content', 'summary']
```

#### Pattern 3: Adapter Pattern Architecture
**Best for:** Applications requiring backward compatibility with minimal duplication

```python
# apps/articles/services.py
class ArticleService:
    @staticmethod
    def get_article_data(article, version='v1'):
        """Return article data in version-specific format"""
        base_data = {
            'id': article.id,
            'title': article.title,
            'content': article.content,
            'created_at': article.created_at.isoformat(),
        }
        
        if version == 'v1':
            return ArticleAdapterV1.adapt(base_data, article)
        elif version == 'v2':
            return ArticleAdapterV2.adapt(base_data, article)
        else:
            return ArticleAdapterV3.adapt(base_data, article)

class ArticleAdapterV1:
    @staticmethod
    def adapt(base_data, article):
        """Adapt data for v1 API format"""
        data = base_data.copy()
        data['user'] = article.author.id  # v1 uses 'user' instead of 'author'
        data['timestamp'] = int(article.created_at.timestamp())  # v1 uses timestamp
        data.pop('created_at', None)  # v1 doesn't have created_at
        return data

class ArticleAdapterV2:
    @staticmethod
    def adapt(base_data, article):
        """Adapt data for v2 API format"""
        data = base_data.copy()
        data['author'] = {
            'id': article.author.id,
            'username': article.author.username,
        }
        data['published_date'] = article.published_date.isoformat() if article.published_date else None
        return data

# apps/articles/api/views/article_views.py
class ArticleViewSet(viewsets.ModelViewSet):
    """Single viewset serving all versions"""
    
    def get_serializer_class(self):
        version = getattr(self.request, 'version', 'v1')
        return {
            'v1': ArticleSerializerV1,
            'v2': ArticleSerializerV2,
            'v3': ArticleSerializerV3,
        }.get(version, ArticleSerializerV1)
    
    def retrieve(self, request, *args, **kwargs):
        article = self.get_object()
        version = getattr(request, 'version', 'v1')
        data = ArticleService.get_article_data(article, version)
        return Response(data)
```

### 3. Architectural Decision Framework

#### Scale Considerations
```python
# Small Application (< 10 endpoints)
# Pattern: Shared Core with simple versioning
class SmallAppArchitecture:
    versioning_strategy = "URLPathVersioning"
    code_organization = "Single views.py with version checks"
    team_structure = "1-2 developers"
    
# Medium Application (10-50 endpoints)
# Pattern: Parallel Version with shared services
class MediumAppArchitecture:
    versioning_strategy = "URLPathVersioning"
    code_organization = "Version-specific modules"
    team_structure = "Small team with version responsibilities"
    
# Large Application (> 50 endpoints)
# Pattern: Full Parallel Architecture
class LargeAppArchitecture:
    versioning_strategy = "URLPathVersioning or HostNameVersioning"
    code_organization = "Separate apps per version"
    team_structure = "Dedicated version teams"
```

#### Migration Architecture
```python
# Migration Strategy Architecture
class MigrationArchitecture:
    def __init__(self, current_version, target_version):
        self.current_version = current_version
        self.target_version = target_version
        self.migration_period = timedelta(days=180)
    
    def setup_parallel_support(self):
        """Run both versions during migration"""
        return {
            'v1_status': 'deprecated',
            'v2_status': 'active',
            'v3_status': 'beta',
            'migration_end': datetime.now() + self.migration_period
        }
    
    def create_adapters(self):
        """Create compatibility adapters"""
        return {
            'v1_to_v2': AdapterV1ToV2(),
            'v2_to_v3': AdapterV2ToV3(),
        }
```

## Why Version Your API

### Key Reasons
1. **Backward Compatibility:** Existing clients continue working while you introduce changes
2. **Controlled Evolution:** Roll out breaking changes without disrupting users
3. **Parallel Support:** Run multiple versions simultaneously during migration periods
4. **Clear Communication:** Signal changes to API consumers
5. **Architectural Evolution:** Allow fundamental changes to data models and business logic

### When to Version
- Breaking changes to request/response formats
- Removing endpoints or fields
- Changing authentication methods
- Modifying business logic that affects behavior
- Renaming resources or fields
- Database schema changes affecting API contracts
- Performance optimizations requiring different response structures

### 4. API Design Principles for Versioned APIs

Based on core API design principles, versioned APIs should maintain consistency across versions:

#### RESTful URI Design Across Versions
```python
# Consistent resource naming across versions
# v1
GET /api/v1/articles/
GET /api/v1/articles/123/
POST /api/v1/articles/

# v2 - Same resource names, enhanced functionality
GET /api/v2/articles/
GET /api/v2/articles/123/
POST /api/v2/articles/

# Avoid this - changing resource names between versions
# BAD: GET /api/v2/posts/ (changed from articles)
```

#### Consistent HTTP Verb Usage
```python
class ArticleViewSetV1(viewsets.ModelViewSet):
    """v1 - Basic CRUD following REST principles"""
    
    def list(self, request):
        # GET /api/v1/articles/ - List articles
        return super().list(request)
    
    def create(self, request):
        # POST /api/v1/articles/ - Create article
        return super().create(request)
    
    def retrieve(self, request, pk=None):
        # GET /api/v1/articles/123/ - Get specific article
        return super().retrieve(request, pk)
    
    def update(self, request, pk=None):
        # PUT /api/v1/articles/123/ - Update article
        return super().update(request, pk)
    
    def partial_update(self, request, pk=None):
        # PATCH /api/v1/articles/123/ - Partial update
        return super().partial_update(request, pk)
    
    def destroy(self, request, pk=None):
        # DELETE /api/v1/articles/123/ - Delete article
        return super().destroy(request, pk)

class ArticleViewSetV2(ArticleViewSetV1):
    """v2 - Enhanced but maintains REST principles"""
    
    def list(self, request):
        # Enhanced filtering but same endpoint
        queryset = self.get_queryset()
        filtered_queryset = self.apply_filters(queryset)
        return Response(self.get_serializer(filtered_queryset, many=True).data)
```

#### Consistent Response Format Evolution
```python
# v1 Response Format
{
    "id": 1,
    "title": "Article Title",
    "content": "Article content",
    "user": 123,  # v1 uses 'user'
    "timestamp": 1640995200  # v1 uses timestamp
}

# v2 Response Format - Enhanced but backward compatible structure
{
    "id": 1,
    "title": "Article Title",
    "content": "Article content",
    "author": {  # Enhanced author object
        "id": 123,
        "username": "author_name"
    },
    "created_at": "2021-12-31T23:59:59Z",  # ISO 8601 format
    "published_date": "2022-01-01T10:00:00Z",  # New field
    "metadata": {  # New metadata section
        "word_count": 1500,
        "reading_time": 5
    }
}
```

#### Error Handling Consistency
```python
class BaseAPIView(APIView):
    """Base error handling for all versions"""
    
    def handle_exception(self, exc):
        """Consistent error format across versions"""
        response = super().handle_exception(exc)
        
        error_format = {
            "error": {
                "code": response.status_code,
                "message": response.data.get('detail', 'An error occurred'),
                "timestamp": timezone.now().isoformat(),
            }
        }
        
        # Add version-specific error details
        if getattr(self.request, 'version', 'v1') in ['v2', 'v3']:
            error_format["error"]["request_id"] = getattr(self.request, 'id', None)
        
        response.data = error_format
        return response
```

#### Pagination Consistency
```python
# v1 Pagination
{
    "count": 100,
    "next": "http://api.example.com/api/v1/articles/?page=2",
    "previous": null,
    "results": [...]
}

# v2 Pagination - Enhanced but consistent structure
{
    "count": 100,
    "next": "http://api.example.com/api/v2/articles/?page=2",
    "previous": null,
    "results": [...],
    "pagination": {  # Enhanced pagination info
        "page": 1,
        "page_size": 20,
        "total_pages": 5,
        "has_next": true,
        "has_previous": false
    }
}
```

### 5. Architectural Best Practices Summary

#### Code Organization Principles
1. **Separate Concerns**: Keep version-specific logic isolated
2. **Share Business Logic**: Common operations in shared services
3. **Consistent Structure**: Maintain similar patterns across versions
4. **Clear Boundaries**: Well-defined interfaces between versions

#### Team Organization Principles
1. **Version Ownership**: Clear responsibility for each version
2. **Cross-Version Coordination**: Teams must coordinate breaking changes
3. **Knowledge Sharing**: Documentation and patterns shared across teams
4. **Migration Planning**: Joint planning for version transitions

#### Technical Principles
1. **Backward Compatibility**: Don't break existing clients without deprecation
2. **Forward Compatibility**: Design for future changes when possible
3. **Performance**: Versioning shouldn't significantly impact performance
4. **Security**: Each version must maintain security standards

## Recommended Versioning Strategy: URL Path Versioning

### Why URL Path Versioning is the Best Choice

**URL Path Versioning** is the industry standard and recommended approach for most APIs. With 80%+ adoption by major companies like Twitter, Stripe, and GitHub, it provides the best balance of clarity, usability, and maintainability.

### Implementation

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2', 'v3'],
}

# urls.py
urlpatterns = [
    path('api/v1/', include('myapp.api.v1.urls')),
    path('api/v2/', include('myapp.api.v2.urls')),
    path('api/v3/', include('myapp.api.v3.urls')),
]
```

**Usage:**
```http
GET /api/v1/articles/
GET /api/v2/articles/
GET /api/v3/articles/
```

### Key Benefits

#### 1. **Maximum Visibility**
- Version is clearly visible in the URL
- Easy to identify which version is being used
- No hidden headers or parameters
- Clear communication to API consumers

#### 2. **Developer Experience**
- Easy to test in browser
- Simple to understand and debug
- Works with all HTTP clients
- No special tooling required

#### 3. **Infrastructure Benefits**
- Cacheable at CDN/proxy level
- Simple URL routing
- Easy load balancing
- Works with existing web infrastructure

#### 4. **Documentation & Testing**
- Easy to document
- Simple to write tests
- Clear endpoint identification
- Straightforward API exploration

### Real-World Examples

- **Twitter API**: `/1.1/statuses/update.json`
- **Stripe API**: `/v1/charges`
- **GitHub API**: `/api/v3/users`
- **Slack API**: `/api/chat.postMessage`

### Best Practices for URL Path Versioning

#### 1. **Always Use Semantic Versioning**
```python
# Good - Semantic versions
/api/v1/articles/
/api/v2/articles/
/api/v3/articles/

# Avoid - Non-semantic versions
/api/articles/v1/
/api/v1.2/articles/
```

#### 2. **Maintain Consistent URL Structure**
```python
# Good - Consistent structure
/api/v1/articles/
/api/v1/users/
/api/v2/articles/
/api/v2/users/

# Avoid - Inconsistent structure
/api/v1/articles/
/api/users/v1/
```

#### 3. **Force Version Specification**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': None,  # Force clients to specify version
    'ALLOWED_VERSIONS': ['v1', 'v2', 'v3'],
}
```

#### 4. **Use Parallel URL Configuration**
```python
# config/urls.py
urlpatterns = [
    # Version 1 - Legacy support
    path('api/v1/', include('apps.articles.api.v1.urls')),
    path('api/v1/', include('apps.users.api.v1.urls')),
    
    # Version 2 - Current stable
    path('api/v2/', include('apps.articles.api.v2.urls')),
    path('api/v2/', include('apps.users.api.v2.urls')),
    
    # Version 3 - Beta/Next
    path('api/v3/', include('apps.articles.api.v3.urls')),
    path('api/v3/', include('apps.users.api.v3.urls')),
]
```

### When to Use URL Path Versioning

✅ **Use URL Path Versioning for:**
- Public APIs (most common use case)
- Third-party integrations
- Mobile applications
- Single Page Applications (React, Vue, Angular)
- Web applications
- Internal APIs
- Development and testing
- Documentation and examples

### URL Path Versioning with Different Architectures

#### Parallel Version Architecture
```python
# apps/articles/api/v1/urls.py
from apps.articles.api.v1.views.article_views import ArticleViewSetV1

urlpatterns = [
    path('articles/', ArticleViewSetV1.as_view({'get': 'list'})),
    path('articles/<int:pk>/', ArticleViewSetV1.as_view({'get': 'retrieve'})),
]

# apps/articles/api/v2/urls.py
from apps.articles.api.v2.views.article_views import ArticleViewSetV2

urlpatterns = [
    path('articles/', ArticleViewSetV2.as_view({'get': 'list'})),
    path('articles/<int:pk>/', ArticleViewSetV2.as_view({'get': 'retrieve'})),
]
```

#### Shared Core Architecture
```python
# apps/articles/api/urls.py
from apps.articles.api.views.base_article_views import BaseArticleViewSet

urlpatterns = [
    path('articles/', BaseArticleViewSet.as_view({'get': 'list'})),
    path('articles/<int:pk>/', BaseArticleViewSet.as_view({'get': 'retrieve'})),
]

# config/urls.py
urlpatterns = [
    path('api/v1/', include('apps.articles.api.urls')),
    path('api/v2/', include('apps.articles.api.urls')),
]
```

## Implementing Version-Specific Logic with URL Path Versioning

### Using request.version in Views

```python
from rest_framework.views import APIView
from rest_framework.response import Response

class ArticleListView(APIView):
    def get(self, request):
        articles = Article.objects.all()
        
        if request.version == 'v1':
            serializer = ArticleSerializerV1(articles, many=True)
        elif request.version == 'v2':
            serializer = ArticleSerializerV2(articles, many=True)
        else:
            serializer = ArticleSerializerV3(articles, many=True)
        
        return Response(serializer.data)
```

### Version-Specific Serializers

```python
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return ArticleSerializerV1
        elif self.request.version == 'v2':
            return ArticleSerializerV2
        return ArticleSerializerV3
```

### Version-Specific Querysets

```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    
    def get_queryset(self):
        if self.request.version == 'v1':
            # v1 returns all articles
            return Article.objects.all()
        elif self.request.version == 'v2':
            # v2 returns only published articles
            return Article.objects.filter(published=True)
        else:
            # v3 returns published articles with related data
            return Article.objects.filter(published=True).select_related('author')
```

## Versioning Best Practices

### 1. Make Versioning Mandatory

Always require a version. Don't release unversioned APIs.

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': None,  # Force clients to specify version
    'ALLOWED_VERSIONS': ['v1', 'v2', 'v3'],
}
```

### 2. Use Semantic Versioning

Follow a clear versioning scheme:
- **Major version (v1, v2, v3):** Breaking changes
- **Minor version (v1.1, v1.2):** Backward-compatible features
- **Patch version (v1.1.1):** Bug fixes

**Example:**
```
/api/v1/articles/      - Major version
/api/v1.2/articles/    - Minor version
/api/v1.2.1/articles/  - Patch version
```

### 3. Maintain Parallel Versions

Run old and new versions in parallel during transition.

```python
# urls.py
urlpatterns = [
    path('api/v1/', include('myapp.api.v1.urls')),  # Deprecated
    path('api/v2/', include('myapp.api.v2.urls')),  # Current
    path('api/v3/', include('myapp.api.v3.urls')),  # Beta
]
```

### 4. Communicate Deprecation

Use HTTP headers to communicate deprecation.

```python
from rest_framework.response import Response
from datetime import datetime, timedelta

class DeprecatedAPIView(APIView):
    def get(self, request):
        response = Response({'data': 'some data'})
        
        # Deprecation warning
        response['Warning'] = '299 - "This API version is deprecated. Please migrate to v2."'
        
        # Sunset date
        sunset_date = datetime.now() + timedelta(days=180)
        response['Sunset'] = sunset_date.strftime('%a, %d %b %Y %H:%M:%S GMT')
        
        # Link to new version
        response['Link'] = '</api/v2/>; rel="successor-version"'
        
        return response
```

### 5. Document Version Changes

Maintain a comprehensive changelog.

```markdown
# API Changelog

## v3 (2024-01-15)
### Breaking Changes
- Removed `author_name` field from Article response
- Changed `created_at` format from timestamp to ISO 8601

### New Features
- Added pagination to all list endpoints
- Added filtering support for articles

### Deprecated
- `/api/v2/` endpoints will be sunset on 2024-07-15

## v2 (2023-06-01)
### Breaking Changes
- Renamed `user` field to `author` in Article model
- Changed authentication from Basic to Token

### New Features
- Added article comments endpoint
- Added user profile endpoint
```

### 6. Set Deprecation Timeline

Establish clear timelines for version support.

```python
API_VERSION_SUPPORT = {
    'v1': {
        'status': 'deprecated',
        'released': '2022-01-01',
        'deprecated': '2023-06-01',
        'sunset': '2024-01-01',
    },
    'v2': {
        'status': 'active',
        'released': '2023-06-01',
        'deprecated': None,
        'sunset': None,
    },
    'v3': {
        'status': 'beta',
        'released': '2024-01-01',
        'deprecated': None,
        'sunset': None,
    },
}
```

### 7. Version Documentation

Use tools like drf-spectacular or drf-yasg to generate version-specific documentation.

```python
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    # v1 documentation
    path('api/v1/schema/', SpectacularAPIView.as_view(api_version='v1'), name='schema-v1'),
    path('api/v1/docs/', SpectacularSwaggerView.as_view(url_name='schema-v1'), name='docs-v1'),
    
    # v2 documentation
    path('api/v2/schema/', SpectacularAPIView.as_view(api_version='v2'), name='schema-v2'),
    path('api/v2/docs/', SpectacularSwaggerView.as_view(url_name='schema-v2'), name='docs-v2'),
]
```

## Reversing URLs for Versioned APIs

### Include Request in reverse()

```python
from rest_framework.reverse import reverse

class ArticleViewSet(viewsets.ModelViewSet):
    def list(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(
            articles,
            many=True,
            context={'request': request}
        )
        
        # Generate versioned URL
        next_url = reverse('article-list', request=request)
        
        return Response({
            'results': serializer.data,
            'next': next_url
        })
```

### Hyperlinked Serializers with Versioning

```python
class ArticleSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Article
        fields = ['url', 'title', 'author', 'created_at']

# In view, always pass request context
def get(self, request):
    articles = Article.objects.all()
    serializer = ArticleSerializer(
        articles,
        many=True,
        context={'request': request}  # Required for versioned URLs
    )
    return Response(serializer.data)
```

## Migration Strategies

### Gradual Migration

```python
class ArticleViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        # Support both v1 and v2 during migration
        if self.request.version in ['v1', 'v2']:
            return ArticleSerializerV1V2Compatible
        return ArticleSerializerV3
```

### Feature Flags

```python
class ArticleViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Article.objects.all()
        
        # Enable new feature only for v2+
        if self.request.version in ['v2', 'v3']:
            queryset = queryset.prefetch_related('tags')
        
        return queryset
```

### Adapter Pattern

```python
class ArticleAdapterV1:
    """Adapt v2 data to v1 format"""
    @staticmethod
    def adapt(data):
        # Convert v2 format to v1 format
        data['user'] = data.pop('author')
        data['timestamp'] = int(data.pop('created_at').timestamp())
        return data

class ArticleViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        article = self.get_object()
        serializer = ArticleSerializerV2(article)
        data = serializer.data
        
        if request.version == 'v1':
            data = ArticleAdapterV1.adapt(data)
        
        return Response(data)
```

## Testing Versioned APIs

```python
from rest_framework.test import APITestCase

class ArticleAPIVersioningTest(APITestCase):
    def test_v1_response_format(self):
        response = self.client.get('/api/v1/articles/')
        self.assertEqual(response.status_code, 200)
        self.assertIn('user', response.data[0])  # v1 uses 'user'
    
    def test_v2_response_format(self):
        response = self.client.get('/api/v2/articles/')
        self.assertEqual(response.status_code, 200)
        self.assertIn('author', response.data[0])  # v2 uses 'author'
    
    def test_version_header(self):
        response = self.client.get(
            '/api/articles/',
            HTTP_ACCEPT='application/json; version=v2'
        )
        self.assertEqual(response.status_code, 200)
    
    def test_invalid_version(self):
        response = self.client.get('/api/v99/articles/')
        self.assertEqual(response.status_code, 404)
```

## Summary

### Architectural Best Practices

1. **Choose Architecture Pattern**: Parallel Version for large apps, Shared Core for incremental changes, Adapter for compatibility
2. **Organize Code Structure**: Separate version modules with shared services and business logic
3. **Design for Scale**: Consider team structure, database architecture, and migration strategies
4. **Maintain Consistency**: Apply RESTful principles consistently across all versions
5. **Plan for Evolution**: Design architecture to support future version requirements

### Versioning Strategy Best Practices

6. **Use URL Path Versioning**: Industry standard with 80%+ adoption, maximum visibility and developer experience
7. **Make Mandatory**: Always require version specification, no unversioned APIs
8. **Semantic Versioning**: Use clear major.minor.patch versioning scheme
9. **Parallel Support**: Run multiple versions simultaneously during transitions
10. **Communicate Changes**: Use deprecation headers and maintain changelog
11. **Set Timelines**: Establish clear deprecation and sunset dates
12. **Document Thoroughly**: Provide version-specific documentation
13. **Test Versions**: Write tests for each API version
14. **Plan Migration**: Provide clear migration paths for clients
15. **Version Early**: Start with v1, don't wait until you need to version

### Implementation Best Practices

16. **Share Business Logic**: Keep common operations in shared services layer
17. **Version-Specific Serializers**: Use different serializers for different response formats
18. **Consistent Error Handling**: Maintain error format consistency across versions
19. **Performance Considerations**: Ensure versioning doesn't impact performance significantly
20. **Security Standards**: Each version must maintain security standards

### Decision Framework

| Application Size | Recommended Architecture | Team Structure |
|------------------|-------------------------|---------------|
| Small (< 10 endpoints) | Shared Core | 1-2 developers |
| Medium (10-50 endpoints) | Parallel Version | Small team with version responsibilities |
| Large (> 50 endpoints) | Full Parallel | Dedicated version teams |

### Key Takeaways

- **Architecture First**: Consider project structure and team organization before implementing versioning
- **URL Path Versioning**: Use the industry standard approach for maximum visibility and developer experience
- **Consistency Matters**: Apply API design principles consistently across all versions
- **Plan for Migration**: Design architecture to support smooth version transitions
- **Share Wisely**: Share business logic but isolate version-specific concerns
- **Communicate Clearly**: Use deprecation headers, changelogs, and clear documentation

Following these architectural practices with URL Path Versioning ensures your API can evolve smoothly while maintaining backward compatibility and providing clear migration paths for clients. The right architectural approach combined with URL Path Versioning creates a scalable, maintainable API that serves both current and future needs.
