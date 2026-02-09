# API Design & Architecture Best Practices

## Overview
This document consolidates best practices for designing robust, scalable, and maintainable REST APIs using Django REST Framework.

## Core Principles

### 1. Treat Your API as a Product
APIs are not just technical implementations but strategic products that power applications, integrations, and revenue models.

**Key Actions:**
- Assign a clear API owner (team or squad)
- Maintain an API roadmap (features, deprecations, improvements)
- Set SLAs and KPIs (uptime, latency, errors, adoption metrics)
- Track API usage and performance metrics

**Benefits:**
- Transforms endpoints into strategic assets
- Enables measurable impact and governance
- Facilitates better planning and resource allocation

### 2. RESTful URI Design

#### Use Plural Resource Nouns
Always use plural nouns for resource endpoints to maintain consistency.

**Bad Practice:**
```
GET: /article/:id/
GET: /user/123/
```

**Good Practice:**
```
GET: /articles/:id/
GET: /users/123/
```

**Rationale:** Fits all endpoint types and maintains consistency whether returning single or multiple resources.

#### Avoid Verbs in URIs
HTTP verbs should describe the action; URIs should identify resources.

**Bad Practice:**
```
GET: /articles/:slug/generateBanner/
POST: /users/711/activate
GET: /deleteUser/123
```

**Good Practice:**
```
GET: /articles/:slug/banner/
POST: /users/711/activation
DELETE: /users/123
```

#### Use Consistent URI Case
Adopt **spinal-case** (kebab-case) for URIs as recommended by RFC3986.

**Recommended:**
```
/api/user-profiles/
/api/blog-posts/
/api/order-items/
```

**Avoid:**
- CamelCase: Ineffective in case-insensitive contexts
- snake_case: Less common in web URLs

**Industry Standard:** Used by Google, PayPal, and other major companies.

### 3. Don't Nest Resources Deeply
Avoid deeply nested resource URIs. Use query parameters for filtering instead.

**Bad Practice:**
```
GET: /authors/12/articles/5/comments/3
GET: /users/1/posts/2/comments/3/replies/4
```

**Good Practice:**
```
GET: /articles/?author_id=12
GET: /comments/?article_id=5
GET: /comments/3
```

**Benefits:**
- Simpler URL structure
- Easier to maintain and scale
- More flexible filtering options

### 4. Handle Trailing Slashes Gracefully
Choose one convention (with or without trailing slash) and stick to it. Configure your framework to redirect gracefully.

**Django Configuration:**
```python
# settings.py
APPEND_SLASH = True  # Redirects /api/users to /api/users/
```

**Best Practice:** Most web frameworks have built-in options for this. Enable them to avoid 404 errors.

### 5. Pagination Strategies: Choosing the Right Approach

#### Three Main Pagination Approaches

**Approach 1: Page Number Pagination (Most Common)**
```
GET: /articles/?page=2&page_size=20
```

**Pros:**
- Easy to understand and implement
- Users can jump to any page
- Good for UI with page numbers
- Predictable page count

**Cons:**
- Performance degrades with large offsets
- Inconsistent results if data changes between requests
- Can miss or duplicate items with concurrent writes

**Use When:**
- Building traditional web UIs with page numbers
- Dataset is relatively stable
- Users need to jump to specific pages
- Total count is important to display

**Approach 2: Limit/Offset Pagination**
```
GET: /articles/?limit=20&offset=40
```

**Pros:**
- Flexible (any offset value)
- Simple to implement
- Good for infinite scroll

**Cons:**
- Same performance issues as page number
- Offset can become very large
- Database query cost increases with offset

**Use When:**
- Need flexibility in result positioning
- Implementing infinite scroll
- Dataset is small to medium size

**Approach 3: Cursor Pagination (Best for Large Datasets)**
```
GET: /articles/?cursor=eyJpZCI6MTAwfQ==
```

