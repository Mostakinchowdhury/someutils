# 📒 Full Payment Integration (Next.js + DRF + PostgreSQL + Stripe + SSLCommerz)

---

## 1️⃣ Project Setup

### Django + DRF + PostgreSQL

```bash
pip install django djangorestframework psycopg2-binary stripe sslcommerz-lib
```

### PostgreSQL settings.py

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "paymentdb",
        "USER": "postgres",
        "PASSWORD": "1234",
        "HOST": "localhost",
        "PORT": "5432",
    }
}
# settings.py

ALLOWED_HOSTS = ["127.0.0.1", "localhost", "buford-presurgical-anders.ngrok-free.dev"] #webhook er jonno live backend link lagbe ngrok  dia kora hoice

CORS_ALLOW_ALL_ORIGINS = True  # সব origin allow করবে
# অথবা চাইলে safe রাখতে পারো
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://127.0.0.1:3000",
    "https://buford-presurgical-anders.ngrok-free.dev",
]

CSRF_TRUSTED_ORIGINS = [
    "http://localhost:3000",
    "http://127.0.0.1:3000",
    "https://buford-presurgical-anders.ngrok-free.dev",
]

```

---

## 2️⃣ Django Models (Order + Payment)

```python
# models.py
from django.db import models

class Order(models.Model):
    STATUS_CHOICES = [
        ("PENDING", "Pending"),
        ("PAID", "Paid"),
        ("FAILED", "Failed"),
        ("CANCELLED", "Cancelled"),
    ]
    product_name = models.CharField(max_length=200)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    currency = models.CharField(max_length=10, default="BDT")
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="PENDING")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

---

## 3️⃣ Serializer

```python
# serializers.py
from rest_framework import serializers
from .models import Order

class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = "__all__"
```

---

## 4️⃣ Views (Stripe + SSLCommerz)

### Stripe Checkout View

```python
# views.py
import stripe
from django.conf import settings
from django.shortcuts import get_object_or_404
from rest_framework.response import Response
from rest_framework.views import APIView
from .models import Order

stripe.api_key = settings.STRIPE_SECRET_KEY

class StripeCheckoutView(APIView):
    def post(self, request, pk):
        order = get_object_or_404(Order, pk=pk)
        checkout_session = stripe.checkout.Session.create(
            payment_method_types=["card"],
            line_items=[{
                "price_data": {
                    "currency": "usd",
                    "product_data": {"name": order.product_name},
                    "unit_amount": int(order.amount * 100),  # cents
                },
                "quantity": 1,
            }],
            mode="payment",
            success_url="http://localhost:3000/success",
            cancel_url="http://localhost:3000/cancel",
            metadata={"order_id": str(order.id)},
        )
        return Response({"url": checkout_session.url})
```

### SSLCommerz Checkout View

```python
from sslcommerz_lib import SSLCOMMERZ

class SSLCommerzCheckoutView(APIView):
    def post(self, request, pk):
        order = get_object_or_404(Order, pk=pk)
        settings_dict = {
            'store_id': settings.SSLCommerz_STORE_ID,
            'store_pass': settings.SSLCommerz_STORE_PASS,
            'issandbox': True
        }
        sslcz = SSLCOMMERZ(settings_dict)

        post_body = {
            'total_amount': str(order.amount),
            'currency': order.currency,
            'tran_id': str(order.id),
            'success_url': "http://127.0.0.1:8000/api/payment/success/",
            'fail_url': "http://127.0.0.1:8000/api/payment/fail/",
            'cancel_url': "http://127.0.0.1:8000/api/payment/cancel/",
            'cus_name': "Test User",
            'cus_email': "test@test.com",
            'cus_phone': "01700000000",
            'cus_add1': "Dhaka",
            'cus_country': "Bangladesh",
            'product_name': "demo",
            'cus_city':"dhaka",
            'product_category': "General",
            'product_profile': "general",
            'shipping_method':'NO'
        }

        response = sslcz.createSession(post_body)
        return Response({"url": response["GatewayPageURL"]})
```

---

## 5️⃣ URL Routing

```python
# urls.py
from django.urls import path
from .views import StripeCheckoutView, SSLCommerzCheckoutView

urlpatterns = [
    path("stripe-checkout/<int:pk>/", StripeCheckoutView.as_view()),
    path("sslcommerz-checkout/<int:pk>/", SSLCommerzCheckoutView.as_view()),
]
```

---

## 6️⃣ Webhook (Order Status Update)

### Stripe Webhook

```python
from django.http import HttpResponse
# stripe webhook
from django.views.decorators.csrf import csrf_exempt
from django.http import HttpResponse, HttpResponseBadRequest

@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    event = None
    endpoint_secret=settings.STRIPE_WEBHOOK_SECRET
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, endpoint_secret
        )
    except ValueError as e:
        # Invalid payload
        print('Error parsing payload: {}'.format(str(e)))
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError as e:
        # Invalid signature
        print('Error verifying webhook signature: {}'.format(str(e)))
        return HttpResponse(status=400)
    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]
        order_id = session["metadata"]["order_id"]
        order = Order.objects.get(id=order_id)
        order.status = "PAIDANDPROCESSING"
        order.save()

    return HttpResponse(status=200)
```

### SSLCommerz Webhook

👉 SSLCommerz সাধারণত success/fail URL এ redirect করে। সেখানে `tran_id` দিয়ে order update করতে হবে।

```python
def payment_success(request):
    tran_id = request.GET.get("tran_id")
    order = Order.objects.get(id=tran_id)
    order.status = "PAID"
    order.save()
    return HttpResponse("Payment Success")
```

---

## 7️⃣ Next.js Frontend (Testing)

### Fetch Order & Pay Button

```tsx
"use client"
import { useState } from "react"

export default function PayPage() {
  const [loading, setLoading] = useState(false)

  const handleStripe = async () => {
    setLoading(true)
    const res = await fetch("http://127.0.0.1:8000/api/stripe-checkout/1/", {method:"POST"})
    const data = await res.json()
    window.location.href = data.url
  }

  const handleSSLCommerz = async () => {
    setLoading(true)
    const res = await fetch("http://127.0.0.1:8000/api/sslcommerz-checkout/1/", {method:"POST"})
    const data = await res.json()
    window.location.href = data.url
  }

  return (
    <div>
      <h1>Payment Test</h1>
      <button onClick={handleStripe} disabled={loading}>Pay with Stripe</button>
      <button onClick={handleSSLCommerz} disabled={loading}>Pay with SSLCommerz</button>
    </div>
  )
}
```

---

## 8️⃣ কিভাবে Sandbox/Test Account খোলা যাবে?

### Stripe

