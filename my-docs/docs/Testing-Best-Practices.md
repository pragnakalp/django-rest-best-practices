# Testing Best Practices

## Overview
Comprehensive testing ensures your Django REST Framework API is reliable, maintainable, and functions as expected. This document covers best practices for testing DRF APIs.

> **🏗️ Architecture Note:** When testing APIs that use the service/selector pattern (see `01-API-Design-and-Architecture.md`), test business logic in service layer tests separately from API endpoint tests.

## Testing Tools

### DRF Testing Framework

Django REST Framework provides `APITestCase` and `APIClient` for testing APIs.

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.contrib.auth.models import User
from .models import Article

class ArticleAPITestCase(APITestCase):
    def setUp(self):
        """Set up test data - runs before each test method"""
        self.client = APIClient()  # DRF's test client with API support
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user
        )
    
    def test_list_articles(self):
        """Test retrieving article list"""
        response = self.client.get('/api/articles/')  # Make GET request
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Check status code
        self.assertEqual(len(response.data), 1)  # Verify one article returned
```

### Additional Testing Tools

```python
# pytest-django for pytest support - provides better test organization and fixtures
pip install pytest-django

# factory_boy for test data generation - creates realistic test data without fixtures
pip install factory-boy

# faker for fake data - generates realistic fake data for testing
pip install faker

# coverage for code coverage
pip install coverage
```

## Test Structure

### Organize Tests by Feature

```
myapp/
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_serializers.py
│   ├── test_views.py
│   ├── test_permissions.py
│   ├── test_authentication.py
│   └── factories.py
```

### Use Factories for Test Data

```python
# tests/factories.py
import factory
from factory.django import DjangoModelFactory
from django.contrib.auth.models import User
from myapp.models import Article, Comment

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')  # Generate unique usernames
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')  # Dynamic email based on username
    first_name = factory.Faker('first_name')  # Random first name
    last_name = factory.Faker('last_name')  # Random last name

class ArticleFactory(DjangoModelFactory):
    class Meta:
        model = Article
    
    title = factory.Faker('sentence', nb_words=6)  # Random 6-word title
    content = factory.Faker('paragraph', nb_sentences=10)  # Random 10-sentence content
    author = factory.SubFactory(UserFactory)  # Create related user automatically
    published = False  # Default to unpublished

class CommentFactory(DjangoModelFactory):
    class Meta:
        model = Comment
    
    article = factory.SubFactory(ArticleFactory)  # Create related article automatically
    author = factory.SubFactory(UserFactory)  # Create related user automatically
    content = factory.Faker('paragraph')

# Usage in tests
article = ArticleFactory()  # Create single article with default values
articles = ArticleFactory.create_batch(5)  # Create 5 articles at once
published_article = ArticleFactory(published=True)  # Create article with custom value
```

## Testing CRUD Operations

### Test List Endpoint

```python
class ArticleListTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.articles = ArticleFactory.create_batch(5, author=self.user)  # Create 5 articles for testing
    
    def test_list_articles(self):
        """Test retrieving list of articles"""
        response = self.client.get('/api/articles/')  # Make GET request to list endpoint
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify success status
        self.assertEqual(len(response.data['results']), 5)  # Check all 5 articles returned
    
    def test_list_articles_pagination(self):
        """Test pagination works correctly"""
        ArticleFactory.create_batch(25)  # Add 25 more articles (total 30)
        
        response = self.client.get('/api/articles/?page=1&page_size=10')  # Request first page with 10 items
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify success
        self.assertEqual(len(response.data['results']), 10)  # Check page size
        self.assertIsNotNone(response.data['next'])  # Verify next page exists
        self.assertIsNone(response.data['previous'])  # Verify no previous page (first page)
    
    def test_list_articles_filtering(self):
        """Test filtering by published status"""
        ArticleFactory.create_batch(3, published=True)  # Create 3 published articles
        ArticleFactory.create_batch(2, published=False)  # Create 2 unpublished articles
        
        response = self.client.get('/api/articles/?published=true')  # Filter for published articles
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify success
        self.assertEqual(len(response.data['results']), 3)  # Check only published articles returned