**Pros:**
- Consistent results even with data changes
- Excellent performance (no offset)
- No duplicate or missing items
- Scales to very large datasets

**Cons:**
- Can't jump to arbitrary pages
- More complex to implement
- Requires ordered field (usually ID or timestamp)
- Can't show total page count

**Use When:**
- Building real-time feeds (Twitter, Facebook style)
- Dataset is very large (millions of records)
- Data changes frequently
- Performance is critical
- Infinite scroll without page numbers

#### Detailed Implementation Comparison

**Page Number Pagination:**
```python
from rest_framework.pagination import PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

# Response:
{
  "count": 1000,
  "next": "http://api.example.com/articles/?page=3",
  "previous": "http://api.example.com/articles/?page=1",
  "results": [...]
}
```

**Cursor Pagination:**
```python
from rest_framework.pagination import CursorPagination

class ArticleCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Required!

# Response:
{
  "next": "http://api.example.com/articles/?cursor=cD0yMDIw...",
  "previous": "http://api.example.com/articles/?cursor=cj0xJnA9MjAyMA==",
  "results": [...]
}
```

#### Performance Comparison

| Dataset Size | Page Number | Cursor | Winner |
|--------------|-------------|--------|--------|
| < 1,000 | Fast | Fast | Page Number (simpler) |
| 1,000 - 10,000 | Medium | Fast | Cursor |
| 10,000 - 100,000 | Slow | Fast | Cursor |
| > 100,000 | Very Slow | Fast | Cursor |
| Real-time feeds | Inconsistent | Consistent | Cursor |

#### Decision Matrix

```
Use Page Number Pagination when:
✓ Building traditional web UIs
✓ Need to show total page count
✓ Users need to jump to specific pages
✓ Dataset < 10,000 records
✓ Data is relatively stable

Use Cursor Pagination when:
✓ Building infinite scroll feeds
✓ Dataset > 10,000 records
✓ Data changes frequently
✓ Performance is critical
✓ Building mobile apps
✓ Real-time or social media style feeds

Use Limit/Offset when:
✓ Need maximum flexibility
✓ Internal APIs with known usage
✓ Prototyping or simple use cases
```

#### Filtering Best Practices
Allow users to filter resources based on specific properties.

```
GET: /articles/?published=true
GET: /articles/?author=john&category=tech
GET: /users/?is_active=true&role=admin
GET: /articles/?created_after=2024-01-01&published=true
```

### 6. Return Proper Content Types
Always set the `Content-Type` header to `application/json` for JSON responses.

**Bad Practice:**
```python
return "{'message': 'success'}"  # Plain text
```

**Good Practice:**
```python
from rest_framework.response import Response

return Response({'message': 'success'})  # Automatically sets Content-Type
```

### 7. Project Structure Best Practices

