# Security Best Practices

## Overview
This document consolidates security best practices for Django REST Framework based on OWASP API Security Top 10 and industry standards.

## OWASP API Security Top 10 (2019)

### API1:2019 Broken Object Level Authorization

#### The Problem
APIs often expose endpoints that handle object identifiers, creating a wide attack surface. Attackers can manipulate IDs to access data belonging to other users.

#### Best Practices

**Always Check Object Permissions:**
```python
from rest_framework import viewsets
from django.shortcuts import get_object_or_404

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def get_object(self):
        """
        Override to ensure object permissions are checked.
        """
        obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
        # CRITICAL: Always check object permissions
        self.check_object_permissions(self.request, obj)
        return obj
```

**Never Do This:**
```python
# BAD - No permission check
def get_object(self):
    return Article.objects.get(pk=self.kwargs["pk"])
```

**Filter Queryset by User:**
```python
def get_queryset(self):
    """
    Users can only access their own articles.
    """
    return Article.objects.filter(author=self.request.user)
```

### API2:2019 Broken User Authentication

#### The Problem
Poorly implemented authentication mechanisms allow attackers to compromise tokens or exploit implementation flaws.

#### Best Practices

**Configure Authentication Properly:**
```python
# settings.py - JWT is recommended for modern applications
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}
```

**Alternative for Traditional Web Apps:**
```python
# For applications requiring both web and mobile support
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**Never Override Without Understanding:**
```python
# BAD - Disabling authentication without understanding
class UnsafeView(APIView):
    authentication_classes = []  # Dangerous!
```

**Good - Explicit Authentication:**
```python
class SecureView(APIView):
    authentication_classes = [JWTAuthentication]
    permission_classes = [IsAuthenticated]
```

**Implement JWT Token Expiration:**
```python
# JWT handles expiration automatically, but configure properly:
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),  # Short-lived access tokens
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),     # Longer refresh tokens
    'ROTATE_REFRESH_TOKENS': True,                   # Security best practice
    'BLACKLIST_AFTER_ROTATION': True,                # Invalidate old refresh tokens
}
```

**Legacy Token Expiration (if using Token Authentication):**
```python
from datetime import timedelta
from django.utils import timezone

def is_token_expired(token):
    """Check if token is older than 24 hours"""
    time_elapsed = timezone.now() - token.created
    return time_elapsed > timedelta(hours=24)
```

**Rate Limit Authentication Endpoints:**
```python
from rest_framework.throttling import AnonRateThrottle

class LoginRateThrottle(AnonRateThrottle):
    rate = '5/hour'

class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]
```

### API3:2019 Excessive Data Exposure

#### The Problem
APIs return more data than necessary, relying on clients to filter. Attackers can access sensitive data not intended for them.

#### Best Practices

**Use Explicit Field Lists:**
```python
# BAD - Exposes all fields
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # NEVER DO THIS

# BAD - Denylist approach
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        exclude = ['password']  # Still risky

# GOOD - Allowlist approach
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name']
```

**Different Serializers for Different Contexts:**
```python
class UserListSerializer(serializers.ModelSerializer):
    """Minimal data for list view"""
    class Meta:
        model = User
        fields = ['id', 'username']

class UserDetailSerializer(serializers.ModelSerializer):
    """More data for detail view"""
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name', 'date_joined']

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    
    def get_serializer_class(self):
        if self.action == 'list':
            return UserListSerializer
        return UserDetailSerializer
```

**Hide Sensitive Fields:**
```python
class UserSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)  # Never return password
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'password']
```

### API4:2019 Lack of Resources & Rate Limiting

#### The Problem
Without rate limiting, APIs are vulnerable to DoS attacks and resource exhaustion.

#### Comparing Rate Limiting Approaches

**Approach 1: DRF Built-in Throttling (Simple)**

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

**Pros:**
- Built into DRF
- Easy to configure
- Per-user and anonymous limits
- No external dependencies

**Cons:**
- Stored in cache (memory/Redis)
- Not distributed by default
- Limited customization
- Can be bypassed with IP rotation

**Use When:**
- Small to medium applications
- Simple rate limiting needs
- Already using Django cache
- Single server deployment

**Approach 2: Django-ratelimit (More Flexible)**

```python
# pip install django-ratelimit
from django_ratelimit.decorators import ratelimit

