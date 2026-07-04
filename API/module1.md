# 🚀 MODULE 3: DRF DEEP DIVE (Django REST Framework)
### Senior Staff Engineer Level — Bengali (বাংলা)

---

> **তোমার লক্ষ্য:** DRF এর প্রতিটি কম্পোনেন্ট এতটা গভীরে বোঝা যেন Production এ কোনো সমস্যা দেখলেই তুমি root cause বলতে পারো।

---

## 📋 মডিউল সূচিপত্র

| # | টপিক |
|---|------|
| 3.1 | DRF কী এবং কেন? |
| 3.2 | Serializers — গভীর বিশ্লেষণ |
| 3.3 | Views ও ViewSets |
| 3.4 | Authentication |
| 3.5 | Permissions |
| 3.6 | Middleware Flow |
| 3.7 | Validation |
| 3.8 | Interview Master Section |
| 3.9 | Hands-on Practice |

---

# 3.1 — DRF কী এবং কেন? (What is DRF & Why?)

## কেন শিখবো?

Django একটি web framework যা HTML পেজ বানাতে ভালো। কিন্তু আজকের দুনিয়ায় **Mobile App, React Frontend, Microservices** সবাই **JSON API** চায়। DRF হলো Django এর উপর একটি powerful layer যা REST API বানানো সহজ করে।

**Real-world উদাহরণ:**
- Uber এর mobile app → Backend API কে request করে
- Netflix এর React frontend → API থেকে movie data নেয়
- Amazon এর warehouse system → Internal APIs ব্যবহার করে

## Problem যা DRF Solve করে

```
ছাড়া DRF:
Django View → Manual JSON serialize → HttpResponse → Error prone

DRF দিয়ে:
Django View → Serializer → Response → Automatic validation, auth, permission
```

## Production Failure যদি ঠিকমতো না বোঝো

❌ Serializer ছাড়া raw query → SQL Injection  
❌ Permission ছাড়া endpoint → Data breach  
❌ Pagination ছাড়া list API → Server crash (1M records একসাথে)  

## DRF এর Architecture (Request Lifecycle)

```
HTTP Request
    │
    ▼
┌─────────────────────────────────┐
│         Django URL Router        │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│     DRF APIView / ViewSet        │
│  ┌──────────────────────────┐   │
│  │  1. Authentication Check  │   │
│  │  2. Permission Check      │   │
│  │  3. Throttle Check        │   │
│  │  4. Content Negotiation   │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│         Business Logic           │
│    (Serializer + QuerySet)       │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│         DRF Response             │
│    (JSON / XML / HTML)           │
└─────────────────────────────────┘
```

---

# 3.2 — Serializers (গভীর বিশ্লেষণ)

## Serializer কী?

Serializer হলো **দোভাষী (Translator)**।

```
Python Object ←──── Serializer ────→ JSON
    (Django Model)                   (HTTP Response)
```

**Amazon এর উদাহরণ:** তুমি Amazon এ একটি Product order করলে:
- Database তে আছে Python Object (Product model)
- তোমার Mobile App চায় JSON
- Serializer মাঝখানে translate করে

## Serializer এর ৩ টি কাজ

| কাজ | দিক | বর্ণনা |
|-----|-----|---------|
| Serialization | Python → JSON | DB data কে response এ পাঠানো |
| Deserialization | JSON → Python | Request data কে validate করে save করা |
| Validation | দুই দিক | Data correct কিনা check করা |

## Types of Serializers

### 1. Basic Serializer

```python
# models.py
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'products'
```

```python
# serializers.py
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.Serializer):
    """
    Basic Serializer — সব manually define করতে হয়
    Senior Tip: এটি দিয়ে non-model data serialize করো (external API response, etc.)
    """
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=200)
    price = serializers.DecimalField(max_digits=10, decimal_places=2)
    stock = serializers.IntegerField(min_value=0)
    
    def create(self, validated_data):
        return Product.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        instance.name = validated_data.get('name', instance.name)
        instance.price = validated_data.get('price', instance.price)
        instance.stock = validated_data.get('stock', instance.stock)
        instance.save()
        return instance
```

### 2. ModelSerializer (সবচেয়ে বেশি ব্যবহৃত)

```python
class ProductModelSerializer(serializers.ModelSerializer):
    """
    ModelSerializer — Model থেকে automatically fields নেয়
    Production তে 80% সময় এটি ব্যবহার হয়
    """
    # Computed field (DB তে নেই, কিন্তু response এ দরকার)
    is_in_stock = serializers.SerializerMethodField()
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'stock', 'is_in_stock', 'created_at']
        read_only_fields = ['id', 'created_at']
        extra_kwargs = {
            'price': {'min_value': 0},
            'name': {'error_messages': {'blank': 'Product name খালি হতে পারবে না'}}
        }
    
    def get_is_in_stock(self, obj):
        """SerializerMethodField — custom computed value"""
        return obj.stock > 0
    
    def validate_price(self, value):
        """Field-level validation"""
        if value <= 0:
            raise serializers.ValidationError("Price অবশ্যই ০ এর বেশি হতে হবে")
        return value
    
    def validate(self, data):
        """Object-level validation — একাধিক field এক সাথে check"""
        if data.get('stock', 0) > 0 and data.get('price', 0) == 0:
            raise serializers.ValidationError(
                "Stock আছে কিন্তু price 0 — এটা সম্ভব না!"
            )
        return data
```

### 3. Nested Serializer (Real Production Pattern)

```python
# Amazon এর Order + OrderItems এর মতো
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']

class ProductDetailSerializer(serializers.ModelSerializer):
    """
    Nested Serializer — Related model এর data include করা
    ⚠️ Senior Warning: এটি N+1 problem তৈরি করতে পারে!
    """
    category = CategorySerializer(read_only=True)  # Nested object
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True  # Write এ id নেবে, Read এ full object দেবে
    )
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'category', 'category_id']
```

