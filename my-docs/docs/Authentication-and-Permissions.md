# Authentication and Permissions Best Practices

## Overview
Authentication identifies users, while permissions determine what authenticated users can do. This document covers best practices for implementing secure authentication and authorization in Django REST Framework.

## Authentication

### Understanding Authentication
Authentication determines the identity of a user making a request. DRF supports multiple authentication schemes that can be used together.

### Setting Up Authentication

#### Global Authentication Configuration
Configure authentication in `settings.py`:

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',  # Recommended for modern APIs
    ],
}
```

**Important:** Always configure authentication classes. The default allows unauthenticated access.

### Authentication Schemes: Detailed Comparison

> **⚠️ RECOMMENDATION:** Use JWT authentication for all new projects. Token authentication is considered legacy and maintained only for backward compatibility.

#### Approach 1: JWT Authentication (Recommended - Stateless, Scalable)

**Best For:** Modern applications, microservices, distributed systems, multi-device support

**Current Industry Standard:** JWT is the recommended authentication method for 80%+ of modern API projects.

```python
# pip install djangorestframework-simplejwt
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

**Pros:**
- ✅ Stateless (no database lookup)
- ✅ Built-in expiration
- ✅ Refresh token support
- ✅ Multi-device support
- ✅ Scalable for microservices
- ✅ Contains user info in token
- ✅ Industry standard (80%+ adoption)
- ✅ Works seamlessly with mobile and web apps

**Cons:**
- Cannot revoke until expiration (mitigated with blacklist)
- Larger token size than simple tokens
- Requires token refresh logic on client
- External dependency (djangorestframework-simplejwt)

**Use When:**
- ✅ **All new projects** (recommended default)
- Building microservices
- Need stateless authentication
- Multi-device support required
- Scalability is priority
- Modern API development

**Security Considerations:**
- ✅ Short access token lifetime (5-15 min)
- ✅ Longer refresh token lifetime (7 days)
- ✅ Implement token blacklist for revocation
- ✅ Rotate refresh tokens on use
- ✅ Use HTTPS only
- ✅ Store tokens securely on client (never in localStorage for web)

**URL Configuration:**
```python
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

#### Approach 2: Token Authentication (⚠️ LEGACY - Not Recommended)

**Status:** Legacy authentication method. Use JWT for new projects.

**Best For:** Existing projects requiring backward compatibility

```python
# settings.py
INSTALLED_APPS = [
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}
```

**Pros:**
- Simple to implement
- Built into DRF
- Easy to revoke (delete token)

**Cons:**
- ❌ Tokens stored in database (stateful)
- ❌ No expiration by default
- ❌ Single token per user (no multi-device)
- ❌ Database query per request
- ❌ No built-in refresh mechanism
- ❌ Not suitable for modern applications

**Use When:**
- Maintaining legacy projects
- Backward compatibility required
- **NOT recommended for new projects**

**Security Considerations:**
- Implement token expiration manually
- Store tokens securely on client
- Use HTTPS only
- Consider migrating to JWT

#### Approach 3: Session Authentication (Traditional Web)

**Best For:** Same-origin web apps, Django templates

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**Pros:**
- Built into Django
- Familiar for Django developers
- Works with Django auth
- CSRF protection included
- Good for same-origin requests

**Cons:**
- Requires cookies
- CSRF token needed
- Not suitable for mobile
- Same-origin only
- Session storage required

**Use When:**
- Building traditional web apps
- Using Django templates
- Same-origin AJAX requests
- Already using Django sessions
- Don't need mobile support

**Security Considerations:**
- Enable CSRF protection
- Secure cookie settings
- HTTPOnly cookies
- SameSite cookie attribute

#### Approach 4: Basic Authentication (Development Only)

**Best For:** Development, testing, internal tools

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
    ],
}
```

**Pros:**
- Extremely simple
- No setup required
- Good for testing
- Works everywhere

**Cons:**
- **Insecure** (credentials in every request)
- No token management
- Password exposed in headers
- Not suitable for production
- No logout mechanism

**Use When:**
- Development/testing only
- Internal tools over VPN
- Quick prototyping
- **NEVER for production**