@ratelimit(key='ip', rate='5/m', method='POST')
@api_view(['POST'])
def login_view(request):
    # Login logic
    pass
```

**Pros:**
- More flexible rate keys
- IP-based limiting
- Method-specific rates
- Good for function-based views

**Cons:**
- Separate from DRF
- Different syntax
- Less integration

**Use When:**
- Need IP-based limiting
- Function-based views
- More granular control
- Specific endpoint protection

**Approach 3: WAF/API Gateway (Production)**

```python
# Cloudflare, AWS WAF, Kong, etc.
# Configure at infrastructure level
```

**Pros:**
- Best performance
- Distributed by default
- Advanced features (geo-blocking, bot detection)
- Protects before hitting application
- DDoS protection

**Cons:**
- Additional cost
- External dependency
- More complex setup
- Requires infrastructure access

**Use When:**
- Production environments
- High-traffic applications
- Need DDoS protection
- Multi-server deployment
- Budget allows

**Approach 4: Redis-based Custom (Scalable)**

```python
import redis
from rest_framework.throttling import SimpleRateThrottle

class RedisRateThrottle(SimpleRateThrottle):
    cache = redis.Redis(host='localhost', port=6379, db=0)
    
    def get_cache_key(self, request, view):
        return f"throttle:{request.user.id}:{view.__class__.__name__}"
```

**Pros:**
- Distributed across servers
- Fast performance
- Scalable
- Custom logic possible

**Cons:**
- Requires Redis
- More code to maintain
- Complexity

**Use When:**
- Multi-server deployment
- Need distributed rate limiting
- High scalability required
- Already using Redis

#### Decision Matrix for Rate Limiting

| Scenario | Recommended Approach | Why |
|----------|---------------------|-----|
| Single server | DRF built-in | Simple, sufficient |
| Multi-server | Redis-based or WAF | Distributed state |
| High traffic | WAF/API Gateway | Best performance |
| Development | DRF built-in | Easy setup |
| Production SaaS | WAF + DRF | Layered protection |
| Budget constrained | DRF built-in | No extra cost |
| Enterprise | WAF/API Gateway | Advanced features |

#### Layered Rate Limiting Strategy (Best Practice)

```python
# Layer 1: WAF/API Gateway (infrastructure)
# - 10,000 requests/minute per IP
# - DDoS protection
# - Bot detection

# Layer 2: DRF Throttling (application)
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
        'login': '5/hour',  # Stricter for sensitive endpoints
    }
}

# Layer 3: Endpoint-specific (critical operations)
class ExpensiveOperationView(APIView):
    throttle_classes = [UserRateThrottle]
    throttle_scope = 'expensive'
```

**Why Layered Approach:**
- WAF blocks obvious attacks
- DRF handles legitimate users
- Endpoint-specific protects critical operations
- Defense in depth

**Custom Throttle Classes:**
```python
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    rate = '60/min'

class SustainedRateThrottle(UserRateThrottle):
    rate = '1000/day'

class ArticleViewSet(viewsets.ModelViewSet):
    throttle_classes = [BurstRateThrottle, SustainedRateThrottle]
```

**Per-View Throttling:**
```python
class ExpensiveOperationView(APIView):
    throttle_classes = [UserRateThrottle]
    throttle_scope = 'expensive_operation'

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'expensive_operation': '10/hour',
    }
}
```

**Implement Pagination:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20
}
```

**Important:** Use WAF or similar for primary rate limiting. DRF throttling should be the last layer.

### API5:2019 Broken Function Level Authorization

#### The Problem
APIs don't properly enforce authorization checks on functions, allowing users to access admin or privileged functions.

#### Best Practices

**Change Default Permission:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',  # Not AllowAny!
    ],
}
```

**Never Use AllowAny Globally:**
```python
# BAD
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',  # Dangerous default
    ],
}
```

**Explicit Permissions Per View:**
```python
from rest_framework.permissions import IsAuthenticated, IsAdminUser

