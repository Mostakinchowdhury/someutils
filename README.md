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
            'product_name': order.product_name,
            'product_category': "General",
            'product_profile': "general",
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
from django.views.decorators.csrf import csrf_exempt
from django.http import HttpResponse
import json

@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    event = None
    try:
        event = stripe.Event.construct_from(
            json.loads(payload), stripe.api_key
        )
    except Exception as e:
        return HttpResponse(status=400)

    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]
        order_id = session["metadata"]["order_id"]
        order = Order.objects.get(id=order_id)
        order.status = "PAID"
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