**Security Considerations:**
- **Only use over HTTPS**
- Only for development
- Never in production
- Consider alternatives

### Comprehensive Comparison Matrix

| Feature | JWT (✅ Recommended) | Token (Legacy) | Session | Basic |
|---------|---------------------|----------------|---------|-------|
| **Stateless** | ✅ | ❌ | ❌ | ✅ |
| **Scalability** | High | Medium | Low | High |
| **Mobile Support** | ✅ | ✅ | ❌ | ✅ |
| **Multi-device** | ✅ | ❌ | ✅ | ✅ |
| **Revocation** | Hard (use blacklist) | Easy | Easy | N/A |
| **Expiration** | Built-in | Manual | Built-in | N/A |
| **Database Queries** | No | Yes | Yes | Yes |
| **Setup Complexity** | Medium | Low | Low | None |
| **Security** | Excellent | Good | Good | Poor |
| **Production Ready** | ✅ | ⚠️ Legacy | ✅ | ❌ |
| **Recommendation** | ✅ **Use This** | ❌ Legacy | Use for web | ❌ Never |

### Decision Tree

```
Choose Authentication Scheme:

New Project?
└─ Yes → ✅ Use JWT (recommended for 95% of cases)

Legacy Project?
├─ Already using Token Auth? → Consider migrating to JWT
└─ Already using Session Auth? → Keep for same-origin web apps

Building mobile app or SPA?
└─ ✅ Use JWT (industry standard)

Building microservices?
└─ ✅ Use JWT (stateless, scalable)

Building traditional Django template app?
└─ Session Auth acceptable (same-origin only)

Development/Testing?
└─ Basic Auth OK (HTTPS only, never production)

⚠️ DEFAULT RECOMMENDATION: Use JWT for all new projects
```

### Best Practice: JWT Authentication (Recommended)

**JWT is the current best practice for most modern applications** due to its stateless nature, scalability, and built-in security features.

```python
# pip install djangorestframework-simplejwt
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

**Why JWT is Recommended:**
- **Stateless**: No database lookups for authentication
- **Scalable**: Perfect for microservices and distributed systems
- **Secure**: Built-in expiration and refresh token mechanism
- **Multi-device**: Supports multiple concurrent sessions
- **Industry Standard**: Widely adopted and well-supported

**Use JWT For:**
- ✅ **Most new applications** (default choice)
- ✅ **Mobile apps** (iOS, Android)
- ✅ **Single Page Applications** (React, Vue, Angular)
- ✅ **Microservices architecture**
- ✅ **Multi-device support requirements**

### Alternative: Multiple Authentication Schemes

For applications requiring both web and mobile support:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**Benefits:**
- JWT for mobile/SPA clients
- Session for traditional web interface
- Flexibility for different client types
- Gradual migration path from legacy systems

### Real-World Recommendations

**Startup/MVP (Recommended):**
```python
# Start with JWT - modern and scalable
'rest_framework_simplejwt.authentication.JWTAuthentication'
```

**Production SaaS:**
```python
# JWT for scalability and multi-device support
'rest_framework_simplejwt.authentication.JWTAuthentication'
```

**Enterprise Internal:**
```python
# JWT for APIs, Session for internal web tools
'rest_framework_simplejwt.authentication.JWTAuthentication',
'rest_framework.authentication.SessionAuthentication',
```

**Public API:**
```python
# JWT for third-party developers
'rest_framework_simplejwt.authentication.JWTAuthentication'
```

**URLs:**
```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

### Per-View Authentication
Override authentication for specific views:

```python
from rest_framework.authentication import TokenAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.views import APIView

class ArticleView(APIView):
    authentication_classes = [TokenAuthentication]
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        return Response({'message': 'Authenticated with token'})
```

**Function-based views:**
```python
from rest_framework.decorators import api_view, authentication_classes, permission_classes

@api_view(['GET'])
@authentication_classes([TokenAuthentication])
@permission_classes([IsAuthenticated])
def article_list(request):
    return Response({'message': 'Authenticated'})
```