1. [Stripe Dashboard](https://dashboard.stripe.com/register) এ account খুলো।
2. Sandbox/Test mode এ automatically `$ test keys` পাওয়া যাবে।
3. Use →

   * Publishable Key → Frontend
   * Secret Key → Backend

👉 Test Card:

* `4242 4242 4242 4242` -> সবসময় confirm (success) হবে।
* Expiry: যেকোনো future date
* CVC: 123
* 4000 0000 0000 0002 → সবসময় declined (fail) হবে।

* 4000 0000 0000 9995 → insufficient funds দেখাবে।

* 4000 0000 0000 9987 → lost card হিসেবে fail হবে।

* 4000 0000 0000 9979 → stolen card হিসেবে fail হবে।

* 4000 0000 0000 0069 → expired card error দেবে।

---

### SSLCommerz

1. [SSLCommerz Sandbox](https://developer.sslcommerz.com/) এ account খুলো।
2. Sandbox Mode এ `Store ID` & `Store Password` পাওয়া যাবে।
3. Test Payment এর জন্য →

   * Card: `4111111111111111`
   * Expiry: 12/26
   * CVV: 123

---

## 9️⃣ Webhook কেন ব্যবহার করব?

* **Problem:** User browser বন্ধ করে দিলে success/fail URL এ নাও আসতে পারে।
* **Solution:** Webhook server-to-server event পাঠায়।
* **Use Case:**

  * Payment confirm হলে **order status update**
  * **invoice generate**
  * **email send**

👉 Stripe → `checkout.session.completed` event শুনে order update করো।
👉 SSLCommerz → success/fail URL এর পাশাপাশি webhook endpoint ও দেওয়া যায় (IPN – Instant Payment Notification)।

---

## 🔟 Invoice Generation

Payment success হলে:

1. `Order` → status = "PAID"
2. `Invoice` Model create করো (Order details + Payment ID রাখো)।
3. User কে email এ PDF Invoice পাঠানো যায় (e.g. `reportlab` দিয়ে)।

---

## ✅ Recap

* Backend: DRF + PostgreSQL এ Order Model বানালাম।
* Stripe & SSLCommerz দুইটা Gateway আলাদা View দিয়ে integrate করলাম।
* Next.js frontend এ checkout button বানালাম।
* Webhook দিয়ে real-time payment status update করলাম।
* Sandbox account (Stripe + SSLCommerz) এ free তে test করা যাবে।

---


---

# 📖 Full Authentication Handnote

**Tech Stack:** Django REST Framework (DRF), Next.js, Axios, JWT

---

## 1️⃣ Backend Setup (Django + DRF)

### 1.1 Install Packages

```bash
pip install djangorestframework djangorestframework-simplejwt django-cors-headers django-environ
```

---

### 1.2 Django Settings (`settings.py`)

```python

from pathlib import Path
from pathlib import Path
import environ
import os

from django.conf.global_settings import CSRF_TRUSTED_ORIGINS

BASE_DIR = Path(__file__).resolve().parent.parent

env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")
# Installed apps
INSTALLED_APPS = [
    ...,
    'rest_framework',
    'corsheaders',
    'rest_framework_simplejwt.token_blacklist',  # for logout/blacklist
]

# Middleware
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    ...
]

# CORS
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",  # frontend URL
]

# DRF + JWT Settings
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}

# Simple JWT Settings (optional custom)
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=15),
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_USER_CLASS': 'rest_framework_simplejwt.models.TokenUser',
}

# Email Settings (for password reset)
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
```

**.env file**:

```env
DB_NAME=food_resturant_db
DB_USER=food_user
DB_PASSWORD=nid1980312092
DB_HOST=127.0.0.1
DB_PORT=5432
EMAIL_HOST_USER=chowdhurymostakin02@gmail.com
EMAIL_HOST_PASSWORD=nvzc tmua bybr rici
FRONTEND_URL=https://food-resturant-front-end-uvgw.vercel.app/
STRIPE_SECRET_KEY=sk_test_51S6unv1t12GV5u99kINdeQFpSJkYTFIlqCvkTAqqucbDtkdqZpqvHMpByVn3PcWrWzyZTaxEn1F3KyFDRGn18LAh00x4LEYRV9
SSLCOMMERZ_STORE_ID=demon68d3e66a8694d
SSLCOMMERZ_STORE_PASS=demon68d3e66a8694d@ssl
STRIPE_WEBHOOK_SECRET=whsec_qrAXbUYqmeg0g0Z1g1ERkupi3EdTQaz1
STRIPE_WEBHOOK_ID=we_1SBG781t12GV5u99ZEHwFu6J

```

> `django-environ` ব্যবহার করে `.env` থেকে load করা যাবে।

---

### 1.3 Models (`users/models.py`)

```python
from django.contrib.auth.models import AbstractUser
from django.db import models
from .manager import myUserManager,

class CustomUser(AbstractBaseUser,PermissionsMixin):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)
    email = models.EmailField(unique=True)
    is_staff = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True)
    is_verified = models.BooleanField(default=False)
    otp_code = models.CharField(max_length=6, blank=True, null=True)
    otp_expiry = models.DateTimeField(blank=True, null=True)
    USERNAME_FIELD="email"
    EMAIL_FIELD="email"
    REQUIRED_FIELDS=["first_name","last_name"]
    objects=myUserManager()
    def __str__(self):
       return self.email
```

> settings.py এ:

```python
"""
Django settings for food_resturant_back_end project.

Generated by 'django-admin startproject' using Django 5.2.5.

For more information on this file, see
https://docs.djangoproject.com/en/5.2/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/5.2/ref/settings/
"""



from pathlib import Path
from pathlib import Path
import environ
import os

from django.conf.global_settings import CSRF_TRUSTED_ORIGINS

BASE_DIR = Path(__file__).resolve().parent.parent

env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")


# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent


# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = 'django-insecure-ot#4#i@^-krs*)9+c(20zno#vhh2+ky8%m5faq%@^udc_oyz&@'

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True

# Application definition

INSTALLED_APPS = [
    "api",
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    "corsheaders",
    'rest_framework_simplejwt.token_blacklist',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    "corsheaders.middleware.CorsMiddleware",
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    "django.middleware.common.CommonMiddleware",
]

CORS_ALLOW_METHODS = (
    "DELETE",
    "GET",
    "OPTIONS",
    "PATCH",
    "POST",
    "PUT",
)

ROOT_URLCONF = 'food_resturant_back_end.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'food_resturant_back_end.wsgi.application'


# Database
# https://docs.djangoproject.com/en/5.2/ref/settings/#databases

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": env("DB_NAME"),
        "USER": env("DB_USER"),
        "PASSWORD": env("DB_PASSWORD"),
        "HOST": env("DB_HOST", default="127.0.0.1"),
        "PORT": env("DB_PORT", default="5432"),
        "CONN_MAX_AGE": env.int("CONN_MAX_AGE", default=60),
    }
}


AUTH_USER_MODEL = 'api.CustomUser'
AUTHENTICATION_BACKENDS = ['api.backends.EmailBackend']



# Password validation
# https://docs.djangoproject.com/en/5.2/ref/settings/#auth-password-validators

AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]


# Internationalization
# https://docs.djangoproject.com/en/5.2/topics/i18n/

LANGUAGE_CODE = 'en-us'

TIME_ZONE = 'UTC'

USE_I18N = True

USE_TZ = True


# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/5.2/howto/static-files/

STATIC_URL = 'static/'

# Default primary key field type
# https://docs.djangoproject.com/en/5.2/ref/settings/#default-auto-field

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'



EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')
EMAIL_HOST_USER = env('EMAIL_HOST_USER')


TIME_ZONE = 'Asia/Dhaka'
USE_TZ = True
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')


from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=15),
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_USER_CLASS': 'rest_framework_simplejwt.models.TokenUser',
}


REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',],
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],

}

FRONTEND_URL= env("FRONTEND_URL", default="http://localhost:3000")




# SSLCOMMERZ_STORE_ID variables
SSLCOMMERZ_STORE_ID=env("SSLCOMMERZ_STORE_ID", default=None)
SSLCOMMERZ_STORE_PASS=env("SSLCOMMERZ_STORE_PASS", default=None)

STRIPE_SECRET_KEY=env("STRIPE_SECRET_KEY")
# webhook keys
STRIPE_WEBHOOK_SECRET=env("STRIPE_WEBHOOK_SECRET", default=None)
STRIPE_WEBHOOK_ID=env("STRIPE_WEBHOOK_ID", default=None)

# settings.py

ALLOWED_HOSTS = ["127.0.0.1", "localhost", "buford-presurgical-anders.ngrok-free.dev","https://food-resturant-front-end-uvgw.vercel.app","https://food-resturant-back-end.onrender.com","food-resturant-back-end.onrender.com"]

CORS_ALLOW_ALL_ORIGINS = True  # সব origin allow করবে
# অথবা চাইলে safe রাখতে পারো
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://127.0.0.1:3000",
    "https://buford-presurgical-anders.ngrok-free.dev",
    "https://food-resturant-front-end-uvgw.vercel.app",
    "https://food-resturant-back-end.onrender.com"
]

CSRF_TRUSTED_ORIGINS = [
    "http://localhost:3000",
    "http://127.0.0.1:3000",
    "https://buford-presurgical-anders.ngrok-free.dev",
    "https://food-resturant-front-end-uvgw.vercel.app",
    "https://food-resturant-back-end.onrender.com"
]


STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

```

---
### manager.py
```python 
from django.contrib.auth.base_user import BaseUserManager
from django.db.models import Manager
# custom manager for cartitem,cart model include get_create method

# import base manager


class myUserManager(BaseUserManager):
    def create_user(self,email,password=None,**extra_field):
        if not email:
            raise ValueError("Email is a requred field")
        email=self.normalize_email(email)
        user=self.model(email=email,password=password,**extra_field)
        user.set_password(password)
        user.save(using=self._db)
        return user
    def create_superuser(self,email,password=None,**extra_field):
        extra_field.setdefault("is_superuser",True)
        extra_field.setdefault("is_staff",True)
        extra_field.setdefault("is_active",True)
        extra_field.setdefault("is_verified",True)
        return self.create_user(email,password,**extra_field)

    def active_user(self):
        return self.get_queryset().filter(is_active=True)


# create custom manager for cart and cartitem model
class CartitemManager(Manager):
    def checked_items(self):
        return self.get_queryset().filter(ischeaked=True)

```
### 1.4 Serializers (`users/serializers.py`)

```python
from .models import Profile, Setting, Tag, Order, Product, Category, Cart, CartItem, ProductReview, Address, PromoUsage
from rest_framework import serializers
from django.contrib.auth import authenticate
from django.contrib.auth import get_user_model
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import AuthenticationFailed

from .utils import password_reset_email


def get_tokens_for_user(user):
    if not user.is_active:
      raise AuthenticationFailed("User is not active")

    refresh = RefreshToken.for_user(user)
    return {
        'refresh': str(refresh),
        'access': str(refresh.access_token),
    }

User = get_user_model()

class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, required=True, style={'input_type': 'password'})
    password2 = serializers.CharField(write_only=True, required=True, style={'input_type': 'password'})
    class Meta:
        model = User
        fields = ['id', 'email', 'first_name', 'last_name','password','password2','is_staff','is_superuser']
        extra_kwargs = {
            'first_name': {'required': True},
            'last_name': {'required': True},
        }
    def validate_password(self, value):
        if not value.strip():
            raise serializers.ValidationError("This field cannot be blank.")
        if len(value)<8:
            raise serializers.ValidationError("Password must be at least 8 characters long.")
        return value.strip()
    def validate_password2(self, value):
        return value.strip()
    def validate(self, attrs):
        if attrs['password'] != attrs['password2']:
            raise serializers.ValidationError({"password": "Password fields didn't match."})
        return attrs


    def create(self, validated_data):
        validated_data.pop('password2')
        user = User.objects.create_user(**validated_data)
        return "We sent an OTP to your email. Please check your inbox and verify your account."


# serializer for user login
class loginserializer(serializers.Serializer):
  password=serializers.CharField(min_length=8)
  email=serializers.EmailField()
  def validate(self, attrs):
     user=authenticate(email=attrs['email'],password=attrs['password'])
     if user is not None:
       attrs['token']=get_tokens_for_user(user)
     else:
       raise serializers.ValidationError("Your information not match with any record try again")
     return attrs


# password change serializer
class ChangePasswordSerializer(serializers.Serializer):
    old_password = serializers.CharField(write_only=True)
    new_password = serializers.CharField(write_only=True)
    confirm_password = serializers.CharField(write_only=True)

    def validate_new_password(self, value):
        if len(value) < 8:
            raise serializers.ValidationError("Password must be at least 8 characters long")
        return value

    def validate(self, attrs):
        user = self.context.get('user')
        old_password = attrs.get('old_password')
        new_password = attrs.get('new_password')
        confirm_password = attrs.get('confirm_password')

        if new_password != confirm_password:
            raise serializers.ValidationError("New password and confirm password do not match")

        if not user.check_password(old_password):
            raise serializers.ValidationError("Old password is incorrect")

        return attrs

    def save(self, **kwargs):
        user = self.context.get('user')
        user.set_password(self.validated_data['new_password'])
        user.save()
        return user



from django.contrib.auth.tokens import PasswordResetTokenGenerator
from django.core.mail import send_mail
from django.conf import settings
from django.utils.http import urlsafe_base64_encode, urlsafe_base64_decode
from django.utils.encoding import smart_str, force_bytes, DjangoUnicodeDecodeError
from .utils import password_reset_email
class ResetPasswordgenaretSerializer(serializers.Serializer):
    email = serializers.EmailField()

    def validate_email(self, value):
        if not User.objects.filter(email=value).exists():
            raise serializers.ValidationError("User with this email does not exist")
        return value
    def save(self, **kwargs):
        email = self.validated_data['email']
        user = User.objects.get(email=email)
        # Here you would typically send an email with a reset link or token
        # For simplicity, we'll just return the user object
        token = PasswordResetTokenGenerator().make_token(user)
        uidb64 = urlsafe_base64_encode(force_bytes(user.id))
        reset_link = f"{settings.FRONTEND_URL}/reset-password/{uidb64}/{token}/"
        send_mail(
                subject="Reset your password",
                message="Click the link to reset your password",
                html_message=password_reset_email(reset_link),
                from_email=f"OrderUK <{settings.EMAIL_HOST_USER}>",
                recipient_list=[email],
            )
        return "we have sent you a link to reset your password in your email account"


class setResetPasswordSerializer(serializers.Serializer):
    password = serializers.CharField(write_only=True)
    confirm_password = serializers.CharField(write_only=True)
    token = serializers.CharField(write_only=True)
    uidb64 = serializers.CharField(write_only=True)

    def validate(self, attrs):
        try:
            password = attrs.get('password')
            confirm_password = attrs.get('confirm_password')
            token = attrs.get('token')
            uidb64 = attrs.get('uidb64')

            if password != confirm_password:
                raise serializers.ValidationError({'confirm_password':"Password and confirm password do not match"})

            id = smart_str(urlsafe_base64_decode(uidb64))
            user = User.objects.get(id=id)

            if not PasswordResetTokenGenerator().check_token(user, token):
                raise serializers.ValidationError("The reset link is invalid or has expired")
            user.set_password(password)
            user.save()
            return attrs
        except DjangoUnicodeDecodeError:
            raise serializers.ValidationError("The reset link is invalid or has expired")
        except User.DoesNotExist:
            raise serializers.ValidationError("User does not exist")
        
```

### 1.5 Views (`users/views.py`)

```python

# api/views.py
from pyexpat.errors import messages
from rest_framework import viewsets, permissions, status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework.decorators import action
from django.contrib.auth import get_user_model
from .permissions import IsAdminOrReadOnly, IsOwnerOrReadOnly, IsOwner, onlygetandcreate, IsadressOwner
from rest_framework.parsers import MultiPartParser, FormParser, JSONParser
from .models import (
    Profile, Setting, Tag, Order, Product,
    Category, Cart, CartItem, ProductReview, Address
)
from .serializers import (
    ProfileSerializer, SettingSerializer, TagSerializer,
    OrderSerializer, ProductSerializer, CategorySerializer,
    CartSerializer, CartItemSerializer, UserRegistrationSerializer,
    loginserializer, ChangePasswordSerializer,
    ResetPasswordgenaretSerializer, setResetPasswordSerializer, ProductReviewSerializer, adressserializer
)
import stripe
from django.conf import settings
from django.shortcuts import get_object_or_404
from rest_framework.response import Response
from rest_framework.views import APIView
from .models import Order

stripe.api_key = settings.STRIPE_SECRET_KEY
from threading import Thread
from .utils import  send_welcome_email
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import AuthenticationFailed
User = get_user_model()
from django.db.models.functions import Coalesce
# function to get tokens for user
def get_tokens_for_user(user):
    if not user.is_active:
      raise AuthenticationFailed("User is not active")

    refresh = RefreshToken.for_user(user)
    return {
        'refresh': str(refresh),
        'access': str(refresh.access_token),
    }


# 🔹 Registration
class RegisterView(APIView):
    permission_classes = [permissions.AllowAny]
    def post(self, request):
        serializer = UserRegistrationSerializer(data=request.data)
        if serializer.is_valid():
            res=serializer.save()
            return Response(res,status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    # login users serialize data
    def get(self, request):
        if not request.user.is_authenticated:
            return Response({"detail": "Authentication credentials were not provided."}, status=status.HTTP_401_UNAUTHORIZED)
        serializer = UserRegistrationSerializer(request.user)
        return Response(serializer.data, status=status.HTTP_200_OK)

# 🔹 Login
class LoginView(APIView):
    permission_classes = [permissions.AllowAny]
    def post(self, request):
        serializer = loginserializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        return Response({"message":"Congratulation you are succesfully loged in",**serializer.validated_data['token']}, status=status.HTTP_200_OK)

# 🔹 Change Password
class ChangePasswordView(APIView):
    permission_classes = [permissions.IsAuthenticated]
    def post(self, request):
        serializer = ChangePasswordSerializer(data=request.data, context={'user': request.user})
        if serializer.is_valid():
            serializer.save()
            return Response({"message": "Password changed successfully",'user':serializer.data}, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


# 🔹 Password Reset (Send Email)
class PasswordResetRequestView(APIView):
    permission_classes = [permissions.AllowAny]
    def post(self, request):
        serializer = ResetPasswordgenaretSerializer(data=request.data)
        if serializer.is_valid():
            msg = serializer.save()
            return Response({"message": msg}, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


# 🔹 Password Reset (Set New Password)
class PasswordResetConfirmView(APIView):
    permission_classes = [permissions.AllowAny]
    def post(self, request):
        serializer = setResetPasswordSerializer(data=request.data)
        if serializer.is_valid():
            return Response({"success": "Password reset successful"}, status=status.HTTP_200_OK)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

from django.utils import timezone
from .utils import send_otp_via_email
# verify email with otp when user register

class verify_register_otp(APIView):
    permission_classes = [permissions.AllowAny]
    def post(self, request):
        otp = request.data.get("otp")
        email = request.data.get("email")
        if not otp:
            return Response({"error": "OTP is required"}, status=status.HTTP_400_BAD_REQUEST)
        if not email:
            return Response({"error": "Email is required"}, status=status.HTTP_400_BAD_REQUEST)
        try:
         user = User.objects.get(email=email)
         if user.is_verified:
             return Response({"error": "Email is already verified"}, status=status.HTTP_400_BAD_REQUEST)
        except User.DoesNotExist:
           return Response({"error": "User not found try again"}, status=status.HTTP_404_NOT_FOUND)

        if user.otp_code == otp and user.otp_expiry > timezone.now():
           user.is_verified = True
           user.otp_code = None
           user.save()
           Thread(target=send_welcome_email, args=[user]).start()
           token=get_tokens_for_user(user)
           Profile.objects.create(user=user)
           Setting.objects.create(user=user)
           return Response({"message": "Email verified successfully",'tokens':token}, status=status.HTTP_200_OK)
        else:
            return Response({"error": "Invalid or expired OTP. Click the Resend OTP button to try again."}, status=status.HTTP_400_BAD_REQUEST)

    # resend otp if user click the resend otp button by get method
    def get(self, request):
        email = request.GET.get("email")
        if not email:
            return Response({"error": "Email is required"}, status=status.HTTP_400_BAD_REQUEST)
        try:
         user = User.objects.get(email=email)
        except User.DoesNotExist:
           return Response({"error": "User not found try again"}, status=status.HTTP_404_NOT_FOUND)
        if user.is_verified:
            return Response({"message": "Your email is already verified, please log in."}, status=status.HTTP_200_OK)
        send_otp_via_email(user)
        return Response({"message": "A new OTP has been sent to your email. Check it and fill in the OTP input."}, status=status.HTTP_200_OK)

```
---

### 1.6 URLs (`users/urls.py`)

```python

from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import (
    ProfileViewSet, SettingViewSet, TagViewSet,
    CategoryViewSet, ProductViewSet, CartViewSet,
    CartItemViewSet, OrderViewSet, ProductReviewViewSet,
    RegisterView, LoginView, ChangePasswordView,
    PasswordResetRequestView, PasswordResetConfirmView, verify_register_otp, Adressviewset, Subscribersviewset,
    PromoCodeListCreateView, ApplyPromoCodeView,SupercategoryViewSet,SSLCommerzCheckoutView,Product_imagesViewSet, payment_success,
    payment_fail, payment_cancel, stripe_webhook, wellcome
)
from rest_framework_simplejwt.views import (
     TokenRefreshView,TokenBlacklistView,TokenVerifyView
)
app_name = 'api'

urlpatterns = [
    # User auth
    path("auth/register/", RegisterView.as_view(), name="register"),
    path("auth/login/", LoginView.as_view(), name="login"),
    path("auth/change-password/", ChangePasswordView.as_view(), name="change-password"),

    # Password reset
    path("auth/request-reset-password/", PasswordResetRequestView.as_view(), name="request-reset-password"),
    path("auth/reset-password-confirm/", PasswordResetConfirmView.as_view(), name="reset-password-confirm"),
    path('auth/verify_user_otp/',verify_register_otp.as_view(),name="verify_user_otp"),
    path("auth/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
    path('auth/token/blacklist/', TokenBlacklistView.as_view(), name='token_blacklist'),
    path('auth/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]


```

### admin.py
```python 
from django.contrib import admin
from django.contrib.auth import get_user_model
from django.contrib.auth.admin import UserAdmin

# Register your models here.
User=get_user_model()
try:
    admin.site.unregister(User)
except admin.sites.NotRegistered:
    pass

@admin.register(User)
class CustomUserAdmin(UserAdmin):
    list_display = (
        "email",
        "fullname",
        "get_language",
        "order_count",
        "get_tags",
    )
    ordering=("email",)
    search_fields = ("email", "first_name", "last_name")
    list_filter = ("is_staff", "is_active", "is_superuser", "groups")
    fieldsets = (
        (None, {"fields": ("email", "password")}),
        ("Personal Info", {"fields": ("first_name", "last_name","is_verified")}),
        ("Permissions", {"fields": ("is_staff", "is_active", "is_superuser", "groups", "user_permissions")}),
    )
    add_fieldsets = (
        (None, {
            "classes": ("wide",),
            "fields": ("email", "first_name", "last_name", "password1", "password2", "is_staff", "is_active", "is_superuser")}
        ),
    )

    def fullname(self, obj):
        return f"{obj.first_name} {obj.last_name}"
    fullname.short_description = "Full Name"

    def get_language(self, obj):
        setting = Setting.objects.filter(user=obj).first()
        return setting.language if setting else "N/A"
    get_language.short_description = "Language"

    def order_count(self, obj):
        return Order.objects.filter(user=obj).count()
    order_count.short_description = "Total Orders"

    def get_tags(self, obj):
        tags = Tag.objects.filter(tag_author=obj)
        return ", ".join(tag.name for tag in tags)
    get_tags.short_description = "Tags"

```

---

## 2️⃣ Frontend Setup (Next.js + Axios)

### 2.1 Axios instance (`lib/axios.ts`)

```ts
import axios from 'axios'
import { clearTokens, getStoredTokens, saveTokens } from './auth'

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_BACKEND_URL, // তোমার backend base URL
  headers: { 'Content-Type': 'application/json' }
})

// Request interceptor → প্রতিটি request এর সাথে token attach করব
api.interceptors.request.use((config) => {
  const { accessToken } = getStoredTokens()
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`
  }
  return config
})