### 4. Dynamic Fields Serializer (FAANG Level Pattern)

```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """
    GraphQL এর মতো — Client decide করে কোন fields চাই
    
    Usage: /api/products/?fields=id,name,price
    
    Netflix এ ব্যবহার হয় bandwidth বাঁচাতে
    """
    def __init__(self, *args, **kwargs):
        fields = kwargs.pop('fields', None)
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
    
    class Meta:
        model = Product
        fields = '__all__'


# View এ ব্যবহার:
class ProductView(APIView):
    def get(self, request):
        fields = request.query_params.get('fields', '').split(',')
        products = Product.objects.all()
        serializer = DynamicFieldsSerializer(
            products, 
            many=True,
            fields=fields if fields[0] else None
        )
        return Response(serializer.data)
```

## Serializer Internal Flow (গভীর ব্যাখ্যা)

```
POST Request আসলো: {"name": "iPhone", "price": -100}
                    │
                    ▼
        serializer = ProductSerializer(data=request.data)
                    │
                    ▼
        serializer.is_valid()  ← এখানে কী হয়?
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
    Field Validation      Object Validation
    (প্রতিটি field আলাদা)  (সব field একসাথে)
          │                    │
    price: -100 → Error!       │
    "Price > 0 হতে হবে"        │
          │                    │
          └─────────┬──────────┘
                    ▼
        serializer.errors → {'price': ['Price > 0 হতে হবে']}
                    │
                    ▼
        400 Bad Request Response
```

## Senior Engineer মিস্টেক — N+1 Problem

```python
# ❌ BAD: N+1 Problem (100 products = 101 queries!)
class OrderSerializer(serializers.ModelSerializer):
    product_name = serializers.SerializerMethodField()
    
    def get_product_name(self, obj):
        return obj.product.name  # প্রতিটি order এ আলাদা query!
    
    class Meta:
        model = OrderItem
        fields = ['id', 'quantity', 'product_name']

# View:
items = OrderItem.objects.all()  # 1 query
serializer = OrderSerializer(items, many=True)  # আরো 100 query!


# ✅ GOOD: select_related ব্যবহার করো
items = OrderItem.objects.select_related('product').all()  # 1 query, JOIN করে আনে
```

---

# 3.3 — Views এবং ViewSets

## Django View vs DRF View

```python
# Django এর পুরনো পদ্ধতি
from django.http import JsonResponse
from django.views import View

class OldProductView(View):
    def get(self, request):
        products = list(Product.objects.values())  # Manual!
        return JsonResponse({'products': products})
    # Authentication? Manual!
    # Permission? Manual!
    # Validation? Manual!
    # Error handling? Manual!


# DRF এর আধুনিক পদ্ধতি
from rest_framework.views import APIView
from rest_framework.response import Response

class NewProductView(APIView):
    # Authentication? Automatic!
    # Permission? Automatic!
    # Content negotiation? Automatic!
    def get(self, request):
        products = Product.objects.all()
        serializer = ProductSerializer(products, many=True)
        return Response(serializer.data)
```

## View এর ৪টি Level (কম থেকে বেশি Power)