class PublicArticleList(APIView):
    permission_classes = [AllowAny]  # Explicitly public

class AdminOnlyView(APIView):
    permission_classes = [IsAdminUser]  # Admin only

class UserArticleView(APIView):
    permission_classes = [IsAuthenticated]  # Authenticated users
```

**Function-Level Permissions:**
```python
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def get_permissions(self):
        if self.action in ['list', 'retrieve']:
            permission_classes = [AllowAny]
        elif self.action == 'create':
            permission_classes = [IsAuthenticated]
        elif self.action in ['update', 'partial_update', 'destroy']:
            permission_classes = [IsAuthenticated, IsOwnerOrAdmin]
        else:
            permission_classes = [IsAdminUser]
        
        return [permission() for permission in permission_classes]
```

### API6:2019 Mass Assignment

#### The Problem
Binding client-provided data to data models without proper filtering allows attackers to modify object properties they shouldn't.

#### Best Practices

**Use Explicit Fields (Allowlist):**
```python
# GOOD - Explicit field list
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['username', 'email', 'first_name', 'last_name']

# BAD - Excludes fields (denylist)
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        exclude = ['is_staff', 'is_superuser']  # Risky

# NEVER DO THIS
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = '__all__'  # Exposes everything
```

**Read-Only Fields:**
```python
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'is_staff', 'date_joined']
        read_only_fields = ['id', 'is_staff', 'date_joined']
```

**Separate Serializers for Read/Write:**
```python
class UserReadSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'is_staff', 'date_joined']

class UserWriteSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['username', 'email', 'first_name', 'last_name']

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    
    def get_serializer_class(self):
        if self.action in ['create', 'update', 'partial_update']:
            return UserWriteSerializer
        return UserReadSerializer
```

### API7:2019 Security Misconfiguration

#### The Problem
Insecure default configurations, incomplete configurations, or verbose error messages expose the system to attacks.

#### Best Practices

**Production Settings:**
```python
# settings.py - Production
DEBUG = False
DEBUG_PROPAGATE_EXCEPTIONS = False

ALLOWED_HOSTS = ['yourdomain.com', 'api.yourdomain.com']

# Generate strong secret key
SECRET_KEY = os.environ.get('SECRET_KEY')  # Never hardcode!

# HTTPS settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# HSTS
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

**Disable Browsable API in Production:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
}

# Or conditionally
if not DEBUG:
    REST_FRAMEWORK['DEFAULT_RENDERER_CLASSES'] = [
        'rest_framework.renderers.JSONRenderer',
    ]
```

**Restrict HTTP Methods:**
```python
class ArticleViewSet(viewsets.ModelViewSet):
    http_method_names = ['get', 'post', 'patch', 'delete']  # No PUT
```

**Custom Error Responses:**
```python
# Don't expose internal details
def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None and not settings.DEBUG:
        # Hide detailed error messages in production
        response.data = {
            'error': 'An error occurred',
            'code': response.status_code
        }
    
    return response

# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'myapp.utils.custom_exception_handler'
}
```

**Validate and Sanitize Input:**
```python
class ArticleSerializer(serializers.ModelSerializer):
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    
    def validate_title(self, value):
        # Sanitize input
        import bleach
        return bleach.clean(value)
    
    def validate_content(self, value):
        # Sanitize HTML content
        import bleach
        allowed_tags = ['p', 'br', 'strong', 'em', 'a']
        return bleach.clean(value, tags=allowed_tags)
```

### API8:2019 Injection

#### The Problem
Untrusted data sent to an interpreter as part of a command or query can lead to injection attacks.

#### SQL Injection Prevention

**Use ORM (Parameterized Queries):**
```python
# GOOD - ORM automatically parameterizes
articles = Article.objects.filter(author__username=username)

# GOOD - Parameterized raw query
from django.db import connection
cursor = connection.cursor()
cursor.execute("SELECT * FROM articles WHERE author_id = %s", [author_id])