### Authentication Best Practices

1. **Use JWT as default:** JWT is the recommended choice for most modern applications
2. **Always configure authentication:** Never rely on DRF defaults
3. **Use HTTPS:** Always use HTTPS in production, especially with authentication
4. **Token security:** Store tokens securely on the client side (httpOnly cookies recommended)
5. **Token rotation:** Implement refresh token mechanisms for long-lived sessions
6. **Short access tokens:** Use 5-15 minute access token lifetime
7. **Multiple schemes:** Support multiple authentication methods when needed

## Permissions

### Understanding Permissions
Permissions determine whether a request should be granted or denied access. This document covers both basic DRF permissions and an advanced RBAC system for enterprise applications.

### Setting Up Permissions

#### Global Permission Configuration
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

**Critical Security Note:** The default is `AllowAny`, which allows unauthenticated access. Always change this for production APIs.

### Permission Approaches

#### Approach 1: Basic DRF Permissions (Simple Applications)
Use built-in DRF permission classes for simple applications with basic permission needs.

#### Approach 2: Advanced RBAC System (Enterprise Applications)
Implement a sophisticated Role-Based Access Control system for complex permission requirements.

### Built-in Permission Classes

#### 1. AllowAny
Allows unrestricted access (authenticated or not).

```python
from rest_framework.permissions import AllowAny

class PublicArticleList(APIView):
    permission_classes = [AllowAny]
```

**Use Case:** Public endpoints like registration, public content listings.

#### 2. IsAuthenticated
Requires user to be authenticated.

```python
from rest_framework.permissions import IsAuthenticated

class PrivateArticleList(APIView):
    permission_classes = [IsAuthenticated]
```

**Use Case:** Any endpoint requiring a logged-in user.

#### 3. IsAuthenticatedOrReadOnly
Authenticated users have full access; unauthenticated users have read-only access.

```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
```

**Use Case:** Public APIs where anyone can read but only authenticated users can modify.

#### 4. IsAdminUser
Only admin users (staff) can access.

```python
from rest_framework.permissions import IsAdminUser

class AdminOnlyView(APIView):
    permission_classes = [IsAdminUser]
```

**Use Case:** Admin-only endpoints.

#### 5. DjangoModelPermissions
Ties into Django's model permissions system.

```python
from rest_framework.permissions import DjangoModelPermissions

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [DjangoModelPermissions]
```

**Required Django permissions:**
- `POST` requires `add` permission
- `PUT/PATCH` requires `change` permission
- `DELETE` requires `delete` permission
- `GET` allowed by default (can be changed)

#### 6. DjangoObjectPermissions
Object-level permissions using django-guardian or similar.

```python
from rest_framework.permissions import DjangoObjectPermissions

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [DjangoObjectPermissions]
```

### Custom Permission Classes

#### Basic Custom Permission
```python
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners of an object to edit it.
    """
    
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed for any request
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Write permissions only for owner
        return obj.owner == request.user
```

#### Advanced Custom Permission
```python
class IsOwnerOrAdmin(permissions.BasePermission):
    """
    Permission to only allow owners or admin users to access.
    """
    message = 'You must be the owner or an admin to perform this action.'
    
    def has_permission(self, request, view):
        # Check if user is authenticated
        return request.user and request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        # Admin users have full access
        if request.user.is_staff:
            return True
        
        # Check if user is the owner
        return obj.owner == request.user
```

#### Permission with Custom Logic
```python
class CanEditArticle(permissions.BasePermission):
    """
    Permission to check if user can edit article based on multiple conditions.
    """
    
    def has_object_permission(self, request, view, obj):
        # Admin can always edit
        if request.user.is_staff:
            return True
        
        # Owner can edit if article is not published
        if obj.author == request.user:
            if not obj.published:
                return True
            # Published articles can only be edited within 24 hours
            from django.utils import timezone
            time_since_publish = timezone.now() - obj.published_date
            return time_since_publish.total_seconds() < 86400  # 24 hours
        
        # Editors can edit any unpublished article
        if request.user.groups.filter(name='Editors').exists():
            return not obj.published
        
        return False
```