### Level 1: Function-Based View (FBV)

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def product_list(request):
    """
    সহজ কিন্তু কম scalable
    Simple endpoint এর জন্য ভালো
    """
    if request.method == 'GET':
        products = Product.objects.all()
        serializer = ProductSerializer(products, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = ProductSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### Level 2: APIView (Class-Based)

```python
from rest_framework.views import APIView
from rest_framework import status
from rest_framework.response import Response

class ProductListView(APIView):
    """
    OOP পদ্ধতি — Method ভাগ করা সহজ
    Medium complexity এর জন্য ভালো
    """
    permission_classes = [IsAuthenticated]
    
    def get(self, request, format=None):
        """GET /api/products/"""
        products = Product.objects.filter(is_active=True)
        serializer = ProductSerializer(products, many=True)
        return Response({
            'status': 'success',
            'count': products.count(),
            'results': serializer.data
        })
    
    def post(self, request, format=None):
        """POST /api/products/"""
        serializer = ProductSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(created_by=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class ProductDetailView(APIView):
    def get_object(self, pk):
        """Helper method — 404 handle করে"""
        try:
            return Product.objects.get(pk=pk)
        except Product.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        """GET /api/products/{pk}/"""
        product = self.get_object(pk)
        serializer = ProductSerializer(product)
        return Response(serializer.data)
    
    def put(self, request, pk):
        """PUT /api/products/{pk}/ — Full update"""
        product = self.get_object(pk)
        serializer = ProductSerializer(product, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def patch(self, request, pk):
        """PATCH /api/products/{pk}/ — Partial update"""
        product = self.get_object(pk)
        serializer = ProductSerializer(product, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        """DELETE /api/products/{pk}/"""
        product = self.get_object(pk)
        product.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### Level 3: Generic Views (DRF Magic)

```python
from rest_framework.generics import (
    ListCreateAPIView, 
    RetrieveUpdateDestroyAPIView,
    ListAPIView
)

class ProductListCreateView(ListCreateAPIView):
    """
    ✨ DRF এর Magic — GET list + POST create
    CRUD এর ৮০% code automatically লেখা!
    
    Equivalent to:
    - GET /api/products/ → list
    - POST /api/products/ → create
    """
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        """Override করে custom filtering"""
        queryset = super().get_queryset()
        category = self.request.query_params.get('category')
        min_price = self.request.query_params.get('min_price')
        
        if category:
            queryset = queryset.filter(category__name=category)
        if min_price:
            queryset = queryset.filter(price__gte=min_price)
        
        return queryset
    
    def perform_create(self, serializer):
        """Save করার আগে extra data add"""
        serializer.save(
            created_by=self.request.user,
            store=self.request.user.store  # User এর store তে product add
        )


class ProductDetailView(RetrieveUpdateDestroyAPIView):
    """
    GET + PUT + PATCH + DELETE — সব একসাথে
    
    Equivalent to:
    - GET /api/products/{pk}/ → retrieve
    - PUT /api/products/{pk}/ → update
    - PATCH /api/products/{pk}/ → partial_update
    - DELETE /api/products/{pk}/ → destroy
    """
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
```

### Level 4: ViewSets (FAANG Level — সবচেয়ে Powerful)

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ProductViewSet(viewsets.ModelViewSet):
    """
    ViewSet = সব operations একটি class এ
    
    Router automatically URLs তৈরি করে:
    GET    /products/          → list()
    POST   /products/          → create()
    GET    /products/{pk}/     → retrieve()
    PUT    /products/{pk}/     → update()
    PATCH  /products/{pk}/     → partial_update()
    DELETE /products/{pk}/     → destroy()
    
    Amazon, Netflix এই pattern ব্যবহার করে
    """
    queryset = Product.objects.select_related('category').prefetch_related('tags')
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticated]
    
    def get_serializer_class(self):
        """
        SENIOR PATTERN: Action অনুযায়ী Serializer বদলাও
        List এ কম fields, Detail এ বেশি fields
        """
        if self.action == 'list':
            return ProductListSerializer  # Lightweight
        elif self.action == 'retrieve':
            return ProductDetailSerializer  # Full details
        return ProductSerializer
    
    def get_permissions(self):
        """
        Action অনুযায়ী Permission বদলাও
        """
        if self.action in ['list', 'retrieve']:
            permission_classes = [AllowAny]  # Public read
        else:
            permission_classes = [IsAuthenticated, IsAdmin]  # Protected write
        return [permission() for permission in permission_classes]
    
    @action(detail=True, methods=['post'], url_path='apply-discount')
    def apply_discount(self, request, pk=None):
        """
        Custom Action — Extra endpoint যোগ করা
        URL: POST /products/{pk}/apply-discount/
        
        DRF router automatically এই URL তৈরি করে
        """
        product = self.get_object()
        discount = request.data.get('discount_percentage', 0)
        
        if not (0 < discount <= 100):
            return Response(
                {'error': 'Discount ০ থেকে ১০০ এর মধ্যে হতে হবে'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        original_price = product.price
        product.price = product.price * (1 - discount/100)
        product.save()
        
        return Response({
            'original_price': str(original_price),
            'discounted_price': str(product.price),
            'discount_applied': f'{discount}%'
        })
    
    @action(detail=False, methods=['get'], url_path='top-selling')
    def top_selling(self, request):
        """
        URL: GET /products/top-selling/
        detail=False মানে এটি collection level action
        """
        top_products = Product.objects.annotate(
            total_sold=Sum('orderitem__quantity')
        ).order_by('-total_sold')[:10]
        
        serializer = self.get_serializer(top_products, many=True)
        return Response(serializer.data)


# urls.py — Router ব্যবহার
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

## Views এর Performance Comparison

```
┌──────────────────────────────────────────────────────────┐
│  View Type        │ Code Lines │ Flexibility │ Best For   │
├──────────────────────────────────────────────────────────┤
│  FBV              │  Many      │  High       │ Simple     │
│  APIView          │  Medium    │  High       │ Custom     │
│  Generic Views    │  Few       │  Medium     │ Standard   │
│  ViewSet          │  Least     │  Highest    │ Full CRUD  │
└──────────────────────────────────────────────────────────┘
```

---

# 3.4 — Authentication (প্রমাণীকরণ)

## Authentication কী?

**"তুমি কে?" — এই প্রশ্নের উত্তর।**

Netflix এর উদাহরণ:
- তুমি Netflix এ login করলে → Token পেলে
- প্রতিটি API request এ সেই Token পাঠালে
- Netflix জানে তুমি কোন user

## Authentication এর ৩টি Level

```
┌────────────────────────────────────────────────┐
│  Level 1: কে তুমি? (Authentication)            │
│  Level 2: তুমি কী করতে পারো? (Permission)     │
│  Level 3: কতটা করতে পারো? (Throttling)        │
└────────────────────────────────────────────────┘
```

## DRF Authentication Types

### 1. Session Authentication (Traditional)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ]
}
```

```python
# ব্যবহার — Browser based apps এর জন্য
# Django এর login session ব্যবহার করে
# CSRF protection দরকার হয়
# Mobile apps এর জন্য ভালো না
```

### 2. Token Authentication

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ]
}
```

```python
# views.py — Login endpoint
from rest_framework.authtoken.models import Token
from rest_framework.authtoken.views import ObtainAuthToken
from rest_framework.response import Response

class CustomAuthToken(ObtainAuthToken):
    def post(self, request, *args, **kwargs):
        serializer = self.serializer_class(
            data=request.data,
            context={'request': request}
        )
        serializer.is_valid(raise_exception=True)
        user = serializer.validated_data['user']
        
        # Token তৈরি বা get
        token, created = Token.objects.get_or_create(user=user)
        
        return Response({
            'token': token.key,
            'user_id': user.pk,
            'email': user.email,
            'message': 'Login সফল!'
        })
```

```
# Client এর Request:
Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

**⚠️ সমস্যা:** Token কখনো expire হয় না → Security risk!

### 3. JWT Authentication (Production Standard)

```python
# Installation
# pip install djangorestframework-simplejwt

# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ]
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),   # 1 ঘণ্টা
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),       # ৭ দিন
    'ROTATE_REFRESH_TOKENS': True,                     # Refresh করলে নতুন token
    'BLACKLIST_AFTER_ROTATION': True,                  # পুরনো token invalid
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': settings.SECRET_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

```python
# Custom JWT Payload — Extra data add করা
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # JWT payload এ extra data যোগ করো
        token['email'] = user.email
        token['full_name'] = user.get_full_name()
        token['role'] = user.role  # admin, seller, buyer
        token['store_id'] = user.store.id if hasattr(user, 'store') else None
        
        return token