#### Recommended Structure (Production-Ready with Versioning)
```
project/
├── config/
│   ├── settings/
│   │   ├── __init__.py              # Makes directory a Python package
│   │   ├── base.py                  # Common settings for all environments
│   │   ├── development.py           # Local dev settings (DEBUG=True)
│   │   ├── staging.py               # Pre-prod testing environment
│   │   └── production.py            # Live production settings (DEBUG=False)
│   ├── urls.py                      # Root URL routing to app APIs
│   ├── schema.py                    # drf-spectacular OpenAPI schema config
│   ├── asgi.py                      # ASGI config for async (channels, etc.)
│   └── wsgi.py                      # WSGI config for traditional serving
│
├── apps/
│   ├── users/
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── views/
│   │   │   │   │   ├── __init__.py           # Package init
│   │   │   │   │   ├── auth.py               # Login/register/logout views
│   │   │   │   │   ├── profile.py            # User profile CRUD views
│   │   │   │   │   └── admin.py              # Admin-only user management
│   │   │   │   ├── serializers/
│   │   │   │   │   ├── __init__.py           # Package init
│   │   │   │   │   ├── auth.py               # Auth request/response serialization
│   │   │   │   │   ├── profile.py            # Profile data validation/serialization
│   │   │   │   │   └── admin.py              # Admin user data serialization
│   │   │   │   ├── urls.py                   # v1 API endpoint routing
│   │   │   │   └── permissions.py            # Custom permission classes
│   │   │   └── v2/                           # Future API version (empty)
│   │   ├── models.py                        # User, Profile models (SINGLE FILE)
│   │   ├── managers.py                      # Custom QuerySet methods
│   │   ├── services/
│   │   │   ├── __init__.py                  # Package init
│   │   │   ├── auth_service.py              # Password hash, token generation
│   │   │   └── user_service.py              # User create/update/delete logic
│   │   ├── selectors/
│   │   │   ├── __init__.py                  # Package init
│   │   │   └── user_selector.py             # Complex read queries (filtering)
│   │   ├── validators.py                    # Model-level custom validators
│   │   ├── tasks.py                         # Celery async tasks (email, etc.)
│   │   ├── signals.py                       # post_save, post_delete hooks
│   │   ├── admin.py                         # Django admin interface config
│   │   ├── migrations/                      # Auto-generated DB migrations
│   │   └── tests/
│   │       ├── __init__.py                  # Package init
│   │       ├── test_models.py               # Model validation tests
│   │       ├── test_services.py             # Business logic tests
│   │       ├── test_selectors.py            # Query tests
│   │       ├── test_views.py                # API endpoint tests
│   │       └── factories.py                 # Test data factories (FactoryBoy)
│   │
│   ├── articles/
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── views/
│   │   │   │   │   ├── __init__.py           # Package init
│   │   │   │   │   ├── article.py            # Article CRUD views
│   │   │   │   │   ├── comment.py            # Comment CRUD views
│   │   │   │   │   └── admin.py              # Article admin views
│   │   │   │   ├── serializers/
│   │   │   │   │   ├── __init__.py           # Package init
│   │   │   │   │   ├── article.py            # Article serialization
│   │   │   │   │   ├── comment.py            # Comment serialization
│   │   │   │   │   └── admin.py              # Admin serialization
│   │   │   │   └── urls.py                   # v1 article API routing
│   │   │   └── v2/                           # Future API version
│   │   ├── models.py                        # Article, Comment, Tag (SINGLE FILE)
│   │   ├── managers.py                      # ArticleQuerySet custom methods
│   │   ├── services/
│   │   │   ├── __init__.py                  # Package init
│   │   │   └── article_service.py           # Article create/publish logic
│   │   ├── selectors/
│   │   │   ├── __init__.py                  # Package init
│   │   │   └── article_selector.py          # Article filtering/pagination
│   │   ├── validators.py                    # Article content validation
│   │   ├── tasks.py                         # Async publishing, notifications
│   │   ├── signals.py                       # Article publish hooks
│   │   ├── admin.py                         # Article admin interface
│   │   ├── migrations/                      # DB migrations
│   │   └── tests/                           # Same test structure as users/
│
├── common/
│   ├── api/
│   │   ├── __init__.py                     # Package init
│   │   ├── pagination.py                   # Custom pagination classes
│   │   ├── exceptions.py                   # APIException subclasses
│   │   ├── response.py                     # Standardized API responses
│   │   └── filters.py                      # django-filter configurations
│   ├── models/
│   │   ├── __init__.py                     # Package init
│   │   ├── timestamped.py                  # Abstract TimeStampedModel
│   │   └── soft_delete.py                  # Abstract SoftDeleteModel
│   ├── utils/
│   │   ├── __init__.py                     # Package init
│   │   ├── validators.py                   # Reusable validators
│   │   ├── helpers.py                      # Utility functions
│   │   ├── constants.py                    # App-wide constants
│   │   └── decorators.py                   # Custom decorators
│   └── middleware/
│       ├── __init__.py                     # Package init
│       ├── request_logging.py              # Request/response logging
│       └── cors.py                         # CORS configuration
│
├── requirements/
│   ├── base.txt                           # Core dependencies
│   ├── development.txt                    # + dev tools (black, pytest)
│   ├── staging.txt                        # Staging environment
│   └── production.txt                     # Production + monitoring
│
├── scripts/
│   ├── manage_admin.py                    # Create superuser script
│   └── deploy.py                          # Deployment automation
│
├── docker/
│   ├── Dockerfile                         # Container build instructions
│   ├── docker-compose.yml                 # Local dev environment
│   └── nginx.conf                         # Reverse proxy config
│
├── .github/workflows/
│   └── ci.yml                             # GitHub Actions CI/CD
│
├── manage.py                              # Django management CLI
├── .env.example                           # Environment variable template
├── .gitignore                             # Git ignore rules
└── README.md                              # Project documentation
```