# BAD - String concatenation
cursor.execute(f"SELECT * FROM articles WHERE author_id = {author_id}")  # NEVER!
```

**Avoid Dangerous Methods:**
```python
# DANGEROUS - Avoid these with user input
Article.objects.raw(f"SELECT * FROM articles WHERE id = {user_input}")  # NO!
Article.objects.extra(where=[f"id = {user_input}"])  # NO!

# GOOD - Use parameterization
Article.objects.raw("SELECT * FROM articles WHERE id = %s", [user_input])
```

#### Command Injection Prevention

**Avoid eval(), exec(), execfile():**
```python
# NEVER DO THIS
user_code = request.data.get('code')
eval(user_code)  # EXTREMELY DANGEROUS
exec(user_code)  # EXTREMELY DANGEROUS
```

#### YAML/Pickle Injection Prevention

**Use Safe Loaders:**
```python
import yaml

# BAD
data = yaml.load(user_file)  # Unsafe

# GOOD
data = yaml.load(user_file, Loader=yaml.SafeLoader)  # Safe
```

**Avoid Pickle with User Input:**
```python
import pickle

# NEVER load user-controlled pickle files
data = pickle.load(user_file)  # DANGEROUS

# Also avoid pandas.read_pickle() with user files
```

### API9:2019 Improper Assets Management

#### The Problem
Outdated documentation, unnecessary endpoints, and old API versions create attack surfaces.

#### Best Practices

**Maintain API Inventory:**
```python
# Document all endpoints
API_INVENTORY = {
    'v1': {
        'endpoints': ['/api/v1/users/', '/api/v1/articles/'],
        'status': 'deprecated',
        'sunset_date': '2024-12-31',
        'environment': 'production'
    },
    'v2': {
        'endpoints': ['/api/v2/users/', '/api/v2/articles/'],
        'status': 'active',
        'environment': 'production'
    }
}
```

**Version Your API:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v2',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# urls.py
urlpatterns = [
    path('api/v1/', include('myapp.api.v1.urls')),
    path('api/v2/', include('myapp.api.v2.urls')),
]
```

**Deprecation Headers:**
```python
from rest_framework.response import Response

class DeprecatedView(APIView):
    def get(self, request):
        response = Response({'data': 'some data'})
        response['Warning'] = '299 - "Deprecated API. Use /api/v2/ instead"'
        response['Sunset'] = 'Sat, 31 Dec 2024 23:59:59 GMT'
        return response
```

**Document Everything:**
- API version and environment
- Authentication methods
- Rate limits and SLAs
- All endpoints with parameters
- Deprecation timelines

### API10:2019 Insufficient Logging & Monitoring

#### The Problem
Lack of logging and monitoring allows attacks to go undetected.

#### Best Practices

**Comprehensive Logging:**
```python
import logging

logger = logging.getLogger(__name__)

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def create(self, request, *args, **kwargs):
        logger.info(
            f"Article creation attempt by user {request.user.id}",
            extra={
                'user_id': request.user.id,
                'ip_address': request.META.get('REMOTE_ADDR'),
                'user_agent': request.META.get('HTTP_USER_AGENT')
            }
        )
        
        try:
            response = super().create(request, *args, **kwargs)
            logger.info(f"Article {response.data['id']} created successfully")
            return response
        except Exception as e:
            logger.error(
                f"Article creation failed: {str(e)}",
                extra={
                    'user_id': request.user.id,
                    'error': str(e),
                    'stack_trace': traceback.format_exc()
                }
            )
            raise
```

**Log Security Events:**
```python
class LoginView(APIView):
    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        
        user = authenticate(username=username, password=password)
        
        if user:
            logger.info(
                f"Successful login for user {username}",
                extra={
                    'user_id': user.id,
                    'ip_address': request.META.get('REMOTE_ADDR')
                }
            )
            token, _ = Token.objects.get_or_create(user=user)
            return Response({'token': token.key})
        else:
            logger.warning(
                f"Failed login attempt for username {username}",
                extra={
                    'username': username,
                    'ip_address': request.META.get('REMOTE_ADDR')
                }
            )
            return Response(
                {'error': 'Invalid credentials'},
                status=status.HTTP_401_UNAUTHORIZED
            )
```