```

### Test Create Endpoint

```python
class ArticleCreateTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.client.force_authenticate(user=self.user)  # Authenticate user for protected endpoint
        self.valid_data = {  # Valid article data for testing
            'title': 'New Article',
            'content': 'Article content here',
            'published': False
        }
    
    def test_create_article_success(self):
        """Test creating article with valid data"""
        response = self.client.post('/api/articles/', self.valid_data)  # POST request to create article
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)  # Verify creation success
        self.assertEqual(Article.objects.count(), 1)  # Check one article exists in database
        self.assertEqual(response.data['title'], self.valid_data['title'])  # Verify returned data
        self.assertEqual(response.data['author'], self.user.id)  # Verify author set correctly
    
    def test_create_article_without_authentication(self):
        """Test creating article without authentication fails"""
        self.client.force_authenticate(user=None)  # Remove authentication
        
        response = self.client.post('/api/articles/', self.valid_data)  # Try to create without auth
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)  # Verify auth required
        self.assertEqual(Article.objects.count(), 0)  # Verify no article created
    
    def test_create_article_invalid_data(self):
        """Test creating article with invalid data fails"""
        invalid_data = {'title': 'ab'}  # Too short title (violates validation)
        
        response = self.client.post('/api/articles/', invalid_data)  # POST with invalid data
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)  # Verify validation error
        self.assertIn('title', response.data)  # Check title field has errors
        self.assertEqual(Article.objects.count(), 0)  # Verify no article created
    
    def test_create_article_missing_required_field(self):
        """Test creating article without required field fails"""
        incomplete_data = {'content': 'Content without title'}  # Missing required title
        
        response = self.client.post('/api/articles/', incomplete_data)  # POST with incomplete data
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)  # Verify validation error
        self.assertIn('title', response.data)  # Check title field error exists
```

### Test Retrieve Endpoint

```python
class ArticleRetrieveTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.article = ArticleFactory(author=self.user)  # Create article for testing
    
    def test_retrieve_article(self):
        """Test retrieving single article"""
        response = self.client.get(f'/api/articles/{self.article.id}/')  # GET specific article
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify success
        self.assertEqual(response.data['id'], self.article.id)  # Check correct article returned
        self.assertEqual(response.data['title'], self.article.title)  # Verify data integrity
    
    def test_retrieve_nonexistent_article(self):
        """Test retrieving non-existent article returns 404"""
        response = self.client.get('/api/articles/99999/')  # Request non-existent article
        
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)  # Verify 404 response
```

### Test Update Endpoint

```python
class ArticleUpdateTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.other_user = UserFactory()  # Different user for permission testing
        self.article = ArticleFactory(author=self.user)  # Article owned by self.user
        self.client.force_authenticate(user=self.user)  # Authenticate as owner
    
    def test_update_article_success(self):
        """Test updating article with valid data"""
        update_data = {'title': 'Updated Title'}  # Data to update
        
        response = self.client.patch(  # PATCH for partial update
            f'/api/articles/{self.article.id}/',
            update_data
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify update success
        self.article.refresh_from_db()  # Reload article from database
        self.assertEqual(self.article.title, 'Updated Title')  # Verify update applied
    
    def test_update_article_without_permission(self):
        """Test non-owner cannot update article"""
        self.client.force_authenticate(user=self.other_user)  # Switch to different user
        update_data = {'title': 'Hacked Title'}  # Attempted unauthorized update
        
        response = self.client.patch(  # Try to update as non-owner
            f'/api/articles/{self.article.id}/',
            update_data
        )
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)  # Verify permission denied
        self.article.refresh_from_db()  # Reload to verify no changes
        self.assertNotEqual(self.article.title, 'Hacked Title')  # Verify title unchanged
```

### Test Delete Endpoint

```python
class ArticleDeleteTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.other_user = UserFactory()  # Different user for permission testing
        self.article = ArticleFactory(author=self.user)  # Article owned by self.user
        self.client.force_authenticate(user=self.user)  # Authenticate as owner
    
    def test_delete_article_success(self):
        """Test deleting article"""
        response = self.client.delete(f'/api/articles/{self.article.id}/')  # DELETE request
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)  # Verify deletion success
        self.assertEqual(Article.objects.count(), 0)  # Verify article removed from database
    
    def test_delete_article_without_permission(self):
        """Test non-owner cannot delete article"""
        self.client.force_authenticate(user=self.other_user)  # Switch to different user
        
        response = self.client.delete(f'/api/articles/{self.article.id}/')  # Try to delete as non-owner
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)  # Verify permission denied
        self.assertEqual(Article.objects.count(), 1)  # Verify article still exists
