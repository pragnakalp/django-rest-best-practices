# API Documentation Best Practices

## Overview
Comprehensive API documentation is crucial for developer experience and API adoption. This document covers best practices for documenting Django REST Framework APIs.

## Why Documentation Matters

### Key Benefits
1. **Developer Onboarding:** Reduces time to first successful API call
2. **Self-Service:** Reduces support burden
3. **API Adoption:** Well-documented APIs are more likely to be used
4. **Internal Alignment:** Helps teams understand API capabilities
5. **Contract Definition:** Serves as agreement between API provider and consumers

## Documentation Tools

### 1. drf-spectacular (Recommended)

Modern OpenAPI 3 schema generation with excellent DRF integration.

**Installation:**
```bash
pip install drf-spectacular
```

**Configuration:**
```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'drf_spectacular',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'DESCRIPTION': 'Comprehensive API for managing articles and users',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'SCHEMA_PATH_PREFIX': r'/api/v[0-9]',
    'COMPONENT_SPLIT_REQUEST': True,
}
```

**URLs:**
```python
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView

urlpatterns = [
    # Schema endpoint
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    
    # Swagger UI
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    
    # ReDoc UI
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

**Enhanced Documentation:**
```python
from drf_spectacular.utils import extend_schema, extend_schema_view, OpenApiParameter, OpenApiExample
from drf_spectacular.types import OpenApiTypes

@extend_schema_view(
    list=extend_schema(
        summary="List all articles",
        description="Retrieve a paginated list of all articles. Supports filtering and searching.",
        parameters=[
            OpenApiParameter(
                name='published',
                type=OpenApiTypes.BOOL,
                location=OpenApiParameter.QUERY,
                description='Filter by publication status',
            ),
            OpenApiParameter(
                name='author',
                type=OpenApiTypes.INT,
                location=OpenApiParameter.QUERY,
                description='Filter by author ID',
            ),
        ],
        responses={200: ArticleSerializer(many=True)},
        tags=['articles'],
    ),
    create=extend_schema(
        summary="Create a new article",
        description="Create a new article. Requires authentication.",
        request=ArticleSerializer,
        responses={201: ArticleSerializer},
        examples=[
            OpenApiExample(
                'Article Example',
                value={
                    'title': 'My First Article',
                    'content': 'This is the article content...',
                    'published': False
                },
                request_only=True,
            ),
        ],
        tags=['articles'],
    ),
)
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
```

### 2. drf-yasg

Swagger/OpenAPI 2.0 generation tool.

**Installation:**
```bash
pip install drf-yasg
```

**Configuration:**
```python
# settings.py
INSTALLED_APPS = [
    # ...
    'drf_yasg',
]
```

**URLs:**
```python
from rest_framework import permissions
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
    openapi.Info(
        title="My API",
        default_version='v1',
        description="API documentation",
        terms_of_service="https://www.example.com/terms/",
        contact=openapi.Contact(email="contact@example.com"),
        license=openapi.License(name="BSD License"),
    ),
    public=True,
    permission_classes=[permissions.AllowAny],
)

urlpatterns = [
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
    path('redoc/', schema_view.with_ui('redoc', cache_timeout=0), name='schema-redoc'),
]
```

## Documenting Serializers

### Add Field Help Text

```python
class ArticleSerializer(serializers.ModelSerializer):
    title = serializers.CharField(
        max_length=200,
        help_text="Article title (max 200 characters)"
    )
    content = serializers.CharField(
        help_text="Main article content in Markdown format"
    )
    published = serializers.BooleanField(
        default=False,
        help_text="Whether the article is published and visible to public"
    )
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'published', 'created_at']
        read_only_fields = ['id', 'created_at']
