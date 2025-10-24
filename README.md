# Introduction & Abstraction

**Project:** Small online shop for *digital products* (single product category) — focused first on database design and backend-ready abstractions.


## Purpose & Goals

* Build a minimal, robust data model and set of abstractions to support selling downloadable digital products (e.g., e-books, software, media, licenses).
* Keep design simple but extensible so future categories (physical goods, services) can be added.
* Support typical e-commerce flows: product management, purchase, payment record, instant delivery (download/license), refunds, access control.


## Scope (what this step covers)

* High-level problem abstraction: identify core domain concepts, responsibilities, and interactions.
* List functional & non-functional requirements, assumptions, and constraints that drive the database design.
* Produce the vocabulary and entity ideas we will later turn into diagrams and concrete tables.

(Implementation, ER diagrams, SQL will be later steps.)


## Key Assumptions

* Single product category: *digital product* (no physical shipping).
* Instant fulfillment after confirmed payment (download link or license key).
* One currency initially (configurable later).
* Users may or may not require accounts (support guest checkout).
* Tax rules and legal compliance (VAT/GST) handled but can be simplified initially.
* Payment provider is external (we store payment records and statuses, not raw card data).
* Small-to-medium scale initially; eventual scalability considered.



## Core Domain Concepts (high-level entities)

* **User / Customer** — identity, contact info, optional account.
* **DigitalProduct** — metadata describing the product (title, description, file(s), price, SKU, tags).
* **ProductAsset** — actual files or references (file path, storage id, file size, checksum, mime).
* **Order** — purchase record (customer, order date, total, currency, status).
* **OrderItem** — line item referencing a DigitalProduct (price at purchase, quantity usually 1).
* **Payment** — external payment record (provider id, status, amount, timestamp).
* **Delivery / Entitlement** — record that grants the customer access to download / license (download token, expiry, allowed downloads).
* **LicenseKey** (optional) — license string, activation limits, expiry (for licensed software).
* **Refund** — refund record (amount, reason, status).
* **AuditLog** — events for security and troubleshooting (product uploaded, order created, refund issued).
* **Promotion** (optional) — discount code, percentage/amount, validity.
* **Tax / Invoicing** — records to support invoices and tax calculations.



## Important Attributes (sample)

* **User:** id, email, name, password_hash (if account), created_at, locale
* **DigitalProduct:** id, sku, title, slug, description, price, currency, visibility, created_at, metadata (json)
* **ProductAsset:** id, product_id, storage_path, file_size, mime_type, checksum
* **Order:** id, user_id (nullable for guest), status (pending/paid/failed/refunded), total_amount, currency, created_at
* **OrderItem:** id, order_id, product_id, unit_price, quantity, line_total
* **Payment:** id, order_id, provider, provider_payment_id, amount, status, paid_at, raw_response (json)
* **Entitlement:** id, user_id, product_id, order_item_id, download_token, downloads_left, expires_at, issued_at
* **LicenseKey:** id, entitlement_id, key, activations_allowed, activated_count, expires_at
* **Refund:** id, order_id, amount, reason, status, created_at



## Functional Requirements (minimum viable)

1. Admin: create/update/delete digital products and upload product assets.
2. Catalog: list/search products; view product details.
3. Checkout: create order, integrate with payment provider, handle webhooks to update payment status.
4. Fulfillment: on successful payment, create entitlement and provide secure download link / deliver license.
5. Access Control: ensure only entitled users can download files; enforce download limits and expiry.
6. Refund/Cancel: store refund requests and update entitlements/orders accordingly.
7. Reporting/Audit: basic logs for sales, refunds, and product changes.



## Non-Functional Requirements / Constraints

* **Security:** no raw payment card data stored; restrict direct file URLs; authenticated download tokens; hashed passwords; rate limiting for download endpoints.
* **Consistency:** strong consistency for orders/payments (avoid double fulfillment).
* **Scalability:** object storage for files (S3-like), CDN for delivery; DB indices on common queries.
* **Availability:** high for payment and download flows; backup strategy.
* **Privacy & Compliance:** store minimal PII; be ready for GDPR-ish requests (data deletion).
* **Performance:** low latency for checkout and download token generation.



## Business Rules / Logic Highlights

* Products can be bought multiple times; purchases create separate entitlements.
* Entitlement may include download limits and expiry (e.g., 5 downloads within 30 days).
* Refund reverses entitlement (revoke download access) and records refund history.
* Pricing is stored on `OrderItem` to preserve historical price even if product price changes later.
* Admins can mark a product as free (price = 0) — still create an order or entitlement depending on flow.



## Typical Use Cases (flows)

1. **Browse & Purchase:** User finds product → adds to cart → checkout → payment → entitlement created → user downloads.
2. **Guest Purchase:** Same as above but without persistent user account; use email to send links.
3. **Admin Upload:** Admin uploads product asset → product becomes available.
4. **Refund:** Customer requests refund → admin issues refund → entitlement revoked.
5. **License Delivery:** On purchase, generate license key and associate with entitlement.



## Success Metrics (what "good" looks like)

* Order-to-delivery latency: < 30 seconds from payment confirmation to download availability.
* Download failure rate: < 0.5%.
* Data consistency: zero double-fulfillments (no duplicate entitlements for a single payment).
* Time to onboard new product: < 5 minutes (file upload + metadata).
* Conversion rate and average order value — tracked later via analytics.



## Minimal Initial Design Decisions (to lock now)

* Use relational DB (PostgreSQL) for main entities.
* Store product files in object storage (S3-compatible) and keep references in DB.
* Payment provider integration via webhook; store provider payment id and status.
* Implement entitlement tokens as UUIDs with expiry and limited download counters.
* Maintain immutable order history (don't mutate order totals; create refunds separately).