```

## Testing Authentication

> **✅ STANDARD:** Use JWT authentication for all new projects. Token authentication is legacy.
>
> **📖 Reference:** See `04-Authentication-and-Permissions.md` for authentication setup.

### Test JWT Authentication (Standard)

```python
from rest_framework_simplejwt.tokens import RefreshToken

class JWTAuthenticationTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.refresh = RefreshToken.for_user(self.user)  # Generate JWT refresh token
        self.access_token = str(self.refresh.access_token)  # Extract access token
    
    def test_authentication_with_valid_jwt(self):
        """Test API access with valid JWT token"""
        self.client.credentials(HTTP_AUTHORIZATION=f'Bearer {self.access_token}')  # Set JWT header
        
        response = self.client.get('/api/articles/')  # Make authenticated request
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify authenticated access works
    
    def test_authentication_with_invalid_jwt(self):
        """Test API access with invalid JWT token"""
        self.client.credentials(HTTP_AUTHORIZATION='Bearer invalid_token')  # Set invalid token
        
        response = self.client.get('/api/articles/')  # Make request with invalid token
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)  # Verify auth fails
    
    def test_authentication_without_jwt(self):
        """Test API access without JWT token"""
        response = self.client.get('/api/articles/')  # Make request without token
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)  # Verify auth required
    
    def test_jwt_token_refresh(self):
        """Test refreshing JWT token"""
        response = self.client.post('/api/token/refresh/', {  # Request token refresh
            'refresh': str(self.refresh)  # Send refresh token
        })
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify refresh success
        self.assertIn('access', response.data)  # Check new access token provided
    
    def test_obtain_jwt_tokens(self):
        """Test obtaining JWT tokens"""
        response = self.client.post('/api/token/', {  # Request tokens with credentials
            'username': self.user.username,
            'password': 'testpass123'
        })
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)  # Verify token generation
        self.assertIn('access', response.data)  # Check access token present
        self.assertIn('refresh', response.data)  # Check refresh token present
```

### Test Token Authentication (⚠️ LEGACY - Not Recommended)

> **⚠️ WARNING:** Token authentication is legacy. Use JWT authentication tests above for new projects.
>
> This section is maintained only for backward compatibility with existing projects.

```python
from rest_framework.authtoken.models import Token

class TokenAuthenticationTestCase(APITestCase):
    """Legacy Token authentication tests - Use JWT for new projects"""
    
    def setUp(self):
        self.user = UserFactory()
        self.token = Token.objects.create(user=self.user)
    
    def test_authentication_with_valid_token(self):
        """Test API access with valid token (LEGACY)"""
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {self.token.key}')
        response = self.client.get('/api/articles/')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

## Testing Permissions

### Test Object-Level Permissions

```python
class ArticlePermissionTestCase(APITestCase):
    def setUp(self):
        self.owner = UserFactory()
        self.other_user = UserFactory()
        self.admin = UserFactory(is_staff=True)
        self.article = ArticleFactory(author=self.owner)
    
    def test_owner_can_update(self):
        """Test owner can update their article"""
        self.client.force_authenticate(user=self.owner)
        
        response = self.client.patch(
            f'/api/articles/{self.article.id}/',
            {'title': 'Updated by owner'}
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_non_owner_cannot_update(self):
        """Test non-owner cannot update article"""
        self.client.force_authenticate(user=self.other_user)
        
        response = self.client.patch(
            f'/api/articles/{self.article.id}/',
            {'title': 'Updated by other'}
        )
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_admin_can_update(self):
        """Test admin can update any article"""
        self.client.force_authenticate(user=self.admin)
        
        response = self.client.patch(
            f'/api/articles/{self.article.id}/',
            {'title': 'Updated by admin'}
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_anonymous_can_read(self):
        """Test anonymous users can read articles"""
        response = self.client.get(f'/api/articles/{self.article.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_anonymous_cannot_create(self):
        """Test anonymous users cannot create articles"""
        response = self.client.post('/api/articles/', {
            'title': 'New Article',
            'content': 'Content'
        })
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
```

