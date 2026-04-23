# Requirements

## System: E-Commerce Platform

---

## Functional Requirements

### 1. User Management
- Users can register, log in, and manage their profile.
- Users can log in via email/password or OAuth (Google, Facebook).
- Users can manage multiple delivery addresses.
- Admins can manage users, products, and orders.

### 2. Product Catalog
- Users can browse products by category, brand, or search keyword.
- Each product has a name, description, price, images, stock quantity, and seller info.
- Products support multiple variants (size, color, etc.).
- Users can filter and sort results (price, ratings, availability).

### 3. Search
- Users can search for products by keyword.
- Search supports autocomplete and typo tolerance.
- Results are ranked by relevance and personalization signals.

### 4. Cart and Wishlist
- Users can add, update, or remove items from their cart.
- Cart persists across sessions (stored server-side for logged-in users).
- Users can save items to a wishlist.

### 5. Order Management
- Users can place orders for one or multiple items.
- System checks stock availability before confirming an order.
- Users can track their order status: placed → confirmed → shipped → delivered.
- Users can cancel orders before they are shipped.
- Users can return or request a refund within a defined window.

### 6. Payment
- Users can pay via credit/debit card, UPI, net banking, or wallet.
- On payment failure, the cart is preserved and the user is notified.
- On successful payment, an invoice is generated and emailed to the user.

### 7. Inventory Management
- Sellers can add and update product stock levels.
- Stock is decremented atomically when an order is placed.
- Low stock alerts are triggered for sellers when quantity drops below a threshold.

### 8. Notifications
- Users receive email and push notifications for order confirmation, shipping updates, and delivery.
- Sellers receive alerts for new orders and low stock.

### 9. Reviews and Ratings
- Verified buyers can submit a rating (1–5 stars) and a text review.
- Reviews are displayed on the product page with an average rating.

### 10. Recommendations
- Users see personalized product recommendations on the homepage and product pages.
- Recommendations are based on browsing history and past purchases.

---

## Non-Functional Requirements

| Property | Requirement | Target |
|---|---|---|
| **Scalability** | Handle millions of concurrent users during sales events | Support 1M+ concurrent users; horizontal scaling per service |
| **Availability** | Platform must be highly available at all times | 99.99% uptime; no single point of failure |
| **Latency** | Fast product browsing and checkout experience | Product search < 100ms P99; order placement < 300ms P99 |
| **Consistency** | Inventory must never be oversold | Strong consistency for stock decrement; eventual consistency for analytics |
| **Fault Tolerance** | Failure in one service must not cascade | Circuit breakers between services; DLQ for failed async events |
| **Durability** | No order or payment data must be lost | Multi-region DB replication; Kafka durability with replication factor 3 |
| **Security** | Protect user data and prevent abuse | HTTPS everywhere; JWT auth; rate limiting; PCI-DSS compliant payment flow |
| **Maintainability** | Easy to deploy, monitor, and debug | Microservices with CI/CD; centralized logging and tracing |