// Response interceptor → 401 পেলে refresh চেষ্টা করবে
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true
      try {
        const { refreshToken } = getStoredTokens()
        if (!refreshToken) {
          return Promise.reject(error)
        }

        // Refresh token দিয়ে নতুন access আনব
        const res = await axios.post(`${process.env.NEXT_PUBLIC_BACKEND_URL}auth/token/refresh/`, {
          refresh: refreshToken
        })

        const newAccess = res.data.access
        const newRefresh = refreshToken // কারণ rotate=false

        // Redux + localStorage update
        saveTokens(newAccess, newRefresh)

        // Retry original request
        originalRequest.headers.Authorization = `Bearer ${newAccess}`
        return api(originalRequest)
      } catch (err) {
        // Refresh কাজ না করলে → logout
        clearTokens()
        return Promise.reject(err)
      }
    }
    return Promise.reject(error)
  }
)

export default api

```

---

### 2.2 Signup API Call & Login API Call

```tsx
'use client'
import { poppins } from '@/components/Navbar'
import { Button } from '@/components/ui/button'
import { loginSuccess } from '@/lib/authcreateslice'
import { RootState } from '@/lib/configstore'
import { isvalidtoken } from '@/lib/validatels'
import { auth_type } from '@/type/auth'
import Image from 'next/image'
import Link from 'next/link'
import { useRouter } from 'next/navigation'
import { useEffect, useState } from 'react'
import { FaEye, FaEyeSlash } from 'react-icons/fa6'
import { useDispatch, useSelector } from 'react-redux'