```

```
JWT Structure:
eyJhbGciOiJIUzI1NiJ9  ← Header (Base64)
.
eyJ1c2VyX2lkIjoxfQ   ← Payload (Base64) — user data
.
SflKxwRJSMeKKF2QT4fw  ← Signature (HMAC)

⚠️ JWT decode করা যায়! Sensitive data রেখো না।
রেখো: user_id, role, email
রেখো না: password, credit card, SSN
```

### 4. Custom Authentication

```python
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed

class APIKeyAuthentication(BaseAuthentication):
    """
    Internal Microservices এর জন্য API Key authentication
    Amazon এর internal services এভাবে communicate করে
    """
    
    def authenticate(self, request):
        api_key = request.META.get('HTTP_X_API_KEY')
        
        if not api_key:
            return None  # এই authenticator handle করবে না, পরেরটা try করবে
        
        try:
            service = ServiceAPIKey.objects.select_related('service').get(
                key=api_key,
                is_active=True
            )
        except ServiceAPIKey.DoesNotExist:
            raise AuthenticationFailed('Invalid API Key')
        
        if service.is_expired():
            raise AuthenticationFailed('API Key মেয়াদ শেষ')
        
        # Log করো (Audit trail)
        service.last_used = timezone.now()
        service.save(update_fields=['last_used'])
        
        return (service.service, service)  # (user, auth_object)
    
    def authenticate_header(self, request):
        return 'X-API-Key'
```

---

# 3.5 — Permissions (অনুমতি)

## Permission কী?

**"তুমি এটা করার অনুমতি আছো কি?" — এই প্রশ্নের উত্তর।**

```
Authentication → তুমি কে? (Identity)
Permission    → তুমি কী করতে পারো? (Access Control)
```

**Uber এর উদাহরণ:**
- Driver → নিজের rides দেখতে পারে
- Rider → নিজের trips দেখতে পারে
- Admin → সব দেখতে পারে

## Built-in Permissions

```python
from rest_framework.permissions import (
    AllowAny,          # সবাই access করতে পারে
    IsAuthenticated,   # শুধু logged-in users
    IsAdminUser,       # শুধু Admin (is_staff=True)
    IsAuthenticatedOrReadOnly,  # Anonymous = read only, Login = read+write
)
```

## Custom Permissions (Production Pattern)

```python
from rest_framework.permissions import BasePermission

class IsOwnerOrReadOnly(BasePermission):
    """
    Object-level permission:
    - যে তৈরি করেছে সে edit করতে পারবে
    - বাকি সবাই শুধু read করতে পারবে
    
    Instagram এর post edit permission এরকমই
    """
    
    message = 'শুধুমাত্র owner এই content modify করতে পারবে।'
    
    def has_object_permission(self, request, view, obj):
        # SAFE_METHODS = GET, HEAD, OPTIONS — সবার জন্য
        if request.method in SAFE_METHODS:
            return True
        
        # Unsafe methods — শুধু owner এর জন্য
        return obj.owner == request.user


class IsVerifiedSeller(BasePermission):
    """
    Amazon Seller এর মতো — Verified হলে product add করতে পারবে
    """
    
    message = 'শুধুমাত্র verified sellers এই action করতে পারবে।'
    
    def has_permission(self, request, view):
        return bool(
            request.user and 
            request.user.is_authenticated and
            request.user.is_seller and
            request.user.seller_profile.is_verified
        )


class RoleBasedPermission(BasePermission):
    """
    RBAC (Role Based Access Control)
    Enterprise systems এ এটা ব্যবহার হয়
    """
    
    # ViewSet action → Required roles
    role_map = {
        'list': ['viewer', 'editor', 'admin'],
        'retrieve': ['viewer', 'editor', 'admin'],
        'create': ['editor', 'admin'],
        'update': ['editor', 'admin'],
        'partial_update': ['editor', 'admin'],
        'destroy': ['admin'],
    }
    
    def has_permission(self, request, view):
        action = getattr(view, 'action', None)
        required_roles = self.role_map.get(action, ['admin'])
        user_role = getattr(request.user, 'role', None)
        return user_role in required_roles
```

```python
# ViewSet এ ব্যবহার
class ProductViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsVerifiedSeller]
    
    def get_permissions(self):
        """Dynamic permission — action অনুযায়ী"""
        if self.action in ['list', 'retrieve']:
            return [AllowAny()]
        return [IsAuthenticated(), IsVerifiedSeller()]
```

---

# 3.6 — Middleware Flow (ভেতরের যাত্রা)

## Request কীভাবে DRF এ যায়?

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────┐
│           Django Middleware Stack            │
│                                             │
│  1. SecurityMiddleware                      │
│     (HTTPS, HSTS, XSS protection)           │
│                                             │
│  2. SessionMiddleware                       │
│     (Session management)                    │
│                                             │
│  3. CommonMiddleware                        │
│     (URL normalization)                     │
│                                             │
│  4. CsrfViewMiddleware                      │
│     (CSRF protection)                       │
│                                             │
│  5. AuthenticationMiddleware                │
│     (User attach করে)                       │
│                                             │
│  6. MessageMiddleware                       │
│     (Flash messages)                        │
└─────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────┐
│              DRF APIView                     │
│                                             │
│  initial() method:                          │
│  1. perform_authentication()                │
│     → request.user set করে                 │
│                                             │
│  2. check_permissions()                     │
│     → permission deny করলে 403             │
│                                             │
│  3. check_throttles()                       │
│     → rate limit হলে 429                   │
│                                             │
│  4. Content Negotiation                     │
│     → Response format decide               │
└─────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────┐
│           Handler Method                     │
│     (get, post, put, patch, delete)         │
└─────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────┐
│            Response Rendering               │
│   Renderer → JSON / XML / HTML              │
└─────────────────────────────────────────────┘
```