### Advanced RBAC Permission System

#### Overview
For enterprise applications requiring sophisticated permission management, implement a Role-Based Access Control (RBAC) system with the following features:

- **Feature-action-permission hierarchy** for granular control
- **Role inheritance** for scalable permission management
- **User-specific overrides** for temporary permissions
- **Multi-tenancy support** with organization scoping
- **Comprehensive audit logging** for security compliance
- **Built-in caching** for performance optimization

#### Core Architecture

```python
# Permission resolution order:
# 1. Superuser → Always granted all permissions
# 2. User Overrides → Individual grants/revokes (highest priority)
# 3. Role Permissions → Including parent role inheritance
# 4. Organization Context → Must be active member

# Feature-Action-Permission Model:
# Feature (voice_agents, campaigns, calls)
#     ↓
# Action (create, read, update, delete, execute, export, import, manage)
#     ↓
# Permission = Feature + Action (campaigns:create)
#     ↓
# Role (collection of permissions)
#     ↓
# User (assigned to roles with possible overrides)
```

#### Service Layer Implementation

```python
# apps/permissions/services.py
from django.core.cache import cache
from django.db import transaction
from .models import Feature, Action, Permission, Role, UserPermissionOverride

class PermissionService:
    CACHE_TIMEOUT = 300  # 5 minutes
    
    @classmethod
    def check_user_permission(cls, user, feature_code, action, organization_id):
        """
        Check if user has permission for specific feature-action in organization.
        """
        # Check cache first
        cache_key = f"perm:{user.id}:{organization_id}:{feature_code}:{action}"
        cached_result = cache.get(cache_key)
        if cached_result is not None:
            return cached_result
        
        # 1. Superuser check
        if user.is_superuser:
            cls._set_cache(cache_key, True)
            return True
        
        # 2. User overrides check
        override = cls._check_user_override(user, feature_code, action, organization_id)
        if override is not None:
            cls._set_cache(cache_key, override)
            return override
        
        # 3. Role permissions check
        has_role_permission = cls._check_role_permissions(user, feature_code, action, organization_id)
        cls._set_cache(cache_key, has_role_permission)
        return has_role_permission
    
    @classmethod
    def get_user_permissions(cls, user, organization_id):
        """
        Get all permissions for user in organization.
        """
        permissions = set()
        
        # Get role permissions
        user_roles = cls._get_user_roles(user, organization_id)
        for role in user_roles:
            role_perms = cls._get_role_permissions(role)
            permissions.update(role_perms)
        
        # Apply user overrides
        overrides = cls._get_user_overrides(user, organization_id)
        for override in overrides:
            perm_key = f"{override.permission.feature.code}:{override.permission.action.code}"
            if override.override_type == 'grant':
                permissions.add(perm_key)
            elif override.override_type == 'revoke':
                permissions.discard(perm_key)
        
        return permissions
    
    @classmethod
    def grant_user_permission(cls, user, organization, permission_id, granted_by, reason=None, expires_at=None):
        """
        Grant specific permission to user with optional expiration.
        """
        with transaction.atomic():
            permission = Permission.objects.get(id=permission_id)
            override = UserPermissionOverride.objects.create(
                user=user,
                organization=organization,
                permission=permission,
                override_type='grant',
                granted_by=granted_by,
                reason=reason,
                expires_at=expires_at
            )
            
            # Clear cache
            cls._clear_user_cache(user, organization.id)
            
            # Log audit trail
            cls._log_permission_change('permission_granted', granted_by, user, permission, reason)
            
            return override
    
    @classmethod
    def revoke_user_permission(cls, user, organization, permission_id, revoked_by, reason=None):
        """
        Revoke specific permission from user.
        """
        with transaction.atomic():
            permission = Permission.objects.get(id=permission_id)
            override = UserPermissionOverride.objects.create(
                user=user,
                organization=organization,
                permission=permission,
                override_type='revoke',
                granted_by=revoked_by,
                reason=reason
            )
            
            # Clear cache
            cls._clear_user_cache(user, organization.id)
            
            # Log audit trail
            cls._log_permission_change('permission_revoked', revoked_by, user, permission, reason)
            
            return override
```