## Testing Serializers

### Test Serializer Validation

```python
class ArticleSerializerTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
    
    def test_valid_serializer(self):
        """Test serializer with valid data"""
        data = {
            'title': 'Valid Title',
            'content': 'Valid content',
            'published': False
        }
        
        serializer = ArticleSerializer(data=data)
        
        self.assertTrue(serializer.is_valid())
        self.assertEqual(serializer.validated_data['title'], data['title'])
    
    def test_invalid_title_too_short(self):
        """Test serializer rejects short title"""
        data = {
            'title': 'ab',  # Too short
            'content': 'Valid content'
        }
        
        serializer = ArticleSerializer(data=data)
        
        self.assertFalse(serializer.is_valid())
        self.assertIn('title', serializer.errors)
    
    def test_missing_required_field(self):
        """Test serializer rejects missing required field"""
        data = {'content': 'Content without title'}
        
        serializer = ArticleSerializer(data=data)
        
        self.assertFalse(serializer.is_valid())
        self.assertIn('title', serializer.errors)
    
    def test_read_only_fields(self):
        """Test read-only fields are not writable"""
        data = {
            'title': 'Title',
            'content': 'Content',
            'id': 999,  # Read-only field
            'created_at': '2020-01-01T00:00:00Z'  # Read-only field
        }
        
        serializer = ArticleSerializer(data=data)
        serializer.is_valid()
        article = serializer.save(author=self.user)
        
        self.assertNotEqual(article.id, 999)
        self.assertNotEqual(
            article.created_at.isoformat(),
            '2020-01-01T00:00:00+00:00'
        )
```

## Testing Filtering and Searching

```python
class ArticleFilterTestCase(APITestCase):
    def setUp(self):
        self.user1 = UserFactory()
        self.user2 = UserFactory()
        
        ArticleFactory.create_batch(3, author=self.user1, published=True)
        ArticleFactory.create_batch(2, author=self.user2, published=False)
    
    def test_filter_by_published(self):
        """Test filtering by published status"""
        response = self.client.get('/api/articles/?published=true')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 3)
    
    def test_filter_by_author(self):
        """Test filtering by author"""
        response = self.client.get(f'/api/articles/?author={self.user1.id}')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 3)
    
    def test_search_in_title(self):
        """Test searching in title"""
        ArticleFactory(title='Django REST Framework Tutorial')
        
        response = self.client.get('/api/articles/?search=Django')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertGreaterEqual(len(response.data['results']), 1)
    
    def test_ordering(self):
        """Test ordering results"""
        response = self.client.get('/api/articles/?ordering=-created_at')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        results = response.data['results']
        
        # Verify descending order
        for i in range(len(results) - 1):
            self.assertGreaterEqual(
                results[i]['created_at'],
                results[i + 1]['created_at']
            )
```

## Testing Custom Actions

```python
class ArticleCustomActionTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.article = ArticleFactory(author=self.user, published=False)
        self.client.force_authenticate(user=self.user)
    
    def test_publish_action(self):
        """Test custom publish action"""
        response = self.client.post(f'/api/articles/{self.article.id}/publish/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.article.refresh_from_db()
        self.assertTrue(self.article.published)
        self.assertIsNotNone(self.article.published_date)
    
    def test_publish_already_published_article(self):
        """Test publishing already published article"""
        self.article.published = True
        self.article.save()
        
        response = self.client.post(f'/api/articles/{self.article.id}/publish/')
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

## Testing Rate Limiting

```python
class RateLimitTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.client.force_authenticate(user=self.user)
    
    def test_rate_limit_not_exceeded(self):
        """Test requests within rate limit succeed"""
        for _ in range(5):
            response = self.client.get('/api/articles/')
            self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_rate_limit_exceeded(self):
        """Test requests exceeding rate limit are throttled"""
        # Make requests until throttled
        for i in range(100):
            response = self.client.get('/api/articles/')
            if response.status_code == status.HTTP_429_TOO_MANY_REQUESTS:
                break
        
        self.assertEqual(response.status_code, status.HTTP_429_TOO_MANY_REQUESTS)
        self.assertIn('detail', response.data)