## Custom Middleware (Production Pattern)

```python
# middleware.py
import time
import logging
import uuid

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware:
    """
    প্রতিটি Request log করা — Production debugging এর জন্য জরুরি
    Netflix এর মতো distributed tracing
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Request শুরু
        request_id = str(uuid.uuid4())[:8]
        request.request_id = request_id
        start_time = time.time()
        
        # Request log
        logger.info(f"[{request_id}] → {request.method} {request.path} "
                   f"| User: {getattr(request.user, 'id', 'anonymous')}")
        
        # View process করো
        response = self.get_response(request)
        
        # Response log
        duration = (time.time() - start_time) * 1000
        logger.info(f"[{request_id}] ← {response.status_code} "
                   f"| {duration:.2f}ms")
        
        # Slow request warning
        if duration > 1000:  # 1 second এর বেশি
            logger.warning(f"🐌 SLOW REQUEST: [{request_id}] "
                          f"{request.path} took {duration:.2f}ms")
        
        # Response এ tracking header যোগ
        response['X-Request-ID'] = request_id
        response['X-Response-Time'] = f"{duration:.2f}ms"
        
        return response


class APIVersionMiddleware:
    """
    URL থেকে API version extract করা
    /api/v1/products/ → request.version = 'v1'
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        path_parts = request.path.split('/')
        if 'api' in path_parts:
            api_index = path_parts.index('api')
            if len(path_parts) > api_index + 1:
                version = path_parts[api_index + 1]
                if version.startswith('v'):
                    request.api_version = version
        
        return self.get_response(request)
```

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'myapp.middleware.RequestLoggingMiddleware',  # Custom Middleware
    'myapp.middleware.APIVersionMiddleware',       # Custom Middleware
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
]
```

---

# 3.7 — Validation (যাচাই)

## Validation এর ৩টি Layer

```
Layer 1: Field Validation    → একটি field এর value check
Layer 2: Object Validation   → একাধিক field এর relation check
Layer 3: View Validation     → Business logic check
```

## Comprehensive Validation Example

```python
from rest_framework import serializers
from django.utils import timezone

class ProductCreateSerializer(serializers.ModelSerializer):
    
    class Meta:
        model = Product
        fields = [
            'name', 'description', 'price', 'sale_price',
            'stock', 'category', 'sku', 'launch_date'
        ]
    
    # ─── Layer 1: Field-Level Validation ──────────────────────────
    
    def validate_name(self, value):
        """Product name validation"""
        if len(value) < 3:
            raise serializers.ValidationError(
                "Product name অন্তত ৩ character হতে হবে"
            )
        
        # Duplicate name check
        if Product.objects.filter(
            name__iexact=value,
            store=self.context['request'].user.store
        ).exists():
            raise serializers.ValidationError(
                f"'{value}' নামে product ইতিমধ্যে আছে"
            )
        
        return value.strip()  # Whitespace trim
    
    def validate_price(self, value):
        """Price validation"""
        if value <= 0:
            raise serializers.ValidationError("Price অবশ্যই positive হতে হবে")
        if value > 1_000_000:
            raise serializers.ValidationError("Price ১০ লাখের বেশি হতে পারবে না")
        return value
    
    def validate_sku(self, value):
        """SKU format validation: ABC-123456"""
        import re
        if not re.match(r'^[A-Z]{3}-\d{6}$', value):
            raise serializers.ValidationError(
                "SKU format হতে হবে: ABC-123456 (3 বড় হাতের letter + 6 সংখ্যা)"
            )
        return value
    
    # ─── Layer 2: Object-Level Validation ──────────────────────────
    
    def validate(self, data):
        """Multiple fields একসাথে validate"""
        
        # Sale price > Regular price হতে পারবে না
        if data.get('sale_price') and data.get('price'):
            if data['sale_price'] >= data['price']:
                raise serializers.ValidationError({
                    'sale_price': 'Sale price, regular price এর চেয়ে কম হতে হবে'
                })
        
        # Launch date past হতে পারবে না
        if data.get('launch_date'):
            if data['launch_date'] < timezone.now().date():
                raise serializers.ValidationError({
                    'launch_date': 'Launch date ভবিষ্যতে হতে হবে'
                })
        
        # Digital product এর stock হয় না
        if data.get('category') and data['category'].is_digital:
            if data.get('stock', 0) != 0:
                data['stock'] = None  # Auto fix বা error দাও
        
        return data
    
    def validate_stock(self, value):
        if value < 0:
            raise serializers.ValidationError("Stock negative হতে পারবে না")
        return value
```

## Custom Validators (Reusable)

```python
from rest_framework.validators import UniqueValidator
from django.core.validators import RegexValidator

# Phone number validator
bd_phone_validator = RegexValidator(
    regex=r'^(?:\+88)?01[3-9]\d{8}$',
    message='বাংলাদেশের valid phone number দিন (01XXXXXXXXX)'
)

class UserProfileSerializer(serializers.ModelSerializer):
    phone = serializers.CharField(validators=[bd_phone_validator])
    email = serializers.EmailField(
        validators=[
            UniqueValidator(
                queryset=User.objects.all(),
                message='এই email দিয়ে ইতিমধ্যে account আছে'
            )
        ]
    )
    
    class Meta:
        model = UserProfile
        fields = ['phone', 'email', 'full_name']
