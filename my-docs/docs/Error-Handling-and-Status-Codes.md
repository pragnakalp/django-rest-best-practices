# Production-Grade Error Handling and Status Codes

## Overview
Proper error handling and status code usage are essential for building reliable, audit-safe APIs. This document covers production-ready patterns for handling errors and using HTTP status codes correctly in Django REST Framework.

## Core Principles

### 1. Single Error Response Format
All API errors must follow one canonical format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "fields": null
  }
}
```

### 2. Thin Views, Rich Services
Views should never contain business try/except blocks. Move business logic to service layers.

### 3. Machine-Readable Error Codes
Every error must have a stable, machine-readable code that never changes.

### 4. Safe Logging Practices
Log identifiers, never payloads. Protect sensitive data.

## HTTP Status Codes

### Understanding Status Code Categories

- **1xx (Informational):** Request received, continuing process
- **2xx (Success):** Request successfully received, understood, and accepted
- **3xx (Redirection):** Further action needed to complete request
- **4xx (Client Error):** Request contains bad syntax or cannot be fulfilled
- **5xx (Server Error):** Server failed to fulfill valid request

### Status Code Strategy

**Use 400 for all validation errors.** Don't mix 400 and 422. Choose one and enforce it globally.

## Production-Grade Error Handling

### 1. Domain Exceptions (Business Logic Errors)

Define domain-specific exceptions that carry error codes and status codes.

```python
# core/exceptions.py
class DomainError(Exception):
    """Base class for all business logic errors"""
    code = "DOMAIN_ERROR"
    status_code = 400
    
    def __init__(self, message=None):
        self.message = message or self.__class__.__name__.replace('_', ' ').title()
        super().__init__(self.message)

class ArticleAlreadyPublished(DomainError):
    code = "ARTICLE_ALREADY_PUBLISHED"
    status_code = 409

class ArticleNotFound(DomainError):
    code = "ARTICLE_NOT_FOUND"
    status_code = 404

class InvalidArticleStatus(DomainError):
    code = "INVALID_ARTICLE_STATUS"
    status_code = 400

class PermissionDenied(DomainError):
    code = "PERMISSION_DENIED"
    status_code = 403
```

### 2. Service Layer (No Try/Except in Views)

Move all business logic to service layers. Views should be thin and exception-free.

```python
# articles/services.py
from core.exceptions import ArticleAlreadyPublished, ArticleNotFound, InvalidArticleStatus

def publish_article(article, user):
    """Publish an article with business rules"""
    if article.author != user:
        raise PermissionDenied("You can only publish your own articles")
    
    if article.published:
        raise ArticleAlreadyPublished("Article is already published")
    
    if article.status not in ['draft', 'review']:
        raise InvalidArticleStatus(f"Cannot publish article with status: {article.status}")
    
    article.published = True
    article.status = 'published'
    article.published_at = timezone.now()
    article.save()
    
    return article

def get_article_for_user(article_id, user):
    """Get article with permission check"""
    try:
        article = Article.objects.get(id=article_id)
    except Article.DoesNotExist:
        raise ArticleNotFound("Article not found")
    
    if article.author != user and not article.published:
        raise PermissionDenied("You don't have permission to view this article")
    
    return article
```

### 3. Thin Views (No Business Try/Except)

Views should orchestrate, not handle business logic.

```python
# articles/api/views.py
from articles.services import publish_article, get_article_for_user

class ArticleViewSet(viewsets.ModelViewSet):
    """Thin viewset - no business try/except"""
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        article = self.get_object()
        published_article = publish_article(article, request.user)
        return Response(self.get_serializer(published_article).data)
    
    def retrieve(self, request, pk=None):
        article = get_article_for_user(pk, request.user)
        return Response(self.get_serializer(article).data)
```

### 4. Global Exception Handler (Single Error Format)

One canonical error response format for all errors.

```python
# core/exception_handler.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status
from core.exceptions import DomainError

def api_exception_handler(exc, context):
    """Global exception handler with single error format"""
    
    # Domain errors (business logic)
    if isinstance(exc, DomainError):
        return Response(
            {
                "error": {
                    "code": exc.code,
                    "message": exc.message,
                    "fields": None
                }
            },
            status=exc.status_code
        )
    
    # DRF validation errors
    response = exception_handler(exc, context)
    if response is not None:
        return Response(
            {
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "Invalid input",
                    "fields": response.data
                }
            },
            status=response.status_code
        )
    
    # Unhandled exceptions (500)
    return Response(
        {
            "error": {
                "code": "INTERNAL_SERVER_ERROR",
                "message": "Something went wrong"
            }
        },
        status=status.HTTP_500_INTERNAL_SERVER_ERROR
    )