const SignPage = () => {
  const [isLogin, setIsLogin] = useState<boolean>(true)
  const [formdata, setFormdata] = useState<auth_type>({
    first_name: '',
    last_name: '',
    email: '',
    password: '',
    password2: ''
  })
  const dispatch = useDispatch()
  const [error, setError] = useState<any>('')
  const [isLoading, setIsLoading] = useState<boolean>(false)
  const [success, setSuccess] = useState<string>('')
  const [showPassword, setShowPassword] = useState<boolean[]>([false, false])
  const handleShowPassword = (index: number) => {
    const updatedShowPassword = [...showPassword]
    updatedShowPassword[index] = !updatedShowPassword[index]
    setShowPassword(updatedShowPassword)
  }
  const isAuthenticated = useSelector((state: RootState) => state.auth.isAuthenticated)
  const router = useRouter()
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setFormdata({ ...formdata, [e.target.name]: e.target.value })
  }
  // handle submit function
  const handlesubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError('')
    setSuccess('')
    setIsLoading(true)
    if (!isLogin && formdata.password !== formdata.password2) {
      setError('Passwords do not match')
      setIsLoading(false)
      return
    }
    const url = isLogin
      ? `${process.env.NEXT_PUBLIC_BACKEND_URL}auth/login/`
      : `${process.env.NEXT_PUBLIC_BACKEND_URL}auth/register/`

    const payload = isLogin
      ? {
          email: formdata.email,
          password: formdata.password
        }
      : {
          first_name: formdata.first_name,
          last_name: formdata.last_name,
          email: formdata.email,
          password: formdata.password,
          password2: formdata.password2
        }
    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(payload)
      })
      const text = await response.text()
      let data
      try {
        data = JSON.parse(text)
      } catch {
        data = { error: 'Invalid JSON response', raw: text }
      }

      if (!response.ok) {
        setError(data)
        setIsLoading(false)
      } else {
        setSuccess(isLogin ? data.message : data)
        dispatch(
          loginSuccess({
            accessToken: data.access,
            refreshToken: data.refresh
          })
        )

        if (isLogin) {
          const isvaild = await isvalidtoken()
          if (isvaild) {
            router.push('/profile')
          }
        }
        console.log(data)
        setIsLoading(false)
        // navigate to verify email page
        if (!isLogin) {
          setTimeout(() => {
            router.push(`/verifyemail?email=${encodeURIComponent(formdata.email)}`)
          }, 1000)
        }
        setFormdata({
          first_name: '',
          last_name: '',
          email: '',
          password: '',
          password2: ''
        })
      }
    } catch (error) {
      setError(`Something went wrong ${JSON.stringify(error)}`)
      console.log(error)
      setIsLoading(false)
    }
  }
  useEffect(() => {
    if (isAuthenticated) {
      router.push('/profile')
    }
  }, [isAuthenticated, router])
  return (
    <section
      className={`flex flex-col md:flex-row items-stretch md:gap-10 gap-4 p-4 bg-txt2 shadow-md rounded-lg ${poppins.className}`}
    >
      {/* image section */}
      <div className="w-full md:w-1/2 flex-1 md:flex hidden">
        <Image
          src={isLogin ? '/login.jpg' : '/signup.jpg'}
          width={400}
          height={400}
          alt="loginimg"
          priority
          className="md:w-auto w-full object-cover rounded-2xl"
        />
      </div>
      {/* form section */}
      <div className="w-full md:w-1/2 flex flex-col gap-4 flex-1 p-6 md:px-16">
        <h2 className="text-size4 md:text-size5 text-white font-bold">
          {isLogin ? 'Welcome Back!' : 'Create an Account!'}
        </h2>
        {/* switching is login */}
        <p className="text-size0 font-normal text-gray-400">
          {isLogin ? 'Not a member? ' : 'Already have an account? '}{' '}
          <span
            className="underline text-blue-800 cursor-pointer"
            onClick={() => setIsLogin(!isLogin)}
          >
            {isLogin ? 'Create account' : 'Login'}
          </span>
        </p>
        {/* form */}
        <form className="flex flex-col gap-4" onSubmit={handlesubmit}>
          {/* first name last name */}
          {!isLogin && (
            <div className="flex justify-start items-start md:items-center md:gap-4 gap-2 flex-col md:flex-row ">
              <input
                type="text"
                className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full"
                placeholder="First name"
                name="first_name"
                required
                value={formdata.first_name}
                onChange={handleChange}
              />
              <input
                type="text"
                className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full"
                placeholder="Last name"
                name="last_name"
                required
                value={formdata.last_name}
                onChange={handleChange}
              />
            </div>
          )}
          {isLogin && error.first_name && (
            <p className="text-red-500 text-size1 font-normal">{error.first_name}</p>
          )}
          {isLogin && error.last_name && (
            <p className="text-red-500 text-size1 font-normal">{error.last_name}</p>
          )}
          {/* email */}
          <input
            type="email"
            className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full"
            placeholder="Email"
            name="email"
            required
            value={formdata.email}
            onChange={handleChange}
          />
          {error.email && <p className="text-red-500 text-size1 font-normal">{error.email}</p>}
          {/* Password */}
          <div className="flex px-4 py-3 has-focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] text-size1 font-normal w-full justify-between items-center gap-3">
            <input
              type={showPassword[0] ? 'text' : 'password'}
              placeholder="Enter your password"
              className="placeholder:text-gray-400 focus:placeholder:text-white outline-0 focus:outline-0 active:outline-0 active:border-0 grow peer text-gray-400 focus:text-white"
              required
              name="password"
              value={formdata.password}
              onChange={handleChange}
            />
            {/* eye button from react icon */}
            {!showPassword[0] ? (
              <FaEye
                size={22}
                className="cursor-pointer text-gray-400 peer-focus:text-white"
                onClick={() => handleShowPassword(0)}
              />
            ) : (
              <FaEyeSlash
                size={22}
                className="cursor-pointer text-gray-400 peer-focus:text-white"
                onClick={() => handleShowPassword(0)}
              />
            )}
          </div>
          {error.password && (
            <p className="text-red-500 text-size1 font-normal">{error.password}</p>
          )}
          {/* confirm password */}
          {!isLogin && (
            <div className="flex px-4 py-3 has-focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] text-size1 font-normal w-full justify-between items-center gap-3">
              <input
                type={showPassword[1] ? 'text' : 'password'}
                placeholder="Confirm your password"
                className="placeholder:text-gray-400 focus:placeholder:text-white outline-0 focus:outline-0 active:outline-0 active:border-0 grow peer text-gray-400 focus:text-white"
                required
                name="password2"
                value={formdata.password2}
                onChange={handleChange}
              />
              {/* eye button from react icon */}
              {!showPassword[1] ? (
                <FaEye
                  size={22}
                  className="cursor-pointer text-gray-400 peer-focus:text-white"
                  onClick={() => handleShowPassword(1)}
                />
              ) : (
                <FaEyeSlash
                  size={22}
                  className="cursor-pointer text-gray-400 peer-focus:text-white"
                  onClick={() => handleShowPassword(1)}
                />
              )}
            </div>
          )}
          {!isLogin && error.password2 && (
            <p className="text-red-500 text-size1 font-normal">{error.password2}</p>
          )}
          {/* submit button */}
          <button
            type="submit"
            className="w-full px-4 py-3 bg-primary text-white rounded-md hover:bg-orange-600 transition cursor-pointer text-size3 font-bold md:text-size4"
          >
            {isLogin
              ? isLoading
                ? 'Logging in...'
                : 'Login'
              : isLoading
              ? 'Signing up...'
              : 'Sign Up'}
          </button>
          {/* error message */}
          {isLogin && (
            <Link href="/forgetpassword" className="w-full">
              <Button variant={'link'} className="cursor-pointer block p-0 ml-auto">
                Forget Password?
              </Button>
            </Link>
          )}
          {error && (
            <p className="text-red-500 text-size1 font-normal">
              {typeof error == 'string'
                ? error
                : error.error
                ? error.error
                : Object.values(error)[0]}
            </p>
          )}
          {/* success message */}
          {success && (
            <p className="text-green-500 text-size1 font-normal">
              {typeof success == 'string' ? success : JSON.stringify(success)}
            </p>
          )}
        </form>
        {/* Divider and in center or register with */}
        <div className="flex justify-between items-center gap-3">
          <div className="bg-gray-400 h-[1px] grow" />
          <p className="text-gray-400 font-normal text-size0">
            Or {isLogin ? 'Login' : 'Register'} with
          </p>
          <div className="bg-gray-400 h-[1px] grow" />
        </div>
        {/* google and facebook social login */}
        <div className="flex justify-center items-center gap-6">
          <div className="w-10 h-10 rounded-full bg-white flex justify-center items-center cursor-pointer hover:scale-105 transition">
            <Image
              src="/google.png"
              alt="google"
              width={160}
              height={160}
              className="rounded-full"
            />
          </div>
          <div className="w-10 h-10 rounded-full bg-white flex justify-center items-center cursor-pointer hover:scale-105 transition">
            <Image
              src="/facebook.png"
              alt="facebook"
              width={160}
              height={160}
              className="rounded-full"
            />
          </div>
        </div>
      </div>
    </section>
  )
}