```

---

# 3.8 — 🎯 INTERVIEW MASTER SECTION

## 🔹 Beginner Interview Questions (৩-৫টি)

### Q1: DRF Serializer এর কাজ কী?

**উত্তর (Senior Level):**

Serializer ৩টি কাজ করে:

১. **Serialization:** Python/Django Model object কে JSON এ convert করে response পাঠায়।
২. **Deserialization:** Client এর JSON request data কে Python object এ convert করে।
৩. **Validation:** Data সঠিক কিনা field level ও object level এ check করে।

**Real-world উদাহরণ:** Amazon এ কেউ product order দিলে:
- Request এ JSON আসে (deserialization + validation)
- Database থেকে order details JSON এ পাঠানো হয় (serialization)

**Common Mistake:** অনেকে বলে "JSON convert করে" — এটা অসম্পূর্ণ। Validation এর কথা না বললে marks কাটবে।

---

### Q2: `ModelSerializer` এবং `Serializer` এর পার্থক্য কী?

**উত্তর:**

| বিষয় | Serializer | ModelSerializer |
|-------|-----------|----------------|
| Fields | Manually define করতে হয় | Model থেকে auto-generate |
| create/update | নিজে লিখতে হয় | Automatically implement |
| Validation | নিজে লিখতে হয় | Model validators inherit |
| ব্যবহার | Non-model data, custom | Model-based CRUD |

**কখন Serializer:** External API data, cache data, non-DB operations
**কখন ModelSerializer:** Database CRUD operations

---

### Q3: `is_valid()` call না করলে কী হবে?

**উত্তর:**

`is_valid()` call না করলে `validated_data` পাওয়া যাবে না এবং `save()` call করলে `AssertionError` throw করবে।

```python
# ❌ WRONG
serializer = ProductSerializer(data=request.data)
serializer.save()  # AssertionError: You must call `.is_valid()` before calling `.save()`

# ✅ CORRECT
serializer = ProductSerializer(data=request.data)
if serializer.is_valid():
    serializer.save()
```

**Production best practice:**
```python
serializer.is_valid(raise_exception=True)  # Automatically 400 error পাঠায়
serializer.save()
```

---

### Q4: `read_only_fields` কেন ব্যবহার করবো?

**উত্তর:**

`read_only_fields` দিয়ে এমন fields mark করা হয় যা client পাঠাতে পারবে না কিন্তু response এ দেখতে পাবে।

**Security কারণ:**
- `id` — client নিজে id দিতে পারবে না
- `created_at` — server generate করবে
- `created_by` — server থেকে set হবে, user manipulate করতে পারবে না

```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'total_amount', 'status', 'created_at', 'created_by']
        read_only_fields = ['id', 'created_at', 'created_by']
        # এখন POST request এ id/created_at/created_by পাঠালেও ignore হবে
```

---

## 🔹 Intermediate Questions

### Q5: N+1 Problem কী? DRF তে কীভাবে solve করবে?

**উত্তর:**

**N+1 Problem:** 1টি query তে list আনা, তারপর প্রতিটি item এর জন্য আলাদা query।

**উদাহরণ:** 100 orders, প্রতিটির product name দরকার = 101 queries!

**Solution:**

```python
# ❌ BAD — N+1
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()  # 1 query
    # Serializer এ product access = আরো N queries

# ✅ GOOD — select_related (FK, OneToOne)
queryset = Order.objects.select_related('product', 'customer').all()

# ✅ GOOD — prefetch_related (ManyToMany, reverse FK)
queryset = Order.objects.prefetch_related('items__product').all()

# ✅ PRODUCTION BEST
queryset = Order.objects.select_related(
    'customer', 'store'
).prefetch_related(
    Prefetch('items', queryset=OrderItem.objects.select_related('product'))
).all()
```

**Performance impact:** 101 queries → 2-3 queries। 100ms → 5ms।

---

### Q6: Generic Views কী? কোনটি কখন ব্যবহার করবো?

**উত্তর:**

```python
# Full CRUD mapping:
ListAPIView          → GET list (read only)
CreateAPIView        → POST create
RetrieveAPIView      → GET detail (read only)
UpdateAPIView        → PUT/PATCH update
DestroyAPIView       → DELETE
ListCreateAPIView    → GET + POST
RetrieveUpdateAPIView → GET + PUT + PATCH
RetrieveDestroyAPIView → GET + DELETE
RetrieveUpdateDestroyAPIView → GET + PUT + PATCH + DELETE
```

**কখন ViewSet বনাম Generic Views:**
- **Generic Views:** একটি endpoint এ specific actions, extra customization দরকার
- **ViewSet:** Standard CRUD, Router দিয়ে URLs auto-generate করতে চাই

---

### Q7: ViewSet এ `get_queryset()` override কেন করবো?

**উত্তর:**

Static `queryset` দিয়ে সবার জন্য একই data আসে। `get_queryset()` override করলে:

```python
class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    
    def get_queryset(self):
        """
        প্রতিটি user শুধু নিজের orders দেখবে
        Admin সব দেখবে
        """
        user = self.request.user
        
        if user.is_staff:
            return Order.objects.all()
        
        return Order.objects.filter(
            customer=user
        ).select_related('store').prefetch_related('items')
```

**Security:** এই pattern ছাড়া user অন্যের data দেখতে পারতো!

---

## 🔹 Advanced / Senior Questions

### Q8: JWT vs Session Authentication — Production এ কোনটি বেছে নেবে এবং কেন?

**উত্তর (Senior Level):**

**Session Auth:**
- ✅ Server side control (logout করলে সাথে সাথে কাজ করে)
- ✅ Token revoke করা সহজ
- ❌ Stateful — Server memory/DB তে session রাখতে হয়
- ❌ Horizontal scaling এ সমস্যা (Load balancer এ sticky session দরকার)
- ❌ Mobile apps/Microservices এর জন্য ভালো না

**JWT:**
- ✅ Stateless — Server কিছু store করে না
- ✅ Microservices এ সহজে share করা যায়
- ✅ Mobile apps, SPAs এর জন্য perfect
- ❌ Token revoke করা কঠিন (blacklist রাখতে হয়)
- ❌ Payload বড় হলে প্রতি request এ বেশি data যায়

**Production Decision:**
```
Monolith + Browser App → Session Auth
Microservices + Mobile + SPA → JWT