#### DRF Integration

```python
# apps/permissions/permissions.py
from rest_framework import permissions
from .services import PermissionService

class FeaturePermission(permissions.BasePermission):
    """
    DRF permission class that integrates with RBAC system.
    """
    def __init__(self, feature_code, action):
        self.feature_code = feature_code
        self.action = action
    
    def has_permission(self, request, view):
        if not request.user or not request.user.is_authenticated:
            return False
        
        return PermissionService.check_user_permission(
            user=request.user,
            feature_code=self.feature_code,
            action=self.action,
            organization_id=getattr(request, 'organization_id', None)
        )
    
    def has_object_permission(self, request, view, obj):
        # For object-level permissions, check additional constraints
        return self.has_permission(request, view)

# Usage in ViewSets
class CampaignViewSet(viewsets.ModelViewSet):
    def get_permissions(self):
        if self.action == 'create':
            return [FeaturePermission('campaigns', 'create')]
        elif self.action in ['update', 'partial_update']:
            return [FeaturePermission('campaigns', 'update')]
        elif self.action == 'destroy':
            return [FeaturePermission('campaigns', 'delete')]
        else:  # list, retrieve
            return [FeaturePermission('campaigns', 'read')]
```

#### Permission Decorator

```python
# apps/permissions/decorators.py
from functools import wraps
from django.http import HttpResponseForbidden
from .services import PermissionService

def require_permission(feature_code, action):
    """
    Decorator for function-based views requiring specific permission.
    """
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            if not PermissionService.check_user_permission(
                user=request.user,
                feature_code=feature_code,
                action=action,
                organization_id=getattr(request, 'organization_id', None)
            ):
                return HttpResponseForbidden("Insufficient permissions")
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

# Usage
@require_permission('campaigns', 'create')
def create_campaign(request):
    # View logic
    pass
```

#### Database Models

```python
# apps/permissions/models.py
import uuid
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()

class Feature(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=100, unique=True)
    category = models.CharField(max_length=50)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'features'

class Action(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    name = models.CharField(max_length=50, unique=True)
    code = models.CharField(max_length=50, unique=True)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'actions'

class Permission(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    feature = models.ForeignKey(Feature, on_delete=models.CASCADE)
    action = models.ForeignKey(Action, on_delete=models.CASCADE)
    constraints = models.JSONField(default=dict)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['feature', 'action']
        db_table = 'permissions'

class Role(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    name = models.CharField(max_length=100)
    code = models.CharField(max_length=100)
    organization = models.ForeignKey('organizations.Organization', on_delete=models.CASCADE, null=True)
    parent_role = models.ForeignKey('self', on_delete=models.CASCADE, null=True)
    is_system_role = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['code', 'organization']
        db_table = 'roles'

class RolePermission(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    role = models.ForeignKey(Role, on_delete=models.CASCADE)
    permission = models.ForeignKey(Permission, on_delete=models.CASCADE)
    granted_at = models.DateTimeField(auto_now_add=True)
    granted_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    
    class Meta:
        unique_together = ['role', 'permission']
        db_table = 'role_permissions'

class UserPermissionOverride(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    organization = models.ForeignKey('organizations.Organization', on_delete=models.CASCADE)
    permission = models.ForeignKey(Permission, on_delete=models.CASCADE)
    override_type = models.CharField(max_length=10, choices=[('grant', 'Grant'), ('revoke', 'Revoke')])
    expires_at = models.DateTimeField(null=True, blank=True)
    reason = models.TextField(blank=True)
    granted_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, related_name='granted_overrides')
    is_active = models.BooleanField(default=True)
    
    class Meta:
        unique_together = ['user', 'organization', 'permission']
        db_table = 'user_permission_overrides'

class PermissionAuditLog(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    organization = models.ForeignKey('organizations.Organization', on_delete=models.CASCADE)
    action_type = models.CharField(max_length=30)
    actor = models.ForeignKey(User, on_delete=models.CASCADE, related_name='permission_actions')
    target_user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='permission_received', null=True)
    permission = models.ForeignKey(Permission, on_delete=models.CASCADE, null=True)
    details = models.JSONField(default=dict)
    ip_address = models.GenericIPAddressField(null=True)
    user_agent = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'permission_audit_logs'
```

