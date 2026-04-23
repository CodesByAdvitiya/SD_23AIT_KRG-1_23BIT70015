# API Design

## System: E-Commerce Platform

> Base URL: `https://api.ecommerce.com/v1`  
> All protected endpoints require: `Authorization: Bearer <JWT>`  
> All responses use `Content-Type: application/json`

---

## 1. User Endpoints

### POST /auth/register
Register a new user.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass@123",
  "full_name": "Rajan Singh",
  "phone": "9876543210"
}
```

**Response — 201 Created:**
```json
{
  "user_id": "a1b2c3d4-...",
  "email": "user@example.com",
  "full_name": "Rajan Singh",
  "created_at": "2026-04-22T10:00:00Z"
}
```

**Status Codes:**
| Code | Meaning |
|---|---|
| 201 | User created successfully |
| 409 | Email already registered |
| 422 | Validation error (weak password, invalid email) |

---

### POST /auth/login
Authenticate and receive a JWT.

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass@123"
}
```

**Response — 200 OK:**
```json
{
  "access_token": "eyJhbGci...",
  "token_type": "Bearer",
  "expires_in": 900
}
```

**Status Codes:**
| Code | Meaning |
|---|---|
| 200 | Login successful |
| 401 | Invalid credentials |
| 429 | Too many login attempts |

---

## 2. Product Endpoints

### GET /products
Browse and search the product catalog.

**Query Parameters:**
| Parameter | Type | Description |
|---|---|---|
| `q` | string | Search keyword |
| `category_id` | UUID | Filter by category |
| `min_price` | number | Minimum price filter |
| `max_price` | number | Maximum price filter |
| `sort` | string | `price_asc`, `price_desc`, `rating`, `newest` |
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Results per page (default: 20, max: 100) |

**Response — 200 OK:**
```json
{
  "total": 1240,
  "page": 1,
  "limit": 20,
  "products": [
    {
      "product_id": "uuid",
      "name": "Wireless Headphones",
      "brand": "Sony",
      "base_price": 2499.00,
      "avg_rating": 4.3,
      "total_reviews": 852,
      "thumbnail_url": "https://cdn.ecommerce.com/images/abc.jpg"
    }
  ]
}
```

---

### GET /products/{product_id}
Get full details of a single product.

**Response — 200 OK:**
```json
{
  "product_id": "uuid",
  "name": "Wireless Headphones",
  "description": "Noise cancelling over-ear headphones...",
  "brand": "Sony",
  "category": { "id": "uuid", "name": "Electronics" },
  "avg_rating": 4.3,
  "variants": [
    {
      "variant_id": "uuid",
      "sku": "SNY-WH1000XM5-BLK",
      "color": "Black",
      "price": 2499.00,
      "stock_qty": 145,
      "images": ["https://cdn.ecommerce.com/img1.jpg"]
    }
  ]
}
```

**Status Codes:**
| Code | Meaning |
|---|---|
| 200 | Product found |
| 404 | Product not found |

---

## 3. Cart Endpoints

### POST /cart
Add an item to the cart. Requires authentication.

**Request Body:**
```json
{
  "variant_id": "uuid",
  "quantity": 2
}
```

**Response — 200 OK:**
```json
{
  "cart_item_id": "uuid",
  "variant_id": "uuid",
  "quantity": 2,
  "unit_price": 2499.00,
  "line_total": 4998.00
}
```

---

### GET /cart
Retrieve all items in the user's cart.

**Response — 200 OK:**
```json
{
  "items": [
    {
      "cart_item_id": "uuid",
      "product_name": "Wireless Headphones",
      "variant": { "color": "Black", "sku": "SNY-WH1000XM5-BLK" },
      "quantity": 2,
      "unit_price": 2499.00,
      "line_total": 4998.00
    }
  ],
  "subtotal": 4998.00
}
```

---

### DELETE /cart/{cart_item_id}
Remove an item from the cart.

**Response:** `204 No Content`

---

## 4. Order Endpoints

### POST /orders
Place an order. Requires authentication.

**Request Body:**
```json
{
  "address_id": "uuid",
  "idempotency_key": "client-uuid-v4",
  "items": [
    { "variant_id": "uuid", "quantity": 1 }
  ]
}
```

**Response — 202 Accepted:**
```json
{
  "order_id": "uuid",
  "status": "placed",
  "total_amount": 2499.00,
  "final_amount": 2499.00,
  "placed_at": "2026-04-22T12:30:00Z"
}
```

**Status Codes:**
| Code | Meaning |
|---|---|
| 202 | Order accepted and queued for processing |
| 409 | Conflict — duplicate idempotency key (returns original response) |
| 422 | Validation error (invalid address, empty items) |
| 503 | Inventory service unavailable |

---

### GET /orders/{order_id}
Get status and details of a specific order.

**Response — 200 OK:**
```json
{
  "order_id": "uuid",
  "status": "shipped",
  "items": [
    {
      "product_name": "Wireless Headphones",
      "variant": { "color": "Black" },
      "quantity": 1,
      "unit_price": 2499.00
    }
  ],
  "shipping_address": { "line1": "42 MG Road", "city": "Amritsar", "pincode": "143001" },
  "payment_status": "success",
  "tracking_id": "DELHIVERY-9876543",
  "placed_at": "2026-04-22T12:30:00Z",
  "updated_at": "2026-04-23T09:00:00Z"
}
```

---

### POST /orders/{order_id}/cancel
Cancel a placed or confirmed order.

**Response — 200 OK:**
```json
{
  "order_id": "uuid",
  "status": "cancelled",
  "refund_initiated": true,
  "refund_amount": 2499.00
}
```

**Status Codes:**
| Code | Meaning |
|---|---|
| 200 | Order cancelled successfully |
| 400 | Order cannot be cancelled (already shipped) |
| 403 | Order does not belong to this user |
| 404 | Order not found |

---

## 5. Payment Endpoints

### POST /payments
Initiate a payment for an order.

**Request Body:**
```json
{
  "order_id": "uuid",
  "method": "upi",
  "upi_id": "user@ybl"
}
```

**Response — 200 OK:**
```json
{
  "payment_id": "uuid",
  "status": "pending",
  "gateway_txn_id": "RZPY_xyz123",
  "redirect_url": "https://gateway.razorpay.com/pay?txn=xyz123"
}
```

---

### GET /payments/{payment_id}
Get payment status.

**Response — 200 OK:**
```json
{
  "payment_id": "uuid",
  "order_id": "uuid",
  "status": "success",
  "method": "upi",
  "amount": 2499.00,
  "currency": "INR",
  "created_at": "2026-04-22T12:31:00Z"
}
```

---

## 6. Review Endpoints

### POST /products/{product_id}/reviews
Submit a review for a purchased product.

**Request Body:**
```json
{
  "order_id": "uuid",
  "rating": 5,
  "title": "Excellent sound quality",
  "body": "Bought these last week and the noise cancellation is outstanding."
}
```

**Response — 201 Created:**
```json
{
  "review_id": "uuid",
  "rating": 5,
  "title": "Excellent sound quality",
  "created_at": "2026-04-23T08:00:00Z"
}
```

**Status Codes:**
| Code | Meaning |
|---|---|
| 201 | Review submitted |
| 403 | User has not purchased this product |
| 409 | Review already submitted for this order |