```

```python
# settings.py
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "core.exception_handler.api_exception_handler"
}
```

### 5. Standard Error Codes

Use consistent, machine-readable error codes across your API.

```python
# core/error_codes.py
class ErrorCodes:
    # Authentication & Authorization
    AUTH_REQUIRED = "AUTH_REQUIRED"
    PERMISSION_DENIED = "PERMISSION_DENIED"
    INVALID_TOKEN = "INVALID_TOKEN"
    TOKEN_EXPIRED = "TOKEN_EXPIRED"
    
    # Validation
    VALIDATION_ERROR = "VALIDATION_ERROR"
    INVALID_INPUT = "INVALID_INPUT"
    MISSING_FIELD = "MISSING_FIELD"
    
    # Resources
    RESOURCE_NOT_FOUND = "RESOURCE_NOT_FOUND"
    ARTICLE_NOT_FOUND = "ARTICLE_NOT_FOUND"
    USER_NOT_FOUND = "USER_NOT_FOUND"
    
    # Business Logic
    ARTICLE_ALREADY_PUBLISHED = "ARTICLE_ALREADY_PUBLISHED"
    INVALID_ARTICLE_STATUS = "INVALID_ARTICLE_STATUS"
    DUPLICATE_RESOURCE = "DUPLICATE_RESOURCE"
    
    # Rate Limiting
    RATE_LIMITED = "RATE_LIMITED"
    TOO_MANY_REQUESTS = "TOO_MANY_REQUESTS"
    
    # Server Errors
    INTERNAL_SERVER_ERROR = "INTERNAL_SERVER_ERROR"
    SERVICE_UNAVAILABLE = "SERVICE_UNAVAILABLE"
```

### 6. Safe Logging Practices

Log identifiers, never payloads. Protect sensitive data.

```python
# core/logging.py
import logging
from django.conf import settings

logger = logging.getLogger(__name__)

def log_error(message, **kwargs):
    """Safe logging with identifiers only"""
    safe_kwargs = {k: v for k, v in kwargs.items() if k not in ['password', 'token', 'secret']}
    logger.error(message, extra=safe_kwargs)

def log_business_error(exc, request=None, **context):
    """Log business errors safely"""
    log_error(
        f"Business error: {exc.code}",
        error_code=exc.code,
        error_message=exc.message,
        user_id=request.user.id if request and hasattr(request, 'user') else None,
        **context
    )

# Usage in services
def publish_article(article, user):
    try:
        # ... business logic
        pass
    except DomainError as exc:
        log_business_error(exc, request=user, article_id=article.id)
        raise
```

### 7. Error Response Examples

All errors follow the same format:

```json
// Validation Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "fields": {
      "title": ["Title must be at least 5 characters long"],
      "email": ["Enter a valid email address"]
    }
  }
}

// Business Logic Error
{
  "error": {
    "code": "ARTICLE_ALREADY_PUBLISHED",
    "message": "Article is already published",
    "fields": null
  }
}

// Permission Error
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "You do not have permission to perform this action",
    "fields": null
  }
}

// Not Found Error
{
  "error": {
    "code": "ARTICLE_NOT_FOUND",
    "message": "Article not found",
    "fields": null
  }
}

// Server Error
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "Something went wrong",
    "fields": null
  }
}
```

## Common Status Codes

### 2xx Success Codes

#### 200 OK
General success response for GET, PUT, PATCH, or DELETE.

```python
from rest_framework import status
from rest_framework.response import Response

class ArticleViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        article = self.get_object()
        serializer = self.get_serializer(article)
        return Response(serializer.data, status=status.HTTP_200_OK)
```

#### 201 Created
Resource successfully created (POST requests).

```python
def create(self, request, *args, **kwargs):
    serializer = self.get_serializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    instance = serializer.save()
    
    headers = {
        'Location': reverse('article-detail', args=[instance.id], request=request)
    }
    
    return Response(
        serializer.data,
        status=status.HTTP_201_CREATED,
        headers=headers
    )
```

#### 202 Accepted
Request accepted for processing but not completed (async operations).

```python
from celery import shared_task

@shared_task
def generate_report(article_id):
    # Long-running task
    pass

class ArticleViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def generate_report(self, request, pk=None):
        article = self.get_object()
        
        # Queue async task
        task = generate_report.delay(article.id)
        
        return Response(
            {
                'task_id': task.id,
                'status': 'PENDING',
                'status_url': f'/api/tasks/{task.id}/'
            },
            status=status.HTTP_202_ACCEPTED
        )