Netflix/Uber → JWT (Stateless, Mobile-first)
Traditional Banking → Session (Control important)
```

---

### Q9: Custom Permission লিখতে বলো যেখানে শুধু Business Owner তার নিজের Business এর data access করতে পারবে।

**উত্তর:**

```python
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsBusinessOwner(BasePermission):
    """
    Object-level permission:
    - Business owner নিজের business manage করতে পারবে
    - Staff read করতে পারবে
    - অন্য কেউ access করতে পারবে না
    """
    
    message = 'এই business এর owner ছাড়া কেউ এটি modify করতে পারবে না।'
    
    def has_permission(self, request, view):
        # Anonymous user কে কোনো access নেই
        return request.user and request.user.is_authenticated
    
    def has_object_permission(self, request, view, obj):
        # Admin সব করতে পারে
        if request.user.is_superuser:
            return True
        
        # Staff read করতে পারে
        if request.method in SAFE_METHODS and request.user.is_staff:
            return True
        
        # Business এর owner check
        # obj = Business model instance
        return obj.owner == request.user

# ViewSet এ ব্যবহার:
class BusinessViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsBusinessOwner]
    
    def get_queryset(self):
        if self.request.user.is_staff:
            return Business.objects.all()
        return Business.objects.filter(owner=self.request.user)
```

---

### Q10: Serializer তে Context কীভাবে ব্যবহার করবে?

**উত্তর:**

Context দিয়ে Serializer এ request, view, extra data পাঠানো যায়:

```python
class ProductSerializer(serializers.ModelSerializer):
    
    price_display = serializers.SerializerMethodField()
    
    def get_price_display(self, obj):
        """Request এর currency অনুযায়ী price format করো"""
        request = self.context.get('request')
        currency = getattr(request, 'currency', 'BDT')
        
        if currency == 'USD':
            return f"${obj.price / 110:.2f}"  # BDT to USD
        return f"৳{obj.price}"
    
    def validate_name(self, value):
        """Context থেকে user নিয়ে store-specific validation"""
        request = self.context.get('request')
        store = request.user.store
        
        if Product.objects.filter(name=value, store=store).exists():
            raise serializers.ValidationError("এই নামে product আছে")
        return value

# View এ context pass করা
class ProductView(APIView):
    def get(self, request):
        products = Product.objects.all()
        serializer = ProductSerializer(
            products,
            many=True,
            context={'request': request}  # Context pass করা হচ্ছে
        )
        return Response(serializer.data)

# Generic View → Automatically context pass করে
# ViewSet → Automatically context pass করে
```

---

## 🔹 FAANG-Level System Design Questions

### Q11: Instagram এর Feed API Design করো যা ১ কোটি user handle করবে।

**উত্তর (System Design):**

```
Requirements:
- User feed (followings এর posts)
- Like, Comment
- Infinite scroll (pagination)
- Real-time updates

Architecture:

┌──────────────────────────────────────────────────────┐
│                    Load Balancer                      │
└──────────────────────────────────────────────────────┘
         │              │              │
         ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  API Server  │ │  API Server  │ │  API Server  │
│  (DRF)      │ │  (DRF)      │ │  (DRF)      │
└──────────────┘ └──────────────┘ └──────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│              Cache Layer (Redis Cluster)              │
│  user_feed:{user_id} → Pre-computed feed (TTL: 5min) │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│           Message Queue (Celery + Redis)              │
│  New post → Async fan-out → Followers' feed update   │
└──────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│              Database (PostgreSQL Sharded)            │
│  Posts table sharded by user_id                      │
└──────────────────────────────────────────────────────┘

API Design:
GET /api/v1/feed/?cursor=eyJpZCI6MTAwfQ&limit=20
→ Cursor-based pagination (offset নয়!)

Response:
{
    "results": [...],
    "next_cursor": "eyJpZCI6ODB9",
    "has_more": true
}
```

**DRF Implementation:**

```python
from rest_framework.pagination import CursorPagination

class FeedCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'
    cursor_query_param = 'cursor'

class FeedViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = PostSerializer
    pagination_class = FeedCursorPagination
    
    def get_queryset(self):
        user = self.request.user
        
        # Cache check
        cache_key = f'feed:{user.id}'
        cached = cache.get(cache_key)
        if cached:
            return cached
        
        # Database query
        following_ids = user.following.values_list('id', flat=True)
        queryset = Post.objects.filter(
            author_id__in=following_ids
        ).select_related('author').prefetch_related(
            'media', 'likes_count'
        ).order_by('-created_at')
        
        # Cache store (5 minutes)
        cache.set(cache_key, queryset, 300)
        
        return queryset
```

---

### Q12: E-Commerce API এ Concurrent Order Problem কীভাবে handle করবে?

**উদাহরণ:** iPhone Launch এ ১০০০ user একসাথে last 1টি iPhone কিনতে চায়।

**উত্তর:**

```python
from django.db import transaction
from django.db.models import F

class OrderCreateView(APIView):
    
    @transaction.atomic
    def post(self, request):
        product_id = request.data.get('product_id')
        quantity = request.data.get('quantity', 1)
        
        # Pessimistic Locking — Row lock করো
        try:
            product = Product.objects.select_for_update(
                nowait=True  # Lock না পেলে exception (না হলে wait)
            ).get(id=product_id)
        except Product.DoesNotExist:
            return Response({'error': 'Product পাওয়া যায়নি'}, status=404)
        except DatabaseError:
            return Response(
                {'error': 'Product busy, একটু পরে try করুন'}, 
                status=409
            )
        
        # Stock check
        if product.stock < quantity:
            return Response(
                {'error': f'মাত্র {product.stock}টি available'},
                status=400
            )
        
        # Atomic stock decrease + Order create
        Product.objects.filter(id=product_id).update(
            stock=F('stock') - quantity  # Race condition safe!
        )
        
        order = Order.objects.create(
            customer=request.user,
            product=product,
            quantity=quantity,
            total_amount=product.price * quantity
        )
        
        return Response(OrderSerializer(order).data, status=201)