#### Migration and Setup

```python
# Create standard features
features = [
    ('voice_agents', 'Voice Agents', 'core'),
    ('campaigns', 'Campaigns', 'core'),
    ('calls', 'Call Management', 'core'),
    ('analytics', 'Analytics', 'advanced'),
    ('billing', 'Billing', 'admin'),
    ('settings', 'Settings', 'admin'),
]

for code, name, category in features:
    Feature.objects.get_or_create(
        code=code,
        defaults={'name': name, 'category': category}
    )

# Create standard actions
actions = ['create', 'read', 'update', 'delete', 'execute', 'export', 'import', 'manage']
for action in actions:
    Action.objects.get_or_create(code=action, defaults={'name': action})

# Create permissions for all feature-action combinations
for feature in Feature.objects.all():
    for action in Action.objects.all():
        Permission.objects.get_or_create(
            feature=feature,
            action=action,
            defaults={'is_active': True}
        )
```

### Using Multiple Permissions
Combine multiple permission classes (all must pass).

```python
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
```

### Per-Action Permissions
Set different permissions for different actions.

```python
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly, IsAdminUser

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def get_permissions(self):
        """
        Instantiate and return the list of permissions for this view.
        """
        if self.action in ['list', 'retrieve']:
            permission_classes = [IsAuthenticatedOrReadOnly]
        elif self.action == 'create':
            permission_classes = [IsAuthenticated]
        elif self.action in ['update', 'partial_update', 'destroy']:
            permission_classes = [IsAuthenticated, IsOwnerOrAdmin]
        else:
            permission_classes = [IsAdminUser]
        
        return [permission() for permission in permission_classes]
```

### Object-Level Permissions

#### Checking Object Permissions
Always check object permissions when retrieving objects.

```python
from rest_framework.exceptions import PermissionDenied

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def get_object(self):
        """
        Override to ensure object permissions are checked.
        """
        obj = get_object_or_404(self.get_queryset(), pk=self.kwargs["pk"])
        self.check_object_permissions(self.request, obj)
        return obj
```

**Critical Security:** Never override `get_object()` without checking permissions. This is API1:2019 Broken Object Level Authorization.

#### Filtering Queryset by Permissions
```python
class ArticleViewSet(viewsets.ModelViewSet):
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        """
        Return only articles the user has permission to see.
        """
        user = self.request.user
        
        if user.is_staff:
            return Article.objects.all()
        else:
            # Users see published articles and their own articles
            return Article.objects.filter(
                Q(published=True) | Q(author=user)
            )
```

### Permission Best Practices

#### Basic DRF Permissions
1. **Change default permissions:** Never use `AllowAny` as default
2. **Principle of least privilege:** Grant minimum necessary permissions
3. **Check object permissions:** Always use `check_object_permissions()`
4. **Don't override without checking:** Never override authentication/permission classes without understanding impact
5. **Use custom permissions:** Create reusable permission classes for complex logic
6. **Test permissions:** Write tests for all permission scenarios
7. **Document permissions:** Clearly document what permissions are required for each endpoint

#### Advanced RBAC System
1. **Use service layer:** Always use `PermissionService` methods, not direct model access
2. **Cache management:** Clear cache appropriately after permission changes
3. **Audit logging:** Ensure all permission changes go through service methods
4. **Role design:** Keep roles focused and use inheritance for common permissions
5. **Override expiration:** Set expiration dates for temporary permission grants
6. **Performance monitoring:** Monitor permission check performance and cache hit rates
7. **Security audits:** Regular permission audits and review of overrides

#### Migration Strategy
1. **Start simple:** Use basic DRF permissions for new projects
2. **Identify complexity:** Migrate to RBAC when you need:
   - Multi-tenancy support
   - Complex permission hierarchies
   - User-specific overrides
   - Audit trail requirements