```

**Use Cases:**
- Async processing
- Batch operations
- Long-running tasks

#### 204 No Content
Successful request with no response body (typically DELETE).

```python
def destroy(self, request, pk=None):
    article = self.get_object()
    article.delete()
    return Response(status=status.HTTP_204_NO_CONTENT)
```

### 3xx Redirection Codes

#### 301 Moved Permanently
Resource permanently moved to new URL.

```python
from django.http import HttpResponsePermanentRedirect

def old_endpoint(request):
    return HttpResponsePermanentRedirect('/api/v2/articles/')
```

#### 302 Found / 307 Temporary Redirect
Resource temporarily at different URL.

```python
from rest_framework.response import Response

def temporary_endpoint(request):
    response = Response(status=status.HTTP_307_TEMPORARY_REDIRECT)
    response['Location'] = '/api/v2/articles/'
    return response
```

### 4xx Client Error Codes

#### 400 Bad Request
Invalid request syntax or validation errors. Use 400 for ALL validation errors.

```python
# Serializer validation (automatically handled by global exception handler)
class ArticleSerializer(serializers.ModelSerializer):
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError(
                "Title must be at least 5 characters long"
            )
        return value

# Response (formatted by global exception handler):
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "fields": {
      "title": ["Title must be at least 5 characters long"]
    }
  }
}
```

#### 401 Unauthorized
Authentication required or failed.

```python
from rest_framework.permissions import IsAuthenticated

class ArticleViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    
    # Missing/invalid token returns (formatted by global exception handler):
    {
      "error": {
        "code": "AUTH_REQUIRED",
        "message": "Authentication credentials were not provided.",
        "fields": null
      }
    }
```

**401 vs 403:**
- **401:** User not authenticated (missing or invalid credentials)
- **403:** User authenticated but lacks permission

#### 403 Forbidden
Authenticated but not authorized.

```python
from rest_framework.permissions import BasePermission

class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user

# Non-owner trying to edit returns (formatted by global exception handler):
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "You do not have permission to perform this action",
    "fields": null
  }
}
```

#### 404 Not Found
Resource not found.

```python
# Handled by service layer (raises ArticleNotFound)
def get_article_for_user(article_id, user):
    try:
        article = Article.objects.get(id=article_id)
    except Article.DoesNotExist:
        raise ArticleNotFound("Article not found")
    
    return article

# Response (formatted by global exception handler):
{
  "error": {
    "code": "ARTICLE_NOT_FOUND",
    "message": "Article not found",
    "fields": null
  }
}
```

#### 409 Conflict
Request conflicts with current state of resource.

```python
# Business logic error (handled by service layer)
def publish_article(article, user):
    if article.published:
        raise ArticleAlreadyPublished("Article is already published")
    
    # ... publish logic
    return article

# Response (formatted by global exception handler):
{
  "error": {
    "code": "ARTICLE_ALREADY_PUBLISHED",
    "message": "Article is already published",
    "fields": null
  }
}
```

#### 429 Too Many Requests
Rate limit exceeded.

```python
from rest_framework.throttling import UserRateThrottle

class ArticleViewSet(viewsets.ModelViewSet):
    throttle_classes = [UserRateThrottle]
    
    # Rate limit exceeded returns (formatted by global exception handler):
    {
      "error": {
        "code": "RATE_LIMITED",
        "message": "Request was throttled.",
        "fields": null
      }
    }
```

#### 405 Method Not Allowed
HTTP method not supported for endpoint.

```python
class ArticleViewSet(viewsets.ModelViewSet):
    http_method_names = ['get', 'post', 'patch', 'delete']  # No PUT
    
    # PUT request returns (formatted by global exception handler):
    {
      "error": {
        "code": "METHOD_NOT_ALLOWED",
        "message": "Method \"PUT\" not allowed.",
        "fields": null
      }
    }
```

### 5xx Server Error Codes

#### 500 Internal Server Error
Unexpected server error. Never expose stack traces in production.

```python
# Handled by global exception handler
# All unhandled exceptions return:
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "Something went wrong",
    "fields": null
  }
}
```

**Important:** Never expose detailed error information in production responses. Use logging for debugging.

#### 503 Service Unavailable
Service temporarily unavailable.

```python
from django.core.management.base import BaseCommand
from rest_framework.response import Response
from rest_framework import status

class MaintenanceModeMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        if settings.MAINTENANCE_MODE:
            return Response(
                {
                    "error": {
                        "code": "SERVICE_UNAVAILABLE",
                        "message": "Service temporarily unavailable",
                        "fields": null
                    }
                },
                status=status.HTTP_503_SERVICE_UNAVAILABLE
            )
        return self.get_response(request)