```

## Testing with Mocks

```python
from unittest.mock import patch, Mock

class ArticleExternalServiceTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        self.client.force_authenticate(user=self.user)
    
    @patch('myapp.services.external_api.publish_to_social_media')
    def test_publish_with_social_media(self, mock_publish):
        """Test publishing article also publishes to social media"""
        mock_publish.return_value = {'status': 'success'}
        
        article = ArticleFactory(author=self.user)
        response = self.client.post(f'/api/articles/{article.id}/publish/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        mock_publish.assert_called_once_with(article)
    
    @patch('myapp.services.email.send_notification_email')
    def test_create_article_sends_notification(self, mock_send_email):
        """Test creating article sends notification email"""
        data = {
            'title': 'New Article',
            'content': 'Content'
        }
        
        response = self.client.post('/api/articles/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        mock_send_email.assert_called_once()
```

## Performance Testing

```python
from django.test import override_settings
from django.db import connection
from django.test.utils import override_settings

class ArticlePerformanceTestCase(APITestCase):
    def setUp(self):
        self.user = UserFactory()
        ArticleFactory.create_batch(100, author=self.user)
    
    def test_list_query_count(self):
        """Test list endpoint doesn't have N+1 queries"""
        with self.assertNumQueries(3):  # Adjust based on your implementation
            response = self.client.get('/api/articles/')
            self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_detail_query_count(self):
        """Test detail endpoint query efficiency"""
        article = Article.objects.first()
        
        with self.assertNumQueries(2):
            response = self.client.get(f'/api/articles/{article.id}/')
            self.assertEqual(response.status_code, status.HTTP_200_OK)
```

## Integration Tests

```python
class ArticleIntegrationTestCase(APITestCase):
    def test_complete_article_workflow(self):
        """Test complete article creation, update, and deletion workflow"""
        # Create user and authenticate
        user = UserFactory()
        self.client.force_authenticate(user=user)
        
        # Create article
        create_response = self.client.post('/api/articles/', {
            'title': 'Integration Test Article',
            'content': 'Test content'
        })
        self.assertEqual(create_response.status_code, status.HTTP_201_CREATED)
        article_id = create_response.data['id']
        
        # Retrieve article
        retrieve_response = self.client.get(f'/api/articles/{article_id}/')
        self.assertEqual(retrieve_response.status_code, status.HTTP_200_OK)
        
        # Update article
        update_response = self.client.patch(
            f'/api/articles/{article_id}/',
            {'title': 'Updated Title'}
        )
        self.assertEqual(update_response.status_code, status.HTTP_200_OK)
        self.assertEqual(update_response.data['title'], 'Updated Title')
        
        # Publish article
        publish_response = self.client.post(f'/api/articles/{article_id}/publish/')
        self.assertEqual(publish_response.status_code, status.HTTP_200_OK)
        
        # Delete article
        delete_response = self.client.delete(f'/api/articles/{article_id}/')
        self.assertEqual(delete_response.status_code, status.HTTP_204_NO_CONTENT)
        
        # Verify deletion
        verify_response = self.client.get(f'/api/articles/{article_id}/')
        self.assertEqual(verify_response.status_code, status.HTTP_404_NOT_FOUND)
```

## Summary

Key testing best practices:

1. **Comprehensive Coverage:** Test all CRUD operations, authentication, permissions
2. **Use Factories:** Generate test data with factory_boy
3. **Test Edge Cases:** Invalid data, missing fields, permission denials
4. **Authentication Tests:** Test with and without authentication
5. **Permission Tests:** Test object-level and view-level permissions
6. **Serializer Tests:** Test validation, required fields, read-only fields
7. **Performance Tests:** Use `assertNumQueries` to catch N+1 problems
8. **Integration Tests:** Test complete workflows
9. **Mock External Services:** Use mocks for external API calls

Comprehensive testing ensures your API is reliable and maintainable.