**What to Log:**
- All failed authentication attempts
- Denied access attempts
- Input validation errors
- Suspicious activities
- API errors with context

**What NOT to Log:**
- Passwords
- API tokens
- PII (unless necessary and secured)
- Credit card numbers
- Other sensitive data

**Structured Logging:**
```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s'
        }
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/api.log',
            'maxBytes': 1024 * 1024 * 15,  # 15MB
            'backupCount': 10,
            'formatter': 'json',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file'],
            'level': 'INFO',
            'propagate': True,
        },
        'myapp': {
            'handlers': ['file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

## Other Security Risks

### Business Logic Bugs

**Prevent Race Conditions:**
```python
from django.db import transaction

class TransferView(APIView):
    @transaction.atomic
    def post(self, request):
        # Use select_for_update to prevent race conditions
        sender = Account.objects.select_for_update().get(id=request.data['sender_id'])
        receiver = Account.objects.select_for_update().get(id=request.data['receiver_id'])
        amount = request.data['amount']
        
        if sender.balance < amount:
            raise ValidationError("Insufficient funds")
        
        sender.balance -= amount
        receiver.balance += amount
        
        sender.save()
        receiver.save()
        
        return Response({'status': 'success'})
```

**Validate Business Rules:**
```python
class OrderSerializer(serializers.ModelSerializer):
    def validate(self, data):
        # Check stock availability
        if data['quantity'] > data['product'].stock:
            raise serializers.ValidationError("Insufficient stock")
        
        # Check minimum order amount
        if data['total'] < 10:
            raise serializers.ValidationError("Minimum order amount is $10")
        
        return data
```

### Secret Management

**Never Hardcode Secrets:**
```python
# BAD
SECRET_KEY = 'django-insecure-hardcoded-key-12345'
DATABASE_PASSWORD = 'mypassword123'

# GOOD - Use environment variables
SECRET_KEY = os.environ.get('SECRET_KEY')
DATABASE_PASSWORD = os.environ.get('DB_PASSWORD')

# BETTER - Use secret management service
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://myvault.vault.azure.net/", credential=credential)
SECRET_KEY = client.get_secret("django-secret-key").value
```

**Use django-environ:**
```python
import environ

env = environ.Env()
environ.Env.read_env()

SECRET_KEY = env('SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)
DATABASE_URL = env.db()
```

## Dependency Management

### Keep Dependencies Updated

**Regular Updates:**
```bash
pip list --outdated
pip install --upgrade djangorestframework
```

**Security Audits:**
```bash
pip install safety
safety check

# Or use pip-audit
pip install pip-audit
pip-audit
```

**Requirements Management:**
```
# requirements/base.txt
Django>=4.2,<5.0
djangorestframework>=3.14,<4.0

# requirements/production.txt
-r base.txt
gunicorn>=20.1.0
```

## SAST Tools

**Use Static Analysis:**
```bash
# Bandit - Python security linter
pip install bandit
bandit -r myapp/

# Semgrep
pip install semgrep
semgrep --config=auto myapp/
```

## Summary

Key security best practices:

1. **Authorization:** Always check object permissions with `check_object_permissions()`
2. **Authentication:** Configure properly, never override without understanding
3. **Data Exposure:** Use explicit field lists (allowlist), never `__all__` or `exclude`
4. **Rate Limiting:** Implement throttling and pagination
5. **Permissions:** Change default from `AllowAny`, use appropriate permissions per endpoint
6. **Mass Assignment:** Use explicit fields in serializers
7. **Configuration:** Secure production settings, disable DEBUG, use HTTPS
8. **Injection:** Use ORM, avoid dangerous methods with user input
9. **Asset Management:** Maintain API inventory, version APIs, document deprecations
10. **Logging:** Log security events, but never log sensitive data
11. **Secrets:** Never hardcode, use environment variables or secret managers
12. **Dependencies:** Keep updated, run security audits regularly

Following these practices protects your API from the most common security vulnerabilities.