```
## Testing Error Handling

### Test Domain Exceptions

```python
from rest_framework.test import APITestCase
from core.exceptions import ArticleAlreadyPublished, ArticleNotFound
from articles.services import publish_article

class ErrorHandlingTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.article = ArticleFactory(author=self.user, published=True)
    
    def test_publish_already_published_article(self):
        """Test business logic error handling"""
        with self.assertRaises(ArticleAlreadyPublished) as context:
            publish_article(self.article, self.user)
        
        self.assertEqual(context.exception.code, "ARTICLE_ALREADY_PUBLISHED")
        self.assertEqual(context.exception.status_code, 409)
    
    def test_publish_unauthorized_article(self):
        """Test permission error handling"""
        other_user = UserFactory()
        other_article = ArticleFactory(author=other_user)
        
        with self.assertRaises(PermissionDenied) as context:
            publish_article(other_article, self.user)
        
        self.assertEqual(context.exception.code, "PERMISSION_DENIED")
        self.assertEqual(context.exception.status_code, 403)
    
    def test_api_error_response_format(self):
        """Test API returns consistent error format"""
        self.client.force_authenticate(user=self.user)
        response = self.client.post(f'/api/articles/{self.article.id}/publish/')
        
        self.assertEqual(response.status_code, 409)
        self.assertIn('error', response.data)
        self.assertEqual(response.data['error']['code'], 'ARTICLE_ALREADY_PUBLISHED')
        self.assertEqual(response.data['error']['fields'], None)
```

### Test Validation Errors

```python
class ValidationTestCase(APITestCase):
    def test_validation_error_format(self):
        """Test validation errors return correct format"""
        self.client.force_authenticate(user=UserFactory())
        
        response = self.client.post('/api/articles/', {
            'title': 'short',  # Too short
            'content': ''      # Required field missing
        })
        
        self.assertEqual(response.status_code, 400)
        self.assertIn('error', response.data)
        self.assertEqual(response.data['error']['code'], 'VALIDATION_ERROR')
        self.assertIn('fields', response.data['error'])
        self.assertIn('title', response.data['error']['fields'])
        self.assertIn('content', response.data['error']['fields'])
```

## Production Deployment Checklist

### 1. Exception Handler Configuration
```python
# settings.py - Production
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "core.exception_handler.api_exception_handler",
    # ... other settings
}

# Ensure DEBUG = False in production
DEBUG = False
```

### 2. Logging Configuration
```python
# settings.py - Production logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            'class': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s'
        }
    },
    'handlers': {
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/api.log',
            'maxBytes': 1024*1024*15,  # 15MB
            'backupCount': 10,
            'formatter': 'json'
        },
        'sentry': {
            'level': 'ERROR',
            'class': 'sentry_sdk.integrations.django.SentryHandler',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file', 'sentry'],
            'level': 'INFO',
            'propagate': False,
        },
        'core': {
            'handlers': ['file', 'sentry'],
            'level': 'INFO',
            'propagate': False,
        },
    }
}
```

### 3. Monitoring Integration
```python
# core/monitoring.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=settings.SENTRY_DSN,
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,
    send_default_pii=False  # Don't send PII
)

# Custom error context
def add_error_context(exc, request):
    if hasattr(request, 'user') and request.user.is_authenticated:
        sentry_sdk.set_user({
            "id": request.user.id,
            "email": request.user.email
        })
    
    sentry_sdk.set_tag("error_code", getattr(exc, 'code', 'UNKNOWN'))
    sentry_sdk.set_context("request", {
        "path": request.path,
        "method": request.method,
        "user_agent": request.META.get('HTTP_USER_AGENT')
    })
```

## Summary

### Production-Grade Error Handling Principles

1. **Single Error Format**: All errors follow the same structure with code, message, and fields
2. **Thin Views**: No business try/except in views - move to service layers
3. **Domain Exceptions**: Business logic errors raise specific exceptions with codes
4. **Global Handler**: One place handles all error formatting
5. **Safe Logging**: Log identifiers, never sensitive data
6. **Consistent Status Codes**: Use 400 for all validation errors
7. **Machine-Readable Codes**: Stable error codes for frontend/mobile clients
8. **Production Safety**: Never expose stack traces or internal details

### Key Benefits

- **Frontend-Friendly**: Consistent error format across all endpoints
- **Mobile-Ready**: Machine-readable codes for mobile apps
- **Audit-Safe**: Proper logging and monitoring
- **Scalable**: Clean separation of concerns
- **Maintainable**: Centralized error handling logic

Following these patterns ensures your API is production-ready, secure, and provides excellent developer experience for API consumers.
    
                