```

### Document Validation Rules

```python
class ArticleSerializer(serializers.ModelSerializer):
    title = serializers.CharField(
        max_length=200,
        min_length=5,
        help_text="Article title (5-200 characters)"
    )
    tags = serializers.ListField(
        child=serializers.CharField(max_length=50),
        max_length=10,
        help_text="List of tags (max 10 tags, each max 50 characters)"
    )
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'tags']
```

## Documenting Views

### Add Docstrings

```python
class ArticleViewSet(viewsets.ModelViewSet):
    """
    ViewSet for managing articles.
    
    Provides standard CRUD operations for articles:
    - list: Get all articles (supports filtering and pagination)
    - create: Create a new article (requires authentication)
    - retrieve: Get a specific article
    - update: Update an article (requires ownership or admin)
    - partial_update: Partially update an article
    - destroy: Delete an article (requires ownership or admin)
    """
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def list(self, request, *args, **kwargs):
        """
        List all articles.
        
        Returns a paginated list of articles. Supports filtering by:
        - published: Filter by publication status (true/false)
        - author: Filter by author ID
        - search: Search in title and content
        
        Example:
            GET /api/articles/?published=true&author=1
        """
        return super().list(request, *args, **kwargs)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Publish an article.
        
        Changes the article status to published and sets the publication date.
        Requires article ownership or admin privileges.
        
        Returns:
            200: Article successfully published
            403: Permission denied
            404: Article not found
        """
        article = self.get_object()
        article.published = True
        article.published_date = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
```

### Document with extend_schema

```python
from drf_spectacular.utils import extend_schema

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    @extend_schema(
        summary="Publish an article",
        description="""
        Changes the article status to published and sets the publication date.
        
        **Requirements:**
        - User must be authenticated
        - User must be the article author or an admin
        
        **Side Effects:**
        - Sets `published` to `true`
        - Sets `published_date` to current timestamp
        - Sends notification emails to subscribers
        """,
        responses={
            200: ArticleSerializer,
            403: OpenApiResponse(description='Permission denied'),
            404: OpenApiResponse(description='Article not found'),
        },
        tags=['articles', 'publishing'],
    )
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        article = self.get_object()
        article.published = True
        article.published_date = timezone.now()
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
```

## Documenting Authentication

### Document Authentication Requirements

```python
from drf_spectacular.utils import extend_schema

@extend_schema(
    summary="Create a new article",
    description="""
    Create a new article.
    
    **Authentication:** Required (JWT Bearer Token)
    
    **Headers:**
    ```
    Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
    ```
    
    The authenticated user will be set as the article author.
    """,
    request=ArticleSerializer,
    responses={
        201: ArticleSerializer,
        401: OpenApiResponse(description='Authentication required'),
        400: OpenApiResponse(description='Validation error'),
    },
)
def create(self, request, *args, **kwargs):
    return super().create(request, *args, **kwargs)
```

### Document JWT Authentication Endpoints

```python
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
from drf_spectacular.utils import extend_schema, OpenApiResponse

class CustomTokenObtainPairView(TokenObtainPairView):
    """
    Obtain JWT tokens for authentication.
    
    Request body should contain username and password.
    Returns access and refresh tokens on success.
    
    Usage: Include the access token in the Authorization header:
    Authorization: Bearer <access_token>
    
    Token Refresh: Use the refresh token endpoint to obtain new access tokens.
    """
    
    @extend_schema(
        responses={
            200: OpenApiResponse(description='JWT tokens obtained successfully'),
            401: OpenApiResponse(description='Invalid credentials'),
        }
    )
    def post(self, request, *args, **kwargs):
        return super().post(request, *args, **kwargs)

class CustomTokenRefreshView(TokenRefreshView):
    """
    Refresh JWT access token.
    
    Accepts a refresh token and returns a new access token.
    """
    
    @extend_schema(
        responses={
            200: OpenApiResponse(description='New access token generated'),
            401: OpenApiResponse(description='Invalid or expired refresh token'),
        }
    )
    def post(self, request, *args, **kwargs):
        return super().post(request, *args, **kwargs)
```

### Legacy Token Authentication (Not Recommended)

> **⚠️ Note:** Token authentication is legacy. For new projects, use JWT authentication as documented above.

If maintaining a legacy system with Token authentication, document it as follows:

```python
from rest_framework.authtoken.views import ObtainAuthToken

class CustomAuthToken(ObtainAuthToken):
    """
    Obtain authentication token (LEGACY - Use JWT for new projects).
    
    **Usage:**
    ```
    Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
    ```
    """
    pass
```

## Documenting Error Responses

### Standard Error Format

> **📖 Reference:** See `09-Error-Handling-and-Status-Codes.md` for the complete error handling strategy.

All error responses follow a consistent format:

```python
class ArticleSerializer(serializers.ModelSerializer):
    """
    Article serializer.
    
    All errors follow the standard format from 09-Error-Handling-and-Status-Codes.md
    with error code, message, and optional fields object.
    
    Error responses include:
    - 400 Bad Request: Validation errors with field-specific messages
    - 401 Unauthorized: Authentication required
    - 403 Forbidden: Permission denied
    - 404 Not Found: Resource not found
    
    Note: Always use 400 for validation errors, never 422.
    """
    class Meta:
        model = Article
        fields = ['id', 'title', 'content']  # ✅ Explicit fields (never use '__all__')
```

### Document Custom Error Responses

```python
from drf_spectacular.utils import extend_schema, OpenApiResponse

@extend_schema(
    responses={
        200: ArticleSerializer,
        400: OpenApiResponse(
            description='Validation Error',
            examples=[
                OpenApiExample(
                    'Validation Error Example',
                    value={
                        'title': ['This field is required.'],
                        'content': ['Article content is too short.']
                    }
                )
            ]
        ),
        403: OpenApiResponse(
            description='Permission Denied',
            examples=[
                OpenApiExample(
                    'Permission Error',
                    value={'detail': 'You must be the author to edit this article.'}
                )
            ]
        ),
    }
)
def update(self, request, *args, **kwargs):
    return super().update(request, *args, **kwargs)
```

## Request/Response Examples

### Add Examples to Serializers

```python
from drf_spectacular.utils import extend_schema_serializer, OpenApiExample

@extend_schema_serializer(
    examples=[
        OpenApiExample(
            'Article Create Example',
            value={
                'title': 'Getting Started with Django REST Framework',
                'content': 'Django REST Framework is a powerful toolkit...',
                'published': False,
                'tags': ['django', 'python', 'api']
            },
            request_only=True,
        ),
        OpenApiExample(
            'Article Response Example',
            value={
                'id': 1,
                'title': 'Getting Started with Django REST Framework',
                'content': 'Django REST Framework is a powerful toolkit...',
                'author': {
                    'id': 1,
                    'username': 'john_doe'
                },
                'published': True,
                'tags': ['django', 'python', 'api'],
                'created_at': '2024-01-15T10:30:00Z',
                'updated_at': '2024-01-15T14:20:00Z'
            },
            response_only=True,
        ),
    ]
)
class ArticleSerializer(serializers.ModelSerializer):
    """Article serializer with explicit field selection."""
    
    class Meta:
        model = Article
        # ✅ Always use explicit fields (NEVER use '__all__' or 'exclude')
        # See 02-Serializers-Best-Practices.md for details
        fields = [
            'id', 'title', 'content', 'author', 
            'published', 'tags', 'created_at', 'updated_at'
        ]
```

## Pagination Documentation

### Document Pagination Format

```python
class StandardResultsSetPagination(PageNumberPagination):
    """
    Standard pagination for list endpoints.
    
    Query Parameters:
    - page: Page number (default: 1)
    - page_size: Number of results per page (default: 20, max: 100)
    
    Response includes count, next, previous, and results array.
    """
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
```

## Filtering and Searching Documentation

### Document Filter Parameters

```python
from drf_spectacular.utils import extend_schema, OpenApiParameter
from drf_spectacular.types import OpenApiTypes

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['published', 'author', 'category']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    
    @extend_schema(
        parameters=[
            OpenApiParameter(
                name='published',
                type=OpenApiTypes.BOOL,
                location=OpenApiParameter.QUERY,
                description='Filter by publication status',
                examples=[
                    OpenApiExample('Published', value='true'),
                    OpenApiExample('Draft', value='false'),
                ]
            ),
            OpenApiParameter(
                name='author',
                type=OpenApiTypes.INT,
                location=OpenApiParameter.QUERY,
                description='Filter by author ID',
            ),
            OpenApiParameter(
                name='search',
                type=OpenApiTypes.STR,
                location=OpenApiParameter.QUERY,
                description='Search in title and content',
            ),
            OpenApiParameter(
                name='ordering',
                type=OpenApiTypes.STR,
                location=OpenApiParameter.QUERY,
                description='Order results by field (prefix with - for descending)',
                examples=[
                    OpenApiExample('Newest first', value='-created_at'),
                    OpenApiExample('Alphabetical', value='title'),
                ]
            ),
        ]
    )
    def list(self, request, *args, **kwargs):
        """
        List articles with filtering and searching.
        
        **Examples:**
        - Get published articles: `?published=true`
        - Search for Django: `?search=django`
        - Get author's articles: `?author=1`
        - Sort by date: `?ordering=-created_at`
        - Combine filters: `?published=true&author=1&ordering=-created_at`
        """
        return super().list(request, *args, **kwargs)
```

## Rate Limiting Documentation

### Document Rate Limits

```python
from drf_spectacular.utils import extend_schema

class ArticleViewSet(viewsets.ModelViewSet):
    throttle_classes = [UserRateThrottle]
    
    @extend_schema(
        summary="List articles",
        description="""
        Retrieve a paginated list of articles.
        
        Rate Limits (see 05-Security-Best-Practices.md for details):
        - Authenticated users: 1000 requests/hour
        - Anonymous users: 100 requests/hour
        
        Rate Limit Headers:
        - X-RateLimit-Limit: Total requests allowed
        - X-RateLimit-Remaining: Requests remaining
        - X-RateLimit-Reset: Time when limit resets (Unix timestamp)
        
        Returns 429 error with RATE_LIMIT_EXCEEDED code when limit is exceeded.
        """
    )
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

## Versioning Documentation

### Document API Versions

```python
# Create version-specific documentation
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    # v1 documentation
    path('api/v1/schema/', SpectacularAPIView.as_view(
        api_version='v1',
        urlconf='myapp.api.v1.urls'
    ), name='schema-v1'),
    path('api/v1/docs/', SpectacularSwaggerView.as_view(url_name='schema-v1'), name='docs-v1'),
    
    # v2 documentation
    path('api/v2/schema/', SpectacularAPIView.as_view(
        api_version='v2',
        urlconf='myapp.api.v2.urls'
    ), name='schema-v2'),
    path('api/v2/docs/', SpectacularSwaggerView.as_view(url_name='schema-v2'), name='docs-v2'),
]
```

## README and Getting Started

### Create Comprehensive README

```markdown
# My API Documentation

## Overview
This API provides access to articles, users, and comments.

## Base URL
```
https://api.example.com/v1/
```

## Authentication
All endpoints except public article listing require authentication.

### Obtaining JWT Tokens
```bash
curl -X POST https://api.example.com/api/token/ \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your_username",
    "password": "your_password"
  }'
```

Response:
```json
{
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Using JWT Tokens
Include the access token in the Authorization header:
```bash
curl -X GET https://api.example.com/api/v1/articles/ \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Refreshing JWT Tokens
```bash
curl -X POST https://api.example.com/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}'
```

### Legacy Token Authentication (Deprecated)
For backward compatibility, token authentication is still supported:
```bash
curl -X POST https://api.example.com/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{
    "username": "your_username",
    "password": "your_password"
  }'
```

Response:
```json
{
  "token": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
}
```

Using legacy token:
```bash
curl -X GET https://api.example.com/api/v1/articles/ \
  -H "Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
```

## Quick Start

### List Articles
```bash
curl https://api.example.com/api/v1/articles/
```

### Create Article
```bash
curl -X POST https://api.example.com/api/v1/articles/ \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My Article",
    "content": "Article content here",
    "author": 1
  }'
```

### Get Article
```bash
curl https://api.example.com/api/v1/articles/1/
```

## Rate Limits
- Authenticated: 1000 requests/hour
- Anonymous: 100 requests/hour

## Support
- Documentation: https://api.example.com/docs/
- Email: support@example.com
- GitHub: https://github.com/example/api
```

## Interactive Documentation

### Enable Browsable API (Development Only)

```python
# settings.py
if DEBUG:
    REST_FRAMEWORK = {
        'DEFAULT_RENDERER_CLASSES': [
            'rest_framework.renderers.JSONRenderer',
            'rest_framework.renderers.BrowsableAPIRenderer',
        ],
    }
else:
    REST_FRAMEWORK = {
        'DEFAULT_RENDERER_CLASSES': [
            'rest_framework.renderers.JSONRenderer',
        ],
    }
```

## Summary

Key documentation best practices:

1. **Use Modern Tools:** drf-spectacular for OpenAPI 3 documentation
2. **Add Docstrings:** Document all views, serializers, and custom actions
3. **Provide Examples:** Include request/response examples
4. **Document Authentication:** Clearly explain how to authenticate
5. **Error Documentation:** Document all possible error responses
6. **Filter Documentation:** Explain filtering, searching, and ordering
7. **Rate Limits:** Document rate limiting policies
8. **Version Documentation:** Provide version-specific docs
9. **Getting Started:** Create quick start guides and README
10. **Keep Updated:** Update documentation with code changes

Well-documented APIs significantly improve developer experience and adoption rates.