export default SignPage

```

---

### 2.4 Password Reset Request && forget

```tsx
'use client'
import LoadingLoader from '@/components/Loader'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { RootState } from '@/lib/configstore'
import axios from 'axios'
import Image from 'next/image'
import { useRouter } from 'next/navigation'
import { ChangeEvent, useState } from 'react'
import { MdVerified } from 'react-icons/md'
import { useSelector } from 'react-redux'
import { toast } from 'sonner'

const ChangePasswordpage = () => {
  const router = useRouter()
  const profileimg =
    useSelector((state: RootState) => state.profile.profile?.profile_image) ||
    '/deafaltprofile_square.jpg'
  const user = useSelector((state: RootState) => state.user.user)
  // loading state
  const [loading, setloading] = useState<boolean>(false)
  // Error state
  // formdata state
  const [email, setemail] = useState<string>('')

  // handle change
  const handlechange = (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const value = e.target.value
    setemail(value)
    console.log(email)
  }

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setloading(true)
    console.log(email)
    if (!/^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/.test(email)) {
      toast('Please Enter a valid email', {
        action: {
          label: 'Undo',
          onClick: () => {
            console.log('remove notice')
          }
        }
      })
      setloading(false)
      return
    }
    try {
      const response = await axios.post<object>(
        `${process.env.NEXT_PUBLIC_BACKEND_URL}auth/request-reset-password/`,
        { email },
        {
          validateStatus: () => true
        }
      )
      if (!(response.status == 200 || response.status == 201)) {
        setloading(false)
        toast(Object.values(response.data).flat()[0], {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
        return
      }
      setloading(false)
      toast(Object.values(response.data).flat()[0], {
        action: {
          label: 'Undo',
          onClick: () => {
            console.log('remove notice')
          }
        }
      })
      router.push('/profile')
    } catch (error) {
      setloading(false)
      console.log()
      if (error instanceof Error) {
        toast(error.message, {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
      } else {
        toast('Sorry something went wrong please contact with us', {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
      }
    }
  }
  if (loading) {
    return <LoadingLoader />
  }
  return (
    <div className="min-h-screen bg-gray-100 flex items-center justify-center rounded-xl p-3 md:p-5 lg:px-18">
      <div className="bg-white shadow-md rounded-lg p-6 w-full md:max-w-md">
        {/* profile base */}
        <div className="text-center">
          <Image
            src={profileimg}
            alt="Profile"
            width={96}
            height={96}
            className="w-24 h-24 rounded-full mx-auto border-4 border-orange-500"
          />

          <div className="flex justify-center items-center  gap-1.5">
            {(user?.is_staff || user?.is_superuser) && (
              <MdVerified className="block" size={24} color="blue" />
            )}
            <h1 className="text-2xl font-bold text-gray-800">
              {user?.first_name} {user?.last_name}
            </h1>
          </div>

          <p className="text-gray-600">
            {user?.is_superuser
              ? 'Application Admin'
              : user?.is_staff
              ? 'Application Manager'
              : 'General User'}
          </p>
        </div>
        {/* contact section */}
        <section className="mt-10 space-y-2">
          <form
            method="POST"
            encType="multipart/formdata"
            className=" mt-10 space-y-2"
            onSubmit={handleSubmit}
          >
            <div className="flex flex-col gap-2">
              <label className="text-primary font-bold text-size3">Email</label>
              <Input
                value={email}
                name="email"
                className="border-primary text-size2 focus-visible:ring-0 notudbtn w-full"
                placeholder="Enter your email"
                onChange={handlechange}
                type="text"
                required
              />
            </div>
            <Button type="submit" className="tracking-wider cursor-pointer block">
              Reset Password
            </Button>
          </form>
        </section>
      </div>
    </div>
  )
}

export default ChangePasswordpage

```

### 2.6 Change Password

```tsx
'use client'
import LoadingLoader from '@/components/Loader'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { RootState } from '@/lib/configstore'
import { formdata_changepassword } from '@/type/editprofiletype'
import api from '@/utils/api'
import Image from 'next/image'
import { useRouter } from 'next/navigation'
import { ChangeEvent, useState } from 'react'
import { MdVerified } from 'react-icons/md'
import { useSelector } from 'react-redux'
import { toast } from 'sonner'

const ChangePasswordpage = () => {
  const router = useRouter()
  const profileimg =
    useSelector((state: RootState) => state.profile.profile?.profile_image) ||
    '/deafaltprofile_square.jpg'
  const user = useSelector((state: RootState) => state.user.user)
  // loading state
  const [loading, setloading] = useState<boolean>(false)
  // Error state
  // formdata state
  const [formdata, setformdata] = useState<formdata_changepassword>({
    old_password: '',
    new_password: '',
    confirm_password: ''
  })

  // handle change
  const handlechange = (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const name = e.target.name
    const value = e.target.value
    setformdata((data) => ({ ...data, [name]: value }))
    console.log(formdata)
  }

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setloading(true)
    console.log(formdata)
    try {
      const response = await api.post<object>(
        `${process.env.NEXT_PUBLIC_BACKEND_URL}auth/change-password/`,
        formdata,
        {
          validateStatus: () => true
        }
      )
      if (!(response.status == 200 || response.status == 201)) {
        setloading(false)
        console.log(response.data)
        toast(Object.values(response.data).flat()[0], {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
        return
      }
      setloading(false)
      toast(Object.values(response.data).flat()[0], {
        action: {
          label: 'Undo',
          onClick: () => {
            console.log('remove notice')
          }
        }
      })
      router.push('/profile')
    } catch (error) {
      setloading(false)
      console.log()
      if (error instanceof Error) {
        toast(error.message, {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
      } else {
        toast('Sorry something went wrong please contact with us', {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
      }
    }
  }
  if (loading) {
    return <LoadingLoader />
  }
  return (
    <div className="bg-gray-100 flex items-center justify-center rounded-xl p-3 md:p-5 lg:px-18">
      <div className="bg-white shadow-md rounded-lg p-6 w-full md:max-w-md">
        {/* profile base */}
        <div className="text-center">
          <Image
            src={profileimg}
            alt="Profile"
            width={96}
            height={96}
            className="w-24 h-24 rounded-full mx-auto border-4 border-orange-500"
          />

          <div className="flex justify-center items-center  gap-1.5">
            {(user?.is_staff || user?.is_superuser) && (
              <MdVerified className="block" size={24} color="blue" />
            )}
            <h1 className="text-2xl font-bold text-gray-800">
              {user?.first_name} {user?.last_name}
            </h1>
          </div>

          <p className="text-gray-600">
            {user?.is_superuser
              ? 'Application Admin'
              : user?.is_staff
              ? 'Application Manager'
              : 'General User'}
          </p>
        </div>
        {/* contact section */}
        <section className="mt-10 space-y-2">
          <form
            method="POST"
            encType="multipart/formdata"
            className=" mt-10 space-y-2"
            onSubmit={handleSubmit}
          >
            <div className="flex flex-col gap-2">
              <label className="text-primary font-bold text-size3">Old password</label>
              <Input
                value={formdata.old_password || ''}
                name="old_password"
                className="border-primary text-size2 focus-visible:ring-0 notudbtn w-full"
                placeholder="Write your current password"
                onChange={handlechange}
                type="text"
                required
              />
            </div>
            <label className="text-primary font-bold text-size3 block">New password</label>
            <Input
              value={formdata.new_password || ''}
              name="new_password"
              className="border-primary text-size2 focus-visible:ring-0 w-full"
              placeholder="Write your new password"
              onChange={handlechange}
              type="text"
              required
            />
            <label className="text-primary font-bold text-size3 block">Confirm Password</label>
            <Input
              value={formdata.confirm_password || ''}
              name="confirm_password"
              className="border-primary text-size2 focus-visible:ring-0 w-full"
              placeholder="Retype your new password"
              onChange={handlechange}
              type="text"
              required
            />
            <Button type="submit" className="tracking-wider cursor-pointer block">
              Changed Password
            </Button>
          </form>
        </section>
      </div>
    </div>
  )
}

export default ChangePasswordpage

```

### verify email

```tsx 
'use client'
import { poppins } from '@/components/Navbar'
import { field_type } from '@/type/auth'
import Image from 'next/image'
import { useRouter } from 'next/navigation'
import { useEffect, useRef, useState } from 'react'

const SignPage = ({ searchParams }: { searchParams?: { email?: string } }) => {
  const router = useRouter()
  const email = searchParams?.email
  const myForm = useRef<HTMLFormElement | null>(null)
  const [formdata, setFormdata] = useState<field_type>({
    field1: '',
    field2: '',
    field3: '',
    field4: '',
    field5: '',
    field6: ''
  })
  const inputs = useRef<HTMLInputElement[]>([])
  const [error, setError] = useState<any>('')
  const [isLoading, setIsLoading] = useState<boolean>(false)
  const [success, setSuccess] = useState<any>('')
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value.replace(/[^0-9]/g, '')
    setFormdata({ ...formdata, [e.target.name]: value })
  }

  const handleKeyUp = (e: React.KeyboardEvent<HTMLInputElement>, index: number) => {
    const key = `field${index + 1}` as keyof field_type
    if (e.key === 'Backspace' && !formdata[key] && index > 0) {
      inputs.current[index - 1]?.focus()
    }
    if (e.key !== 'Backspace' && formdata[key] && index < 5) {
      inputs.current[index + 1]?.focus()
    }
  }
  // handle resend otp function

  const handleResend = async () => {
    setFormdata({
      field1: '',
      field2: '',
      field3: '',
      field4: '',
      field5: '',
      field6: ''
    })
    if (!email) {
      setError('email not found')
      setIsLoading(false)
      return
    }
    setError('')
    setSuccess('')
    setIsLoading(true)
    const url = `${
      process.env.NEXT_PUBLIC_BACKEND_URL
    }auth/verify_user_otp/?email=${encodeURIComponent(email)}`
    try {
      const response = await fetch(url)
      const text = await response.text()
      let data
      try {
        data = JSON.parse(text)
      } catch {
        data = { error: 'Invalid JSON response', raw: text }
      }
      setSuccess(data)
      setIsLoading(false)
      inputs.current[0].focus()
    } catch (error) {
      setError(error)
      setIsLoading(false)
    }
  }
  // handle submit function
  const handlesubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError('')
    setSuccess('')
    setIsLoading(true)
    const url = `${process.env.NEXT_PUBLIC_BACKEND_URL}auth/verify_user_otp/`
    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ email: email, otp: Object.values(formdata).join('') })
      })

      const text = await response.text()
      let data
      try {
        data = JSON.parse(text)
      } catch {
        data = { error: 'Invalid JSON response', raw: text }
      }

      if (!response.ok) {
        setError(data)
        setIsLoading(false)
      } else {
        setSuccess(data)
        console.log(data)
        setIsLoading(false)
        setFormdata({
          field1: '',
          field2: '',
          field3: '',
          field4: '',
          field5: '',
          field6: ''
        })
      }
    } catch (error) {
      setError(error)
      setIsLoading(false)
    }
  }

  useEffect(() => {
    if (!email) {
      router.push('/sign')
    }
    inputs.current[0].focus()
  }, [email, router])

  useEffect(() => {
    if (
      myForm.current &&
      formdata.field1 &&
      formdata.field2 &&
      formdata.field3 &&
      formdata.field4 &&
      formdata.field5 &&
      formdata.field6
    ) {
      myForm.current.requestSubmit()
    }
  }, [formdata])

  console.log(JSON.stringify(formdata))
  return (
    <section
      className={`flex flex-col md:flex-row items-stretch md:gap-10 gap-4 p-4 bg-txt2 shadow-md rounded-lg ${poppins.className}`}
    >
      {/* image section */}
      <div className="w-full md:w-1/2 flex-1 md:flex hidden">
        <Image
          src={'/verifyemail.jpg'}
          width={400}
          height={400}
          alt="loginimg"
          priority
          className="md:w-auto w-full object-cover rounded-2xl"
        />
      </div>
      {/* form section */}
      <div className="w-full md:w-1/2 flex flex-col gap-4 flex-1 p-6 md:px-16">
        <h2 className="text-size4 md:text-size5 text-white font-bold">Verify Your Email</h2>
        <h3 className="text-size1 font-normal text-gray-300">
          We sent a OTP to your email{' '}
          <span className="text-primary font-semibold underline">{email}</span>
        </h3>
        {/* form */}
        <form className="flex flex-col gap-4" onSubmit={handlesubmit} ref={myForm}>
          <div className="flex justify-start items-start md:items-center md:gap-4 gap-2">
            <input
              type="text"
              className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full notudbtn text-center"
              name="field1"
              required
              value={formdata.field1}
              onChange={(e) => handleChange(e)}
              maxLength={1}
              ref={(el) => {
                if (el) {
                  inputs.current[0] = el
                }
              }}
              onKeyUp={(e) => handleKeyUp(e, 0)}
              pattern="[0-9]{1}"
            />
            <input
              type="text"
              className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full notudbtn text-center"
              name="field2"
              required
              value={formdata.field2}
              onChange={(e) => handleChange(e)}
              maxLength={1}
              ref={(el) => {
                if (el) {
                  inputs.current[1] = el
                }
              }}
              onKeyUp={(e) => handleKeyUp(e, 1)}
              pattern="[0-9]{1}"
            />
            <input
              type="text"
              className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full notudbtn text-center"
              name="field3"
              required
              value={formdata.field3}
              onChange={(e) => handleChange(e)}
              maxLength={1}
              pattern="[0-9]{1}"
              ref={(el) => {
                if (el) {
                  inputs.current[2] = el
                }
              }}
              onKeyUp={(e) => handleKeyUp(e, 2)}
            />
            <input
              type="text"
              className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full notudbtn text-center"
              name="field4"
              required
              value={formdata.field4}
              pattern="[0-9]{1}"
              onChange={(e) => handleChange(e)}
              maxLength={1}
              ref={(el) => {
                if (el) {
                  inputs.current[3] = el
                }
              }}
              onKeyUp={(e) => handleKeyUp(e, 3)}
            />
            <input
              type="text"
              className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full notudbtn text-center"
              name="field5"
              required
              value={formdata.field5}
              onChange={(e) => handleChange(e)}
              maxLength={1}
              pattern="[0-9]{1}"
              ref={(el) => {
                if (el) {
                  inputs.current[4] = el
                }
              }}
              onKeyUp={(e) => handleKeyUp(e, 4)}
            />
            <input
              type="text"
              className="block px-4 py-3 focus:border-2 border-0 border-primary rounded-md bg-[#363c4c] placeholder:text-gray-400 focus:placeholder:text-white text-gray-400 focus:text-white text-size1 font-normal outline-0 focus:outline-0 active:outline-0 active:border-0 w-full notudbtn text-center"
              name="field6"
              required
              value={formdata.field6}
              onChange={(e) => handleChange(e)}
              maxLength={1}
              pattern="[0-9]{1}"
              ref={(el) => {
                if (el) {
                  inputs.current[5] = el
                }
              }}
              onKeyUp={(e) => handleKeyUp(e, 5)}
            />
          </div>
          <div className="flex gap-2 items-center">
            <button
              type="submit"
              className="w-full px-4 py-3 bg-primary text-white rounded-md hover:bg-orange-600 transition cursor-pointer text-size3 font-bold md:text-size4"
            >
              {isLoading ? 'Loading...' : 'Verify Email'}
            </button>
            <button
              type="button"
              className="w-full px-4 py-3 bg-primary text-white rounded-md hover:bg-orange-600 transition cursor-pointer text-size3 font-bold md:text-size4"
              onClick={handleResend}
            >
              {isLoading ? 'Loading...' : 'Resend OTP'}
            </button>
          </div>
          {/* error message */}
          {error && (
            <p className="text-red-500 text-size1 font-normal">
              {typeof error == 'string'
                ? error
                : error.error
                ? error.error
                : Object.values(error)[0]}
            </p>
          )}
          {/* success message */}
          {success && (
            <p className="text-green-500 text-size1 font-normal">
              {success && (
                <p className="text-green-500 text-size1 font-normal">
                  {typeof success === 'string' ? success : String(Object.values(success)[0])}
                </p>
              )}
            </p>
          )}
        </form>
      </div>
    </section>
  )
}

export default SignPage

```


### reset confirm
```pgsql 
--->directory

->app->reset password->[...params]

```
```tsx 
'use client'
import LoadingLoader from '@/components/Loader'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { logout } from '@/lib/authcreateslice'
import { AppDispatch } from '@/lib/configstore'
import { formdata_resetpassword } from '@/type/editprofiletype'
import axios from 'axios'
import { useRouter } from 'next/navigation'
import { ChangeEvent, useState } from 'react'
import { useDispatch } from 'react-redux'
import { toast } from 'sonner'

const ChangePasswordpage = ({ params }: { params: { params: string[] } }) => {
  const [uid, token] = params.params
  const router = useRouter()
  // const profileimg =
  //   useSelector((state: RootState) => state.profile.profile?.profile_image) ||
  //   '/deafaltprofile_square.jpg'
  // const user = useSelector((state: RootState) => state.user.user)
  // loading state
  const [loading, setloading] = useState<boolean>(false)
  // Error state
  // formdata state
  const [formdata, setformdata] = useState<formdata_resetpassword>({
    password: '',
    confirm_password: '',
    token: token,
    uidb64: uid
  })

  // dispatch

  const dispatch = useDispatch<AppDispatch>()

  // handle change
  const handlechange = (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const name = e.target.name
    const value = e.target.value
    setformdata((data) => ({ ...data, [name]: value }))
    console.log(formdata)
  }

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setloading(true)
    console.log(formdata)
    try {
      const response = await axios.post<object>(
        `${process.env.NEXT_PUBLIC_BACKEND_URL}auth/reset-password-confirm/`,
        formdata,
        {
          validateStatus: () => true
        }
      )
      if (!(response.status == 200 || response.status == 201)) {
        setloading(false)
        console.log(response.data)
        toast(Object.values(response.data).flat()[0], {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
        return
      }
      setloading(false)
      toast(Object.values(response.data).flat()[0], {
        action: {
          label: 'Undo',
          onClick: () => {
            console.log('remove notice')
          }
        }
      })
      dispatch(logout())
      router.push('/sign')
    } catch (error) {
      setloading(false)
      console.log()
      if (error instanceof Error) {
        toast(error.message, {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
      } else {
        toast('Sorry something went wrong please contact with us', {
          action: {
            label: 'Undo',
            onClick: () => {
              console.log('remove notice')
            }
          }
        })
      }
    }
  }
  if (loading) {
    return <LoadingLoader />
  }
  return (
    <div className="min-h-screen bg-gray-100 flex items-center justify-center rounded-xl p-3 md:p-5 lg:px-18">
      <div className="bg-white shadow-md rounded-lg p-6 w-full md:max-w-md">
        <h1 className="text-center text-size5 text-primary font-bold border-b-3 px-2 w-fit mx-auto">
          Reset Your Password
        </h1>
        {/* contact section */}
        <section className="mt-10 space-y-2">
          <form
            method="POST"
            encType="multipart/formdata"
            className=" mt-10 space-y-2"
            onSubmit={handleSubmit}
          >
            <div className="flex flex-col gap-2">
              <label className="text-primary font-bold text-size3">Password</label>
              <Input
                value={formdata.password || ''}
                name="password"
                className="border-primary text-size2 focus-visible:ring-0 notudbtn w-full"
                placeholder="Write your new password"
                onChange={handlechange}
                type="text"
                required
              />
            </div>
            <label className="text-primary font-bold text-size3 block">Confirm Password</label>
            <Input
              value={formdata.confirm_password || ''}
              name="confirm_password"
              className="border-primary text-size2 focus-visible:ring-0 w-full"
              placeholder="Retype your new password"
              onChange={handlechange}
              type="text"
              required
            />
            <Button type="submit" className="tracking-wider cursor-pointer block">
              Reset Password
            </Button>
          </form>
        </section>
      </div>
    </div>
  )
}

export default ChangePasswordpage

```

---

### 3️⃣ Next.js Pages / Components

* **Signup page** → form call `signup()`
* **Login page** → form call `login()`
* **Forget Password page** → form call `requestPasswordReset()`
* **Reset Password page** → form call `resetPassword(uid, token, newPassword)`
* **Change Password page** → form