```

**Trade-offs:**
- `select_for_update` → Database level lock → Performance কমে কিন্তু Consistency নিশ্চিত
- High traffic এ → Redis distributed lock বা Optimistic locking ব্যবহার করো

---

## 🔹 Scenario-Based Questions

### Q13: Production এ হঠাৎ API response slow হয়ে গেছে। কীভাবে debug করবে?

**উত্তর (Senior Level Approach):**

```
Step 1: Metrics দেখো
- Response time spike কখন থেকে?
- কোন endpoint?
- DB query time? Cache hit rate?

Step 2: Django Debug Toolbar (Dev) / Django Silk (Prod)
pip install django-silk

# settings.py
MIDDLEWARE = ['silk.middleware.SilkyMiddleware', ...]
# /silk/ → সব request এর query count, time দেখাবে

Step 3: Slow Query Log
# settings.py
LOGGING = {
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',  # সব SQL log করো
        }
    }
}

Step 4: সাধারণ কারণ গুলো:
1. N+1 Problem → select_related/prefetch_related যোগ করো
2. Missing Index → EXPLAIN ANALYZE চালাও
3. Cache miss → Redis TTL check করো
4. Serializer heavy → only() দিয়ে fields কমাও
5. Pagination নেই → হাজার records একসাথে load হচ্ছে

Step 5: Quick Fix
queryset = Product.objects.only(
    'id', 'name', 'price'  # শুধু দরকারি columns
).filter(is_active=True)[:100]
```

---

# 3.9 — Hands-on Practice

## 🛠 Exercise 1: Complete Product API বানাও

```
Requirements:
✅ ProductSerializer with full validation
✅ CRUD ViewSet
✅ JWT Authentication
✅ Permission: Owner only can edit
✅ Filtering by category, price range
✅ Pagination (cursor-based)
✅ N+1 problem solve করা
```

## 🛠 Exercise 2: Broken API Debug করো

```python
# এই code এ কী সমস্যা?
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()  # Bug 1
    serializer_class = OrderSerializer
    
    def create(self, request):
        serializer = OrderSerializer(data=request.data)
        serializer.save()  # Bug 2
        return Response(serializer.data)
    
    def list(self, request):
        orders = self.queryset
        return Response(OrderSerializer(orders).data)  # Bug 3

# উত্তর:
# Bug 1: সব user এর orders দেখা যাবে (Authorization issue)
# Bug 2: is_valid() call নেই
# Bug 3: many=True নেই, context নেই
```

## 🛠 Exercise 3: Spring Boot Equivalent

```java
// Spring Boot এ DRF ViewSet এর equivalent
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping
    public ResponseEntity<Page<ProductDTO>> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String category) {
        
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Page<ProductDTO> products = productService.findAll(category, pageable);
        return ResponseEntity.ok(products);
    }
    
    @PostMapping
    @PreAuthorize("hasRole('SELLER')")  // DRF IsVerifiedSeller এর equivalent
    public ResponseEntity<ProductDTO> createProduct(
            @Valid @RequestBody CreateProductRequest request,  // @Valid = DRF is_valid()
            @AuthenticationPrincipal UserDetails user) {
        
        ProductDTO product = productService.create(request, user.getUsername());
        return ResponseEntity.status(HttpStatus.CREATED).body(product);
    }
}
```

---

# 📋 MODULE 3 Summary Sheet

## Key Takeaways

| Component | মূল কাজ | Senior Tip |
|-----------|---------|-----------|
| Serializer | Data validate + Transform | Context ব্যবহার করো, N+1 সাবধান |
| ModelSerializer | Model CRUD | `read_only_fields` সব সময় set করো |
| APIView | HTTP method handle | `get_object_or_404` ব্যবহার করো |
| ViewSet | Full CRUD + Router | `get_queryset()` override করো |
| Authentication | User identify | JWT for stateless, Session for stateful |
| Permission | Access control | Object-level permission ভুলো না |
| Middleware | Cross-cutting concerns | Logging + Request ID সব সময় রাখো |
| Validation | Data integrity | Field + Object + Business logic আলাদা |

## Senior Engineer Checklist ✅

- [ ] সব endpoints এ authentication আছে
- [ ] `read_only_fields` set করা আছে
- [ ] `select_related`/`prefetch_related` দিয়ে N+1 solve করা
- [ ] Pagination implement করা
- [ ] Object-level permission check করা
- [ ] Custom error messages বাংলা/ইংরেজি তে
- [ ] `is_valid(raise_exception=True)` ব্যবহার
- [ ] Request logging middleware আছে
- [ ] JWT token expiry সঠিকভাবে set
- [ ] Sensitive data JWT payload এ নেই

## Production Best Practices

```python
# settings.py — Production DRF Config
REST_FRAMEWORK = {
    # Authentication
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    
    # Permission — Default সব authenticated
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    
    # Pagination — Default page size
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.CursorPagination',
    'PAGE_SIZE': 20,
    
    # Throttling — Rate limiting
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
    
    # Renderer — Production এ JSON only
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
    
    # Exception handling
    'EXCEPTION_HANDLER': 'myapp.exceptions.custom_exception_handler',
}
```

---

> **পরবর্তী:** MODULE 4 — Performance Optimization (N+1, Caching, Query Optimization, Pagination Strategies)

---
*তৈরি করা হয়েছে Senior Staff Engineer Learning Path অনুযায়ী — FAANG Interview Preparation*
