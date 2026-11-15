# Django / DRF Code Review Guide

> A Django / DRF code review guide covering core topics such as security review, N+1 query optimization, serializer anti-patterns, ViewSet best practices, async views, and production security configuration.

## Table of Contents

- [Security Review](#security-review)
- [N+1 Query Optimization](#n1-query-optimization)
- [Serializer Anti-Patterns](#serializer-anti-patterns)
- [ViewSet Best Practices](#viewset-best-practices)
- [Async Views](#async-views)
- [Middleware and Settings](#middleware-and-settings)
- [Review Checklist](#review-checklist)

---

## Security Review

### XSS Protection

The Django template engine auto-escapes by default. Review focus: usage of `mark_safe`, `autoescape off`, and `format_html`.

> **For cross-framework XSS protection, see the [XSS Prevention Guide](cross-cutting/xss-prevention.md)**, including React/Vue/Angular/Svelte examples and CSP configuration.

### CSRF Protection

```python
from django.views.decorators.csrf import csrf_exempt

# ❌ CSRF protection disabled
@csrf_exempt
def process_payment(request):
    # Any malicious site can submit this form
    amount = request.POST["amount"]
    charge(amount)

# ✅ Keep the default CSRF protection
from django.middleware.csrf import CsrfViewMiddleware

# settings.py - make sure the CSRF middleware is enabled
MIDDLEWARE = [
    # ...
    "django.middleware.csrf.CsrfViewMiddleware",
    # ...
]

# ✅ APIs use token authentication instead of CSRF
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
}

# ✅ Frontend AJAX requests include the CSRF token
# JavaScript: fetch("/api/endpoint/", {
#   headers: {"X-CSRFToken": document.querySelector("[name=csrfmiddlewaretoken]").value}
# })
```

### Cookie Security Settings

```python
# settings.py

# ❌ Insecure cookie configuration
SESSION_COOKIE_SECURE = False
CSRF_COOKIE_SECURE = False
SESSION_COOKIE_HTTPONLY = False

# ✅ Secure cookie configuration for production
SESSION_COOKIE_SECURE = True    # HTTPS only
SESSION_COOKIE_HTTPONLY = True   # Not readable by JavaScript
SESSION_COOKIE_SAMESITE = "Lax"  # Mitigates CSRF
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = "Lax"
```

### SQL Injection Protection

The Django ORM parameterizes queries automatically. Review focus: string concatenation in `raw()`, `extra()`, `RawSQL`, and `connection.cursor()`.

> **For cross-language SQL injection protection, see the [SQL Injection Prevention Guide](cross-cutting/sql-injection-prevention.md)**, including Python/Java/Go/Node.js/PHP/C# examples and unsafe ORM usage.

### File Upload Security

```python
# settings.py

# ❌ Insecure default upload configuration
FILE_UPLOAD_MAX_MEMORY_SIZE = 2621440  # 2.5 MB - acceptable
MEDIA_ROOT = "/var/www/uploads"         # Under the web root
ALLOWED_UPLOAD_TYPES = None             # No type restriction

# ✅ Limit upload size and location
DATA_UPLOAD_MAX_MEMORY_SIZE = 10485760  # 10 MB
FILE_UPLOAD_MAX_MEMORY_SIZE = 2621440   # 2.5 MB in-memory
MEDIA_ROOT = "/srv/media/"              # Outside the web root

# ✅ Validate file types
import mimetypes
from pathlib import Path

ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".pdf"}

def validate_upload(file):
    ext = Path(file.name).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ValidationError(f"File type {ext} is not allowed.")
    mime, _ = mimetypes.guess_type(file.name)
    if mime not in {"image/jpeg", "image/png", "application/pdf"}:
        raise ValidationError("Invalid MIME type.")
```

---

## N+1 Query Optimization

> 📖 For general principles and cross-language solutions, see the [N+1 Queries Cross-Language Guide](cross-cutting/n-plus-one-queries.md)

### select_related (ForeignKey / OneToOne)

```python
# ❌ N+1: one publisher query per book
books = Book.objects.all()
for book in books:
    print(book.publisher.name)  # N extra queries

# ✅ select_related uses a single JOIN query
books = Book.objects.select_related("publisher")
for book in books:
    print(book.publisher.name)  # No extra queries

# ✅ Multi-level relations
books = Book.objects.select_related("publisher", "publisher__country")

# ✅ Fetch only the needed fields (deferred loading optimization)
books = Book.objects.select_related("publisher").only(
    "title", "publisher__name"
)
```

### prefetch_related (M2M / Reverse ForeignKey)

```python
# ❌ N+1: one books query per author
authors = Author.objects.all()
for author in authors:
    print(author.books.all())  # N extra queries

# ✅ prefetch_related: two queries + merge in Python
authors = Author.objects.prefetch_related("books")
for author in authors:
    print(list(author.books.all()))  # No extra queries

# ✅ Nested prefetch
authors = Author.objects.prefetch_related(
    "books",
    "books__publisher",
)

# ✅ Prefetch objects control prefetch behavior
from django.db.models import Prefetch

authors = Author.objects.prefetch_related(
    Prefetch(
        "books",
        queryset=Book.objects.filter(published=True).only("title", "author_id"),
        to_attr="published_books",
    )
)
for author in authors:
    print(author.published_books)  # Already filtered, stored in to_attr
```

### QuerySet Cache Misuse

```python
# ❌ count() then iterate - two queries
qs = Book.objects.all()
count = qs.count()          # Query 1: SELECT COUNT(*) - does not populate the cache
titles = [b.title for b in qs]  # Query 2: SELECT * - re-evaluated

# ✅ When you need both the objects and the count, use len() to evaluate once and reuse the cache
qs = Book.objects.all()
count = len(qs)             # Query 1: SELECT * - loads and caches everything
titles = [b.title for b in qs]  # Reuses the cache, no new query

# ✅ If you need to iterate multiple times, convert to a list first
books = list(Book.objects.all())  # One query
count = len(books)
titles = [b.title for b in books]
```

### Slicing/Indexing Does Not Populate the Cache

```python
# ❌ Repeatedly indexing an unevaluated QuerySet - hits the database every time
qs = Book.objects.all()
qs[0]   # Query 1: SELECT ... LIMIT 1
qs[0]   # Query 2 - slicing/indexing does not populate the cache

# ✅ Evaluate the whole thing first; the cache holds all rows and later indexing uses it
qs = Book.objects.all()
list(qs)    # SELECT * - evaluates and caches all rows
qs[0]       # From cache, no query
qs[5]       # From cache, no query

# ✅ When you only need the first N rows, slice once and convert to a list
books = list(Book.objects.all()[:10])  # One query: SELECT ... LIMIT 10
first = books[0]
rest = books[1:]   # Already a Python list, no query
```

### len() vs count()

```python
# ❌ len() loads all objects into memory
total = len(Book.objects.all())  # SELECT * FROM book - loads the whole table

# ✅ count() counts on the database side
total = Book.objects.count()  # SELECT COUNT(*) - efficient

# ✅ If you already need the QuerySet results, then use len
books = list(Book.objects.filter(published=True))
total = len(books)  # Already in memory, no extra query needed
```

### if qs vs qs.exists()

```python
# ❌ if qs loads all records
qs = Book.objects.filter(author_id=author_id)
if qs:  # SELECT * FROM book WHERE ... - loads everything
    return qs[0]

# ✅ exists() only checks whether any record exists
if Book.objects.filter(author_id=author_id).exists():
    return Book.objects.filter(author_id=author_id).first()

# ✅ Or use get/first directly and check for None
book = Book.objects.filter(author_id=author_id).first()
if book is not None:
    return book
```

---

## Serializer Anti-Patterns

### Exclude Sensitive Fields

```python
from rest_framework import serializers

# ❌ __all__ exposes every field, including sensitive data
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = "__all__"  # Password hash, is_superuser, etc. are all exposed

# ✅ Explicitly list allowed fields
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "username", "email", "first_name", "last_name"]

# ✅ Be careful when using exclude too
class UserProfileSerializer(serializers.ModelSerializer):
    class Meta:
        model = UserProfile
        exclude = ["internal_notes", "admin_flags"]

# ✅ Use write_only for password fields
class RegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)

    class Meta:
        model = User
        fields = ["id", "username", "email", "password"]

    def create(self, validated_data):
        user = User(**validated_data)
        user.set_password(validated_data["password"])
        user.save()
        return user
```

### Missing Validation

```python
from rest_framework import serializers

# ❌ No validation, trusts all input
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["quantity", "price", "discount"]

# ✅ Field-level validation
class OrderSerializer(serializers.ModelSerializer):
    quantity = serializers.IntegerField(min_value=1, max_value=100)
    price = serializers.DecimalField(max_digits=10, decimal_places=2, min_value=0)
    discount = serializers.DecimalField(
        max_digits=5, decimal_places=2, min_value=0, max_value=1, required=False
    )

    class Meta:
        model = Order
        fields = ["quantity", "price", "discount"]

# ✅ Object-level validation
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["quantity", "price", "discount"]

    def validate(self, attrs):
        if attrs.get("discount", 0) > 0.5 and attrs.get("quantity", 0) < 10:
            raise serializers.ValidationError(
                "Bulk discount requires minimum 10 items."
            )
        return attrs

# ✅ Custom field validation methods
class BookingSerializer(serializers.ModelSerializer):
    class Meta:
        model = Booking
        fields = ["start_date", "end_date", "room"]

    def validate_start_date(self, value):
        if value < date.today():
            raise serializers.ValidationError("Start date cannot be in the past.")
        return value

    def validate(self, attrs):
        if attrs["end_date"] <= attrs["start_date"]:
            raise serializers.ValidationError("End date must be after start date.")
        return attrs
```

### Nested Writes

```python
from rest_framework import serializers

# ❌ Nested serializer without create/update implemented
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ["id", "name"]

class ArticleSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True)  # Nested writes will fail

    class Meta:
        model = Article
        fields = ["id", "title", "tags"]

# ✅ Option 1: nested read-only + PrimaryKeyRelatedField for writes
class ArticleSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True, read_only=True)
    tag_ids = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        many=True,
        write_only=True,
        source="tags",
    )

    class Meta:
        model = Article
        fields = ["id", "title", "tags", "tag_ids"]

# ✅ Option 2: implement create() to handle nesting
class ArticleSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True)

    class Meta:
        model = Article
        fields = ["id", "title", "tags"]

    def create(self, validated_data):
        tags_data = validated_data.pop("tags")
        article = Article.objects.create(**validated_data)
        for tag_data in tags_data:
            tag, _ = Tag.objects.get_or_create(**tag_data)
            article.tags.add(tag)
        return article

    def update(self, instance, validated_data):
        tags_data = validated_data.pop("tags", None)
        instance = super().update(instance, validated_data)
        if tags_data is not None:
            instance.tags.clear()
            for tag_data in tags_data:
                tag, _ = Tag.objects.get_or_create(**tag_data)
                instance.tags.add(tag)
        return instance
```

### Missing read_only_fields

```python
from rest_framework import serializers

# ❌ Computed and auto-populated fields can be overwritten by users
class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ["id", "body", "author", "created_at", "updated_at"]
        # created_at, updated_at, author can be tampered with by the client

# ✅ Mark read-only fields
class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ["id", "body", "author", "created_at", "updated_at"]
        read_only_fields = ["author", "created_at", "updated_at"]

# ✅ Set read-only fields in the view (e.g., the current user)
class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer

    def get_serializer_context(self):
        context = super().get_serializer_context()
        context["request"] = self.request
        return context

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

---

## ViewSet Best Practices

### Choose the Right Base Class

```python
from rest_framework import viewsets

# ❌ ModelViewSet provides full CRUD, but only reads are needed
class TagViewSet(viewsets.ModelViewSet):
    queryset = Tag.objects.all()
    serializer_class = TagSerializer
    # Exposes destroy, update, create - tags should not be freely modified

# ✅ Use ReadOnlyModelViewSet for read-only cases
class TagViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Tag.objects.all()
    serializer_class = TagSerializer
    # Provides only list and retrieve

# ✅ Use mixins when you need a custom set of actions
from rest_framework import mixins

class TagViewSet(
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    mixins.CreateModelMixin,
    generics.GenericAPIView,
):
    queryset = Tag.objects.all()
    serializer_class = TagSerializer
```

### Per-User Data Scoping

```python
from rest_framework import viewsets

# ❌ Any user can see all data
class DocumentViewSet(viewsets.ModelViewSet):
    queryset = Document.objects.all()
    serializer_class = DocumentSerializer

# ✅ get_queryset scopes data to the current user
class DocumentViewSet(viewsets.ModelViewSet):
    serializer_class = DocumentSerializer

    def get_queryset(self):
        return Document.objects.filter(
            owner=self.request.user
        ).select_related("owner")

# ✅ Admins see everything, regular users see their own
class DocumentViewSet(viewsets.ModelViewSet):
    serializer_class = DocumentSerializer

    def get_queryset(self):
        qs = Document.objects.select_related("owner")
        if self.request.user.is_staff:
            return qs
        return qs.filter(owner=self.request.user)

# ✅ perform_create automatically associates the current user
class DocumentViewSet(viewsets.ModelViewSet):
    serializer_class = DocumentSerializer

    def get_queryset(self):
        return Document.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Permission Control

```python
from rest_framework import permissions, viewsets

# ❌ No permission control
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

# ✅ Class-level permissions
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [permissions.IsAuthenticated]

# ✅ Action-level permissions
from rest_framework.decorators import action

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

    def get_permissions(self):
        if self.action in ("list", "retrieve"):
            return [permissions.AllowAny()]
        if self.action == "create":
            return [permissions.IsAuthenticated()]
        return [permissions.IsAdminUser()]

# ✅ Custom object-level permissions
class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.owner == request.user
```

### Pagination and Throttling

```python
# settings.py

# ❌ No pagination or throttling configured
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
    ],
}

# ✅ Global pagination and throttling
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/hour",
        "user": "1000/hour",
    },
}

# ✅ Custom paginator
from rest_framework.pagination import PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 25
    page_size_query_param = "page_size"
    max_page_size = 100

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    pagination_class = StandardPagination
```

---

## Async Views

### Using the Sync ORM Correctly in Async Views

```python
import asyncio
from asgiref.sync import sync_to_async
from django.http import JsonResponse

# ❌ Calling the sync ORM directly in an async view - blocks the event loop
async def user_list(request):
    users = User.objects.all()  # Synchronous ORM call in async context!
    data = [{"id": u.id, "name": u.username} for u in users]
    return JsonResponse(data, safe=False)

# ✅ Use the async ORM (Django 4.1+)
async def user_list(request):
    users = User.objects.all()
    data = []
    async for user in users:  # async iteration
        data.append({"id": user.id, "name": user.username})
    return JsonResponse(data, safe=False)

# ✅ Use aget / afilter / acreate
async def user_detail(request, pk):
    user = await User.objects.aget(pk=pk)
    return JsonResponse({"id": user.id, "name": user.username})

# ✅ Use sync_to_async for complex queries
@sync_to_async
def get_user_with_profile(pk):
    return User.objects.select_related("profile").get(pk=pk)

async def user_profile(request, pk):
    user = await get_user_with_profile(pk)
    return JsonResponse({
        "id": user.id,
        "name": user.username,
        "bio": user.profile.bio,
    })
```

### Missing await

```python
from django.http import JsonResponse

# ❌ Forgot await - the coroutine never runs; you get a coroutine object instead of data
async def user_detail(request, pk):
    user = User.objects.aget(pk=pk)  # Missing await!
    # user is a coroutine object, not a User instance
    return JsonResponse({"name": user.username})  # RuntimeError

# ✅ Always await async ORM calls
async def user_detail(request, pk):
    user = await User.objects.aget(pk=pk)
    return JsonResponse({"name": user.username})

# ✅ Use the async version of get_object_or_404
from django.shortcuts import aget_object_or_404

async def user_detail(request, pk):
    user = await aget_object_or_404(User, pk=pk)
    return JsonResponse({"name": user.username})
```

### Transactions in Async Views

```python
from django.db import transaction
from asgiref.sync import sync_to_async

# ❌ transaction.atomic() is synchronous and cannot be used directly in async code
async def create_order(request):
    async with transaction.atomic():  # Error! Not async-compatible
        order = await Order.objects.acreate(total=100)
        await OrderItem.objects.acreate(order=order, product_id=1)
    return JsonResponse({"order_id": order.id})

# ✅ Wrap the transaction block with sync_to_async
@sync_to_async
def _create_order_with_items():
    with transaction.atomic():
        order = Order.objects.create(total=100)
        OrderItem.objects.create(order=order, product_id=1)
        return order.id

async def create_order(request):
    order_id = await _create_order_with_items()
    return JsonResponse({"order_id": order_id})

# ✅ Bundle multiple operations into one sync_to_async call
@sync_to_async
def _bulk_create_products(items):
    with transaction.atomic():
        products = Product.objects.bulk_create([Product(**i) for i in items])
        return [p.id for p in products]

async def import_products(request):
    ids = await _bulk_create_products(request.data)
    return JsonResponse({"ids": ids})
```

### Sync Middleware Slows Down Async Performance

```python
# ❌ Sync middleware downgrades async views to synchronous execution
class TimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):  # sync - blocks async views
        start = time.time()
        response = self.get_response(request)
        elapsed = time.time() - start
        response["X-Elapsed"] = str(elapsed)
        return response

# ✅ Async-capable middleware: async def __call__, and mark the instance in __init__
import time
from asgiref.sync import iscoroutinefunction, markcoroutinefunction

class TimingMiddleware:
    async_capable = True
    sync_capable = False

    def __init__(self, get_response):
        self.get_response = get_response
        # Mark itself when get_response is a coroutine function, so Django awaits this instance
        if iscoroutinefunction(self.get_response):
            markcoroutinefunction(self)

    async def __call__(self, request):
        start = time.time()
        response = await self.get_response(request)
        elapsed = time.time() - start
        response["X-Elapsed"] = str(elapsed)
        return response

# ✅ To support both sync and async, use a factory function + the built-in decorator
from django.utils.decorators import sync_and_async_middleware
```

### async for Iteration Patterns

```python
from django.http import JsonResponse

# ❌ Synchronously iterating a large QuerySet blocks in an async view
async def export_users(request):
    users = User.objects.all()
    data = []  # Sync iteration blocks the event loop
    for user in users:
        data.append({"id": user.id, "name": user.username})
    return JsonResponse(data, safe=False)

# ✅ Use async for for asynchronous iteration
async def export_users(request):
    data = []
    async for user in User.objects.all():
        data.append({"id": user.id, "name": user.username})
    return JsonResponse(data, safe=False)

# ✅ For large datasets, use aiterator() + chunked processing
async def export_large_dataset(request):
    data = []
    async for user in User.objects.all().aiterator(chunk_size=500):
        data.append({"id": user.id, "name": user.username})
    return JsonResponse(data, safe=False)

# ✅ Use values() to reduce memory
async def lightweight_export(request):
    data = []
    async for row in User.objects.values("id", "username"):
        data.append(row)
    return JsonResponse(data, safe=False)
```

---

## Middleware and Settings

### Production Security Configuration Checklist

```python
# settings.py - required security settings for production

# ❌ Development defaults must not appear in production
DEBUG = True
SECRET_KEY = "django-insecure-..."
ALLOWED_HOSTS = ["*"]
SECURE_SSL_REDIRECT = False

# ✅ Production security configuration

# --- Basic security ---
DEBUG = False
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]  # Read from environment variable
ALLOWED_HOSTS = ["example.com", "www.example.com"]

# --- HTTPS ---
SECURE_SSL_REDIRECT = True          # Redirect HTTP to HTTPS
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# --- Security headers ---
SECURE_HSTS_SECONDS = 31536000      # 1 year HSTS
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_CONTENT_TYPE_NOSNIFF = True   # X-Content-Type-Options: nosniff
X_FRAME_OPTIONS = "DENY"             # Prevent clickjacking
SECURE_REFERRER_POLICY = "strict-origin-when-cross-origin"

# --- Password validation ---
AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 12}},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
]

# --- Session ---
SESSION_COOKIE_AGE = 3600 * 8  # 8 hours
SESSION_SAVE_EVERY_REQUEST = True
SESSION_EXPIRE_AT_BROWSER_CLOSE = True
```

### Database Connection Security

```python
# settings.py

# ❌ Plaintext password in code
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "USER": "admin",
        "PASSWORD": "hunter2",  # Do not hardcode passwords
        "HOST": "localhost",
        "PORT": "5432",
    }
}

# ✅ Read from environment variables
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("DB_NAME", "mydb"),
        "USER": os.environ.get("DB_USER", "mydb_user"),
        "PASSWORD": os.environ["DB_PASSWORD"],
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
        "OPTIONS": {
            "sslmode": "require",  # Enforce SSL connections
        },
        "CONN_MAX_AGE": 60,  # Persistent connections
    }
}
```

### CORS Configuration

```python
# settings.py (using django-cors-headers)

# ❌ Allow all origins
CORS_ALLOW_ALL_ORIGINS = True

# ✅ Restrict allowed origins
CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "https://app.example.com",
]

# ✅ Production CORS settings
CORS_ALLOW_ALL_ORIGINS = False
CORS_ALLOWED_ORIGINS = os.environ.get("CORS_ORIGINS", "").split(",")
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOW_METHODS = ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
CORS_ALLOW_HEADERS = [
    "authorization",
    "content-type",
    "x-csrftoken",
]
```

### Logging Configuration

```python
# settings.py

# ❌ Default logging configuration (or none at all)
LOGGING = {}

# ✅ Production logging configuration
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{levelname} {asctime} {module} {process:d} {thread:d} {message}",
            "style": "{",
        },
    },
    "handlers": {
        "file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": "/var/log/django/app.log",
            "maxBytes": 10 * 1024 * 1024,  # 10 MB
            "backupCount": 5,
            "formatter": "verbose",
        },
    },
    "loggers": {
        "django": {
            "handlers": ["file"],
            "level": "INFO",
            "propagate": False,
        },
        "myapp": {
            "handlers": ["file"],
            "level": "DEBUG" if DEBUG else "INFO",
            "propagate": False,
        },
    },
}
```

---

## Review Checklist

### Security Review

- [ ] `mark_safe` is not used to render unescaped user input
- [ ] CSRF middleware is enabled, no `@csrf_exempt`
- [ ] Session and CSRF cookies set `Secure`, `HttpOnly`, `SameSite`
- [ ] SQL queries are parameterized (ORM or parameterized `raw()`), no string concatenation
- [ ] File uploads have type and size limits
- [ ] `SECRET_KEY` is read from an environment variable, not in the code repository
- [ ] `DEBUG = False` in production

### HTTPS and Security Headers

- [ ] `SECURE_SSL_REDIRECT = True`
- [ ] `SECURE_HSTS_SECONDS` is set (≥ 31536000)
- [ ] `SECURE_CONTENT_TYPE_NOSNIFF = True`
- [ ] `X_FRAME_OPTIONS` set to `DENY` or `SAMEORIGIN`
- [ ] `ALLOWED_HOSTS` does not contain `"*"`
- [ ] Database connections use SSL

### N+1 Queries

- [ ] ForeignKey relations use `select_related`
- [ ] M2M / reverse relations use `prefetch_related`
- [ ] Related objects are not accessed inside loops
- [ ] Use `count()` instead of `len(queryset)` for counting
- [ ] Use `exists()` instead of `if queryset` for existence checks
- [ ] Large datasets use `only()` / `defer()` or `values()` to reduce queried fields
- [ ] Sliced QuerySets are not iterated repeatedly

### Serializer

- [ ] `fields = "__all__"` is not used on sensitive models
- [ ] Password fields are marked `write_only=True`
- [ ] Field-level and object-level validation exist
- [ ] Nested writes implement `create()` / `update()` or use `read_only=True`
- [ ] Computed and auto-populated fields are in `read_only_fields`
- [ ] Serializer does not include fields that should not be modified

### ViewSet

- [ ] Read-only cases use `ReadOnlyModelViewSet`
- [ ] `get_queryset()` scopes data to the current user
- [ ] `permission_classes` is set
- [ ] `perform_create()` automatically sets owner/author on create
- [ ] Pagination is configured (globally or per ViewSet)
- [ ] Throttling is configured

### Async Views

- [ ] Async views do not call the sync ORM directly (use `aget`/`afilter`/`sync_to_async`)
- [ ] All async calls are awaited
- [ ] `transaction.atomic()` is wrapped with `sync_to_async`
- [ ] Middleware is marked `async_capable = True` to avoid downgrading
- [ ] Large QuerySets use `async for` + `aiterator()`

### Production Configuration

- [ ] `CORS_ALLOWED_ORIGINS` is used instead of `CORS_ALLOW_ALL_ORIGINS = True`
- [ ] Password validators are configured (minimum length, common password check)
- [ ] Session expiry is reasonable (`SESSION_COOKIE_AGE`)
- [ ] Logging uses RotatingFileHandler and does not output to stdout in production
- [ ] Database connections use `CONN_MAX_AGE` for persistent connections