3. **Gradual migration:** Implement RBAC alongside existing permissions
4. **Testing:** Comprehensive test coverage for both systems during migration

#### Performance Optimization
1. **Database indexes:** Create indexes on permission-related fields
2. **Caching strategy:** Use Redis for distributed caching in production
3. **Query optimization:** Minimize N+1 queries in permission checks
4. **Cache warming:** Pre-populate cache for frequently accessed permissions

#### Security Considerations
1. **Principle of least privilege:** Default roles should have minimal permissions
2. **Audit trails:** Maintain complete audit logs of all permission changes
3. **Override monitoring:** Monitor for unusual permission override patterns
4. **Regular reviews:** Schedule regular permission audits and reviews

## Third-Party Authentication Packages

### django-rest-auth
Provides registration, login, logout, password reset endpoints.

```bash
pip install dj-rest-auth
```

### django-allauth
Social authentication (Google, Facebook, GitHub, etc.).

```bash
pip install django-allauth
pip install dj-rest-auth[with_social]
```

### OAuth2
For OAuth2 provider/consumer functionality.

```bash
pip install django-oauth-toolkit
```

## Security Considerations

### 1. Secure Token Storage
```python
# Bad - Storing tokens in localStorage (vulnerable to XSS)
localStorage.setItem('token', token);

# Good - Use httpOnly cookies or secure storage
# Server sets httpOnly cookie, or use secure mobile storage
```

### 2. Token Expiration
```python
# Implement token expiration
from datetime import timedelta
from django.utils import timezone

class ExpiringToken(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    token = models.CharField(max_length=40)
    created = models.DateTimeField(auto_now_add=True)
    
    def is_expired(self):
        return timezone.now() > self.created + timedelta(hours=24)
```

### 3. Rate Limiting Authentication Endpoints
```python
from rest_framework.throttling import AnonRateThrottle

class LoginRateThrottle(AnonRateThrottle):
    rate = '5/hour'  # 5 login attempts per hour

class LoginView(APIView):
    throttle_classes = [LoginRateThrottle]
```

### 4. HTTPS Only
```python
# settings.py - Force HTTPS in production
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

### 5. Validate User Input
```python
class LoginSerializer(serializers.Serializer):
    username = serializers.CharField(max_length=150)
    password = serializers.CharField(max_length=128, write_only=True)
    
    def validate_username(self, value):
        # Prevent username enumeration
        if not User.objects.filter(username=value).exists():
            raise serializers.ValidationError("Invalid credentials")
        return value
```

## Testing Authentication and Permissions

### Testing Authentication
```python
from rest_framework.test import APITestCase
from rest_framework.authtoken.models import Token

class ArticleAPITestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.token = Token.objects.create(user=self.user)
    
    def test_authenticated_request(self):
        self.client.credentials(HTTP_AUTHORIZATION='Token ' + self.token.key)
        response = self.client.get('/api/articles/')
        self.assertEqual(response.status_code, 200)
    
    def test_unauthenticated_request(self):
        response = self.client.get('/api/articles/')
        self.assertEqual(response.status_code, 401)
```

### Testing Permissions
```python
class ArticlePermissionTestCase(APITestCase):
    def setUp(self):
        self.owner = User.objects.create_user(username='owner')
        self.other_user = User.objects.create_user(username='other')
        self.article = Article.objects.create(
            title='Test Article',
            author=self.owner
        )
    
    def test_owner_can_edit(self):
        self.client.force_authenticate(user=self.owner)
        response = self.client.patch(
            f'/api/articles/{self.article.id}/',
            {'title': 'Updated Title'}
        )
        self.assertEqual(response.status_code, 200)
    
    def test_non_owner_cannot_edit(self):
        self.client.force_authenticate(user=self.other_user)
        response = self.client.patch(
            f'/api/articles/{self.article.id}/',
            {'title': 'Updated Title'}
        )
        self.assertEqual(response.status_code, 403)