#### Architecture Explanation

**config/** - Project configuration
- `settings/__init__.py` - Package initialization
- `settings/base.py` - Common settings (database, installed apps, middleware)
- `settings/development.py` - Debug mode, local database, dev tools
- `settings/staging.py` - Pre-production testing environment
- `settings/production.py` - Production settings (security, performance)
- `urls.py` - Root URL routing to all app APIs
- `schema.py` - OpenAPI schema configuration (drf-spectacular)
- `asgi.py` - ASGI server config for async/WebSockets
- `wsgi.py` - WSGI server config for traditional deployment

**apps/** - Application modules (domain-driven design)
- Each app is a bounded context (users, articles, etc.)
- Self-contained with its own API, models, services, tests

**api/** - API layer (per app) with versioning
- `v1/` - Current stable API version
  - `views/` - Split by feature (auth.py, profile.py, admin.py)
  - `serializers/` - Split by feature matching views
  - `urls.py` - Version-specific URL routing
  - `permissions.py` - Custom permission classes (IsOwner, IsAdmin)
- `v2/` - Future API version (breaking changes only)
  - Only include files that differ from v1
  - Inherit unchanged components from v1

**models.py** - Django models (SINGLE FILE per app)
- All models for the app in one file
- Keeps related models together
- Easier to understand relationships

**managers.py** - Custom QuerySet managers
- Custom query methods (e.g., `published()`, `active()`)
- Reusable query logic
- Chainable queryset methods

**services/** - Business logic layer (shared across versions)
- Write operations (create, update, delete)
- Complex business rules and validation
- External API calls
- File uploads, email sending
- Transaction management
- **Version-agnostic** - Reused by all API versions

**selectors/** - Query layer (shared across versions)
- Read operations (get, list, filter)
- Complex queries with joins (select_related, prefetch_related)
- Data aggregation and annotations
- Keeps views thin
- **Version-agnostic** - Reused by all API versions

**validators.py** - Custom validation logic
- Model-level validators
- Field-specific validation rules
- Reusable validation functions

**tasks.py** - Celery async tasks
- Background jobs (email sending, notifications)
- Scheduled tasks (cleanup, reports)
- Long-running operations
- Decouples slow operations from request/response

**signals.py** - Django signals
- post_save, post_delete hooks
- Automatic actions on model changes
- Decoupled event handling

**tests/** - Comprehensive test suite
- `test_models.py` - Model validation and methods
- `test_services.py` - Business logic tests
- `test_selectors.py` - Query optimization tests
- `test_views.py` - API endpoint integration tests
- `factories.py` - Test data generation (FactoryBoy)

**common/** - Shared utilities across all apps
- `api/` - Shared API components
  - `pagination.py` - Custom pagination classes
  - `exceptions.py` - Custom exception handlers
  - `response.py` - Standardized API responses
  - `filters.py` - django-filter configurations
- `models/` - Abstract base models
  - `timestamped.py` - created_at, updated_at fields
  - `soft_delete.py` - Soft delete functionality
- `utils/` - Helper functions
  - `validators.py` - Reusable validators
  - `helpers.py` - Utility functions
  - `constants.py` - App-wide constants
  - `decorators.py` - Custom decorators
- `middleware/` - Custom middleware
  - `request_logging.py` - Log all requests/responses
  - `cors.py` - CORS configuration

**requirements/** - Dependency management
- `base.txt` - Core dependencies (Django, DRF, PostgreSQL)
- `development.txt` - Dev tools (black, pytest, ipdb)
- `staging.txt` - Staging-specific packages
- `production.txt` - Production tools (gunicorn, sentry)

**scripts/** - Automation scripts
- `manage_admin.py` - Create superuser, manage permissions
- `deploy.py` - Deployment automation

**docker/** - Containerization
- `Dockerfile` - Container build instructions
- `docker-compose.yml` - Multi-container setup (app, db, redis)
- `nginx.conf` - Reverse proxy configuration

**.github/workflows/** - CI/CD
- `ci.yml` - Automated testing, linting, deployment

**Root files:**
- `manage.py` - Django management CLI
- `.env.example` - Environment variable template
- `.gitignore` - Git ignore rules
- `README.md` - Project documentation

**Benefits:**
- **Clear separation of concerns** - API, business logic, data access separated
- **Scalable** - Easy to add new apps and features
- **Testable** - Each layer tested independently with factories
- **Maintainable** - Easy to find and modify code
- **Team-friendly** - Multiple developers can work without conflicts
- **Service/Selector pattern** - Keeps views thin, business logic reusable
- **API versioning built-in** - Support multiple API versions simultaneously
- **DRY principle** - Services/selectors shared across versions
- **Production-ready** - Docker, CI/CD, environment configs included
- **Async support** - Celery tasks for background jobs
- **Type safety** - Managers and validators ensure data integrity

#### Service Layer Pattern

**services/user_service.py** - Write operations
```python
from django.db import transaction

class UserService:
    @staticmethod
    @transaction.atomic
    def create_user(*, email: str, password: str, **kwargs):
        """
        Create user with validation and side effects
        """
        user = User.objects.create_user(
            email=email,
            password=password,
            **kwargs
        )
        # Send welcome email
        send_welcome_email(user)
        # Create user profile
        Profile.objects.create(user=user)
        return user
    
    @staticmethod
    @transaction.atomic
    def update_profile(*, user, **data):
        """
        Update user profile with business logic
        """
        profile = user.profile
        for key, value in data.items():
            setattr(profile, key, value)
        profile.save()
        return profile
```

#### Selector Layer Pattern

**selectors/user_selector.py** - Read operations
```python
from django.db.models import Count, Q

class UserSelector:
    @staticmethod
    def get_active_users():
        """
        Get all active users with optimized query
        """
        return User.objects.filter(
            is_active=True
        ).select_related('profile').prefetch_related('groups')
    
    @staticmethod
    def get_user_with_stats(*, user_id: int):
        """
        Get user with article and comment counts
        """
        return User.objects.filter(
            id=user_id
        ).annotate(
            article_count=Count('articles'),
            comment_count=Count('comments')
        ).first()
```

#### Using Services and Selectors in Views

**api/v1/views/profile.py**
```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from ..serializers.profile import ProfileSerializer
from ....services.user_service import UserService
from ....selectors.user_selector import UserSelector

class ProfileViewSet(viewsets.ViewSet):
    def list(self, request):
        """Use selector for read operations"""
        users = UserSelector.get_active_users()
        serializer = ProfileSerializer(users, many=True)
        return Response(serializer.data)
    
    def retrieve(self, request, pk=None):
        """Use selector with stats"""
        user = UserSelector.get_user_with_stats(user_id=pk)
        serializer = ProfileSerializer(user)
        return Response(serializer.data)
    
    @action(detail=True, methods=['patch'])
    def update_profile(self, request, pk=None):
        """Use service for write operations"""
        user = self.get_object()
        profile = UserService.update_profile(
            user=user,
            **request.data
        )
        serializer = ProfileSerializer(profile)
        return Response(serializer.data)
```

#### API Versioning in Practice

**config/urls.py** - Root URL configuration
```python
from django.urls import path, include

urlpatterns = [
    # API v1
    path('api/v1/users/', include('apps.users.api.v1.urls')),
    path('api/v1/articles/', include('apps.articles.api.v1.urls')),
    
    # API v2 (future)
    # path('api/v2/users/', include('apps.users.api.v2.urls')),
    # path('api/v2/articles/', include('apps.articles.api.v2.urls')),
]
```

**apps/users/api/v1/urls.py** - Version-specific URLs
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import auth, profile, admin

router = DefaultRouter()
router.register('profiles', profile.ProfileViewSet, basename='profile')

urlpatterns = [
    path('', include(router.urls)),
    path('auth/login/', auth.LoginView.as_view(), name='login'),
    path('auth/register/', auth.RegisterView.as_view(), name='register'),
]
```

**When to create v2/** - Only for breaking changes
```python
# apps/users/api/v2/serializers/profile.py
# Only create if v2 has DIFFERENT serializer than v1

from rest_framework import serializers
from apps.users.models import User

class ProfileSerializer(serializers.ModelSerializer):
    # v2 changes: Added new field, removed old field
    full_name = serializers.CharField(source='get_full_name')  # NEW
    # Removed 'first_name' and 'last_name' (breaking change)
    
    class Meta:
        model = User
        fields = ['id', 'email', 'full_name', 'avatar']  # Different from v1
```

**Reusing unchanged components from v1**
```python
# apps/users/api/v2/views/profile.py
# If profile view logic is same, import from v1
from ..v1.views.profile import ProfileViewSet

# Or if only serializer changed:
from rest_framework import viewsets
from ..serializers.profile import ProfileSerializer  # v2 serializer
from ....services.user_service import UserService  # Same service
from ....selectors.user_selector import UserSelector  # Same selector

class ProfileViewSet(viewsets.ViewSet):
    # Same logic as v1, just different serializer
    def list(self, request):
        users = UserSelector.get_active_users()  # Same selector
        serializer = ProfileSerializer(users, many=True)  # v2 serializer
        return Response(serializer.data)
```

#### Why This Architecture?

**Traditional (Fat Views):**
```python
# All logic in view - hard to test, reuse, maintain
class ArticleViewSet(viewsets.ModelViewSet):
    def create(self, request):
        # 50+ lines of business logic here
        # Hard to test, hard to reuse
        pass
```

**Service/Selector Pattern (Thin Views):**
```python
# View is just coordination - easy to test, reuse
class ArticleViewSet(viewsets.ModelViewSet):
    def list(self, request):
        articles = ArticleSelector.get_published_articles()
        return Response(ArticleSerializer(articles, many=True).data)
    
    def create(self, request):
        article = ArticleService.create_article(**request.data)
        return Response(ArticleSerializer(article).data, status=201)
```

**Advantages:**
- Views are 5-10 lines (coordination only)
- Business logic is reusable (services)
- Queries are optimized (selectors)
- Easy to test each layer
- Easy to find code
- Prevents code duplication

### 8. Serializer Relations: HyperlinkedModelSerializer vs ModelSerializer

#### Approach Comparison

**Option 1: HyperlinkedModelSerializer (Recommended for Public APIs)**
```python
from rest_framework import serializers

class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']

# Response:
{
    "url": "http://api.example.com/users/1/",
    "username": "john",
    "email": "john@example.com",
    "groups": [
        "http://api.example.com/groups/1/",
        "http://api.example.com/groups/2/"
    ]
}
```

**Option 2: ModelSerializer (Simpler for Internal APIs)**
```python
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'groups']

# Response:
{
    "id": 1,
    "username": "john",
    "email": "john@example.com",
    "groups": [1, 2]
}
```

#### When to Use Each Approach

| Aspect | HyperlinkedModelSerializer | ModelSerializer |
|--------|---------------------------|-----------------|
| **Use When** | Building public/external APIs | Building internal/private APIs |
| **Best For** | RESTful, discoverable APIs | Simple, fast responses |
| **Client Type** | Web browsers, third-party apps | Mobile apps, SPAs with known structure |
| **Response Size** | Larger (full URLs) | Smaller (just IDs) |
| **Discoverability** | High (self-documenting) | Low (requires documentation) |
| **Client Complexity** | Lower (follow links) | Higher (construct URLs) |
| **Performance** | Slightly slower (URL generation) | Faster (simple IDs) |
| **Caching** | Better (full URLs cacheable) | Requires client-side URL construction |

#### Why HyperlinkedModelSerializer is Better for Public APIs

1. **HATEOAS Compliance:** Follows Hypermedia as the Engine of Application State principle
2. **Self-Documenting:** Clients can discover related resources without documentation
3. **Decoupling:** URL structure changes don't break clients
4. **Navigation:** Clients can follow links without constructing URLs
5. **RESTful:** Aligns with REST architectural principles

#### Why ModelSerializer is Better for Internal APIs

1. **Performance:** Smaller payload size, faster serialization
2. **Simplicity:** Easier to understand and implement
3. **Known Structure:** Internal clients already know URL patterns
4. **Bandwidth:** Reduces data transfer for mobile/low-bandwidth scenarios
5. **Caching:** Client can cache ID-to-URL mappings

#### Hybrid Approach (Best of Both Worlds)

```python
class UserSerializer(serializers.HyperlinkedModelSerializer):
    id = serializers.IntegerField(read_only=True)  # Include both
    
    class Meta:
        model = User
        fields = ['id', 'url', 'username', 'email', 'groups']

# Response includes both ID and URL:
{
    "id": 1,
    "url": "http://api.example.com/users/1/",
    "username": "john",
    "email": "john@example.com",
    "groups": [
        "http://api.example.com/groups/1/",
        "http://api.example.com/groups/2/"
    ]
}
```

**When to Use Hybrid:**
- APIs serving both web and mobile clients
- Transitioning from ID-based to hyperlinked
- Need flexibility for different client types

## HTTP Methods Usage

### GET - Retrieve Resources
- Should be idempotent and safe
- Must NOT alter state
- Use for fetching data only

**Bad Practice:**
```
GET /users/711/activate  # Altering state with GET
```

**Good Practice:**
```
POST /users/711/activation  # Use POST for state changes
```

### POST - Create Resources
- Creates new resources
- Returns 201 Created with Location header on success
- Returns 400 Bad Request for validation errors (never 422)
- Non-idempotent

> **📖 Reference:** See `09-Error-Handling-and-Status-Codes.md` for complete HTTP status code usage guide.

### PUT vs PATCH: When to Use Each

#### Detailed Comparison

**PUT - Full Resource Replacement**
```python
# PUT requires ALL fields
PUT /users/123
{
  "username": "john",
  "email": "john@example.com",
  "first_name": "John",
  "last_name": "Doe",
  "bio": "Software developer",
  "avatar": "http://example.com/avatar.jpg"
}
```

**PATCH - Partial Resource Update**
```python
# PATCH requires only changed fields
PATCH /users/123
{
  "email": "newemail@example.com"
}
```

#### When to Use PUT

| Scenario | Why PUT is Better |
|----------|-------------------|
| **Complete replacement** | Client has full resource representation |
| **Form submissions** | All fields are available from form |
| **Data synchronization** | Ensuring complete state match |
| **Versioning conflicts** | Preventing partial updates on stale data |
| **Simple clients** | Client doesn't need to track changes |

**PUT Implementation:**
```python
from rest_framework import viewsets

class UserViewSet(viewsets.ModelViewSet):
    def update(self, request, pk=None):
        """PUT - Requires all fields"""
        user = self.get_object()
        serializer = UserSerializer(user, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=400)
```

#### When to Use PATCH

| Scenario | Why PATCH is Better |
|----------|---------------------|
| **Single field updates** | Changing email, password, or status |
| **Mobile apps** | Reduces bandwidth usage |
| **Progressive forms** | Multi-step data collection |
| **Optimistic updates** | UI updates before server confirmation |
| **Large resources** | Avoid sending entire object |

**PATCH Implementation:**
```python
class UserViewSet(viewsets.ModelViewSet):
    def partial_update(self, request, pk=None):
        """PATCH - Accepts partial data"""
        user = self.get_object()
        serializer = UserSerializer(user, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=400)
```

#### Why PATCH is Generally Preferred

1. **Bandwidth Efficiency:** Only send changed fields
2. **Flexibility:** Clients don't need complete resource state
3. **User Experience:** Faster updates, better for mobile
4. **Concurrent Updates:** Less likely to overwrite other users' changes
5. **API Evolution:** New fields don't break existing clients

#### Why PUT Might Be Better

1. **Simplicity:** Clear "replace everything" semantics
2. **Consistency:** Guaranteed complete state after update
3. **Validation:** Easier to validate complete resource
4. **Conflict Prevention:** Ensures no partial/inconsistent states
5. **Idempotency:** Repeated calls have same effect

#### Best Practice: Support Both

```python
class UserViewSet(viewsets.ModelViewSet):
    http_method_names = ['get', 'post', 'put', 'patch', 'delete']
    
    def update(self, request, pk=None):
        """PUT - Full update"""
        return super().update(request, pk)
    
    def partial_update(self, request, pk=None):
        """PATCH - Partial update"""
        return super().partial_update(request, pk)
```

**Decision Matrix:**

```
Use PUT when:
✓ Client has complete resource representation
✓ Need to ensure complete state consistency
✓ Replacing entire resource is the intent
✓ Simpler client implementation

Use PATCH when:
✓ Updating one or few fields
✓ Optimizing for bandwidth/performance
✓ Building mobile or progressive web apps
✓ Supporting partial/incremental updates
✓ Default choice for most update operations
```

### DELETE - Remove Resources
- Deletes resources
- Idempotent
- Returns 204 No Content on success

## API Inventory and Documentation

### Maintain API Inventory
Keep a comprehensive inventory of all API hosts and endpoints.

**Document:**
- API version and environment (production, staging, development)
- Network access requirements (public, internal, partners)
- Authentication methods
- Rate limits and SLAs
- All endpoints with parameters, requests, and responses

### Create API Roadmap
- Track what's coming
- Document what's being deprecated
- Communicate changes like product releases
- Maintain changelog

## Design for Scalability

### Performance Targets
- Set latency targets (e.g., 95% of calls < 300ms)
- Monitor and track performance metrics
- Treat performance as ongoing maintenance

### Build Performance-First
- Implement pagination by default
- Fix N+1 query problems
- Cache heavy read operations
- Use database indexing strategically

## Summary

Key takeaways for API design and architecture:

1. **Consistency:** Use plural nouns, spinal-case, and consistent patterns
2. **Simplicity:** Avoid deep nesting, use query parameters for filtering
3. **RESTful:** Use proper HTTP methods, hyperlinked relations
4. **Scalability:** Design with performance in mind from the start
5. **Documentation:** Maintain comprehensive API inventory and roadmap
6. **Product Mindset:** Treat APIs as strategic products, not just technical implementations

Following these best practices ensures your Django REST Framework APIs are maintainable, scalable, and developer-friendly.