```

### Testing RBAC Permissions

```python
class TestRBACPermissions(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user('test@example.com')
        self.org = Organization.objects.create(name='Test Org')
        self.feature = Feature.objects.create(name='Campaigns', code='campaigns')
        self.action = Action.objects.create(name='create', code='create')
        self.permission = Permission.objects.create(
            feature=self.feature,
            action=self.action
        )
    
    def test_permission_check_without_role(self):
        result = PermissionService.check_user_permission(
            user=self.user,
            feature_code='campaigns',
            action='create',
            organization_id=self.org.id
        )
        self.assertFalse(result)
    
    def test_permission_check_with_role(self):
        role = Role.objects.create(name='Test Role', organization=self.org)
        RolePermission.objects.create(role=role, permission=self.permission)
        
        # Assign role to user
        OrganizationMember.objects.create(
            user=self.user,
            organization=self.org,
            role=role
        )
        
        result = PermissionService.check_user_permission(
            user=self.user,
            feature_code='campaigns',
            action='create',
            organization_id=self.org.id
        )
        self.assertTrue(result)
    
    def test_user_override_grant(self):
        # Grant temporary permission
        PermissionService.grant_user_permission(
            user=self.user,
            organization=self.org,
            permission_id=self.permission.id,
            granted_by=self.admin_user,
            expires_at=timezone.now() + timedelta(hours=1)
        )
        
        result = PermissionService.check_user_permission(
            user=self.user,
            feature_code='campaigns',
            action='create',
            organization_id=self.org.id
        )
        self.assertTrue(result)
    
    def test_user_override_revoke(self):
        # Grant role permission first
        role = Role.objects.create(name='Test Role', organization=self.org)
        RolePermission.objects.create(role=role, permission=self.permission)
        OrganizationMember.objects.create(
            user=self.user,
            organization=self.org,
            role=role
        )
        
        # Revoke specific permission
        PermissionService.revoke_user_permission(
            user=self.user,
            organization=self.org,
            permission_id=self.permission.id,
            revoked_by=self.admin_user
        )
        
        result = PermissionService.check_user_permission(
            user=self.user,
            feature_code='campaigns',
            action='create',
            organization_id=self.org.id
        )
        self.assertFalse(result)
```

### Integration Tests

```python
class TestPermissionAPI(APITestCase):
    def test_campaign_creation_requires_permission(self):
        # Test without permission
        response = self.client.post('/api/campaigns/', {})
        self.assertEqual(response.status_code, 403)
        
        # Grant permission
        PermissionService.grant_user_permission(
            user=self.user,
            organization=self.org,
            permission_id=self.permission.id,
            granted_by=self.admin_user
        )
        
        # Test with permission
        response = self.client.post('/api/campaigns/', {})
        self.assertEqual(response.status_code, 201)
```

## Summary

Key authentication and permission best practices:

### Authentication
1. **Use JWT as default:** JWT is the recommended choice for most modern applications
2. **Always configure authentication:** Never rely on DRF defaults
3. **Use appropriate schemes:** JWT for APIs/SPAs, Session for traditional web apps
4. **Secure transmission:** Always use HTTPS in production
5. **Token management:** Implement expiration, refresh, and secure storage
6. **Short access tokens:** Use 5-15 minute access token lifetime for security
7. **Multiple schemes:** Support multiple authentication methods when needed

### Permissions
8. **Choose the right approach:** Basic DRF permissions for simple apps, RBAC for enterprise
9. **Principle of least privilege:** Grant minimum necessary permissions
10. **Object-level checks:** Always check object permissions with `check_object_permissions()`
11. **Service layer abstraction:** Use service methods for RBAC operations
12. **Audit trails:** Maintain complete audit logs of permission changes
13. **Performance optimization:** Implement caching and database indexes
14. **Comprehensive testing:** Test all permission scenarios and edge cases
15. **Regular security audits:** Review permissions and overrides periodically

### Migration Strategy
16. **Start with JWT:** Begin with JWT authentication for modern applications
17. **Scale permissions:** Migrate to RBAC as permission complexity grows
18. **Maintain compatibility:** Ensure smooth transition between systems

Following these practices ensures your API is secure, scalable, and properly controls access to resources using the most appropriate permission system for your application's needs.
