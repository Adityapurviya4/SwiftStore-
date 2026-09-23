# SwiftStore
Technical Blueprint &amp; Implementation Guide Tech Stack: React.js (Frontend) + Python FastAPI (Backend) + PostgreSQL (Database) + Firebase Authentication (Phone OTP)

**1. What is the project?**
"OmniCart is a full-stack e-commerce web application inspired by platforms like Flipkart and Meesho. It allows users to log in using their phone number and OTP, browse products by category, filter by size or color, manage saved addresses, place orders using Razorpay or Cash on Delivery, and download tax invoices."

**2. What did you build in each section?**

* **Phone Authentication (Firebase + FastAPI):**
"I used Firebase for fast OTP login. When the user enters their phone number and gets the OTP, Firebase verifies it and returns a token. My Python FastAPI backend checks this token and logs the user in safely."
* **Product Catalog & Search (PostgreSQL):**
"I created a PostgreSQL database schema that handles product variants like size, color, and material. Users can search for products, filter by price, and sort items easily."
* **Address & Location Management:**
"Users can save multiple delivery addresses like Home or Work. I also built a pincode checker to verify if delivery and Cash on Delivery (COD) are available in their area."
* **Persistent Cart System:**
"Instead of storing cart items only in the browser, I save the cart directly in the PostgreSQL database. This means if a user logs out or switches devices, their cart items stay saved."
* **Payments (Prepaid + COD):**
"I integrated Razorpay for online payments like UPI, Cards, and NetBanking. I also added a Cash on Delivery option, which directly creates a pending order in the database."
* **Orders & PDF Invoices:**
"Once an order is placed, users can track its status (Processing, Shipped, Delivered). I used Python to automatically generate a downloadable PDF tax invoice for every order."

---

**Quick Technical Flow (To Explain Your Code Architecture)**

```
[React Frontend] ---> Sends Phone OTP via Firebase ---> Gets Token
       |
       v
[FastAPI Backend] ---> Verifies Token ---> Queries PostgreSQL DB
       |
       +---> Processes Cart & Addresses
       |
       +---> Creates Order via Razorpay API / COD
       |
       v
[PostgreSQL DB] ---> Saves User Data, Orders, & Cart Items

```





-
-
-

-
-
-
-
--

-
--

--

-
-
-
-
-
-
-

-
-
-
-

-
-
-
-
-
-
-
-
-
-
-
-
-
-
-
---
-

-
-
------

-
-
-
-
-
-
-



# SwiftStore: Full-Stack Technical Blueprint & Architecture Guide

**SwiftStore** is an enterprise-grade e-commerce platform built using **React.js**, **Python FastAPI**, **PostgreSQL**, **Firebase Phone Auth**, and **Razorpay**. Below is the deep-dive technical blueprint, database schema, API contracts, and implementation strategy.

---

## 1. System Architecture & Authentication Flow

The system uses a decoupled client-server pattern. The client manages authentication UI and state using the Firebase JS SDK, while the backend utilizes the `firebase-admin` SDK to cryptographically verify JSON Web Tokens (JWTs) issued by Firebase.

---

### Authentication Execution Workflow

1. **Client-Side OTP Request:** Firebase Phone Auth.
The user enters a 10-digit Indian mobile number. React calls `signInWithPhoneNumber(auth, phoneNumber, appVerifier)`. Firebase sends an SMS OTP via its gateway.


2. **OTP Verification & ID Token Generation:**
The user submits the 6-digit OTP. Firebase validates the code and returns a `UserCredential` object containing a short-lived Firebase `idToken` (JWT).


3. **FastAPI Token Verification:**
React passes the `idToken` in the HTTP header (`Authorization: Bearer <idToken>`). FastAPI's dependency injection layer parses the header and calls `firebase_admin.auth.verify_id_token(id_token)`.


4. **User Provisioning & Session Persistence:**
If valid, FastAPI extracts the `phone_number` and `uid`. It queries PostgreSQL: if the user exists, it updates `last_login`; if not, it provisions a new user record.


---

## 2. PostgreSQL Database Schema Design

To support Flipkart/Meesho-style product attributes (where a single product can have multiple sizes, colors, and stock levels), a **Product-Variant architecture** is implemented.

```sql
-- 1. USERS TABLE
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firebase_uid VARCHAR(128) UNIQUE NOT NULL,
    phone_number VARCHAR(15) UNIQUE NOT NULL,
    full_name VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 2. CATEGORIES TABLE
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_id INT REFERENCES categories(id) ON DELETE SET NULL
);

-- 3. PRODUCTS (BASE TABLE)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id INT REFERENCES categories(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    brand VARCHAR(100),
    base_price DECIMAL(10, 2) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 4. PRODUCT VARIANTS (SKU Level)
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(100) UNIQUE NOT NULL,
    size VARCHAR(20),       -- e.g., 'S', 'M', 'L', 'XL'
    color VARCHAR(50),      -- e.g., 'Crimson Red'
    material VARCHAR(50),   -- e.g., '100% Cotton'
    price_override DECIMAL(10, 2), -- Optional variant specific pricing
    stock_quantity INT DEFAULT 0,
    image_urls TEXT[],      -- Array of S3 / CDN image links
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 5. PERSISTENT CART TABLE
CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    quantity INT NOT NULL CHECK (quantity > 0),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT unique_user_variant UNIQUE(user_id, variant_id)
);

-- 6. ADDRESSES TABLE
CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(15) NOT NULL,
    pincode VARCHAR(10) NOT NULL,
    street_address TEXT NOT NULL,
    city VARCHAR(50) NOT NULL,
    state VARCHAR(50) NOT NULL,
    address_type VARCHAR(10) DEFAULT 'HOME', -- HOME / WORK
    is_default BOOLEAN DEFAULT FALSE
);

-- 7. SERVICEABLE PINCODES
CREATE TABLE serviceable_pincodes (
    pincode VARCHAR(10) PRIMARY KEY,
    city VARCHAR(50) NOT NULL,
    state VARCHAR(50) NOT NULL,
    is_cod_available BOOLEAN DEFAULT TRUE,
    estimated_delivery_days INT DEFAULT 5
);

-- 8. ORDERS TABLE
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    address_id UUID REFERENCES addresses(id),
    total_amount DECIMAL(10, 2) NOT NULL,
    discount_amount DECIMAL(10, 2) DEFAULT 0.00,
    payment_method VARCHAR(20) NOT NULL, -- 'RAZORPAY' or 'COD'
    payment_status VARCHAR(20) DEFAULT 'PENDING', -- 'PENDING', 'PAID', 'FAILED'
    order_status VARCHAR(20) DEFAULT 'PROCESSING', -- 'PROCESSING', 'SHIPPED', 'DELIVERED', 'CANCELLED'
    razorpay_order_id VARCHAR(100) UNIQUE,
    razorpay_payment_id VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 9. ORDER ITEMS TABLE
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
    variant_id UUID REFERENCES product_variants(id),
    unit_price DECIMAL(10, 2) NOT NULL,
    quantity INT NOT NULL
);

```

---

## 3. Deep-Dive Implementation Specs

### A. Server-Synced Persistent Cart

Instead of storing state purely in `localStorage` (which breaks cross-device syncing), SwiftStore uses a **Database-First Cart with Client-Side Fallback**.

* **Guest Mode:** Items are stored in React state & local storage.
* **On Login:** The frontend dispatches a `POST /api/cart/sync` payload carrying all guest cart items.
* **Database Upsert:** FastAPI executes an `ON CONFLICT (user_id, variant_id) DO UPDATE` query to merge guest items with existing backend cart items without duplicating rows.

### B. Address & Serviceability Validation

Before proceeding to payment, the frontend issues a lookup to `/api/pincode/check/{pincode}`.

* The API checks `serviceable_pincodes`.
* Returns `is_cod_available` and `estimated_delivery_days`.
* If invalid, the checkout button for COD is disabled dynamically on the React UI.

### C. Razorpay Integration & Secure Webhooks

---

1. **Order Initiation:** React calls `POST /api/orders/create`. FastAPI creates an order in Razorpay using their SDK and returns a `razorpay_order_id`.
2. **Client Checkout:** React opens the Razorpay Modal options. The user completes payment via UPI / NetBanking / Cards.
3. **Verification (Dual Guard):**
* **Immediate Path:** Client passes `razorpay_payment_id` and `razorpay_signature` to `/api/orders/verify`. FastAPI calculates HMAC-SHA256 signature using the secret key.
* **Async Webhook Path:** In case the user closes the browser before redirection, Razorpay hits a FastAPI webhook endpoint (`/api/webhooks/razorpay`) to update `payment_status = 'PAID'` asynchronously.



### D. Automated PDF Invoice Generation

Upon order confirmation, FastAPI uses `ReportLab` or `WeasyPrint` to render HTML templates into legal tax invoices.

* **Trigger:** Order transitions to `PAID` or `CONFIRMED`.
* **Storage:** PDF is compiled in memory (`io.BytesIO`), streamed to S3 bucket / local media folder, and a pre-signed download link is sent to the frontend.

---

## 4. Complete Code Implementation Snippets

### Backend: FastAPI Authentication Middleware (`deps.py`)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import firebase_admin
from firebase_admin import auth, credentials
import os

# Initialize Firebase Admin SDK
cred = credentials.Certificate(os.getenv("FIREBASE_CREDENTIALS_PATH"))
firebase_admin.initialize_app(cred)

security = HTTPBearer()

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    token = credentials.credentials
    try:
        # Cryptographically verify token signature and expiry with Firebase
        decoded_token = auth.verify_id_token(token)
        return {
            "uid": decoded_token["uid"],
            "phone_number": decoded_token.get("phone_number")
        }
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=f"Invalid authentication token: {str(e)}"
        )

```

### Backend: Razorpay Payment & Verification API (`orders.py`)

```python
from fastapi import APIRouter, Depends, HTTPException, status
import razorpay
import hmac
import hashlib
import os
from pydantic import BaseModel

router = APIRouter(prefix="/api/orders", tags=["Orders"])

client = razorpay.Client(auth=(os.getenv("RAZORPAY_KEY_ID"), os.getenv("RAZORPAY_KEY_SECRET")))

class VerificationSchema(BaseModel):
    razorpay_order_id: str
    razorpay_payment_id: str
    razorpay_signature: str

@router.post("/verify")
async def verify_payment(payload: VerificationSchema):
    # Construct expected signature
    msg = f"{payload.razorpay_order_id}|{payload.razorpay_payment_id}"
    generated_signature = hmac.new(
        key=os.getenv("RAZORPAY_KEY_SECRET").encode(),
        msg=msg.encode(),
        digestmod=hashlib.sha256
    ).hexdigest()

    if generated_signature == payload.razorpay_signature:
        # Update database status to PAID
        # trigger_invoice_generation(payload.razorpay_order_id)
        return {"status": "SUCCESS", "message": "Payment verified successfully"}
    else:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Invalid payment signature. Potential tampering detected."
        )

```

---

## 5. Core API Endpoint Reference

| Endpoint | Method | Header Requirement | Description |
| --- | --- | --- | --- |
| `/api/auth/sync` | `POST` | `Bearer <Firebase_JWT>` | Creates or updates user record on initial login |
| `/api/products` | `GET` | Public | Lists products with limit, offset, search, and category query params |
| `/api/products/{id}` | `GET` | Public | Returns detailed product view including all available size/color SKUs |
| `/api/cart` | `GET` | `Bearer <Firebase_JWT>` | Retrieves active server-synced cart for user |
| `/api/cart/item` | `POST` | `Bearer <Firebase_JWT>` | Adds/updates item quantity in cart table |
| `/api/pincode/{code}` | `GET` | Public | Checks delivery and COD serviceability status |
| `/api/orders/create` | `POST` | `Bearer <Firebase_JWT>` | Initializes Razorpay order or creates COD order |
| `/api/orders/verify` | `POST` | `Bearer <Firebase_JWT>` | Verifies Razorpay HMAC signature |
| `/api/orders/{id}/invoice` | `GET` | `Bearer <Firebase_JWT>` | Returns downloadable PDF stream for tax invoice |


-
-
-
-
-
-
-

-
-
-
-
-
--
-
-

Welcome to the **SwiftStore System Design Canvas**. This layout acts as your digital whiteboard, breaking down the application into independent architectural zones spanning from the client interface down to the relational data layer.

---

## 🏛️ Zone 1: System Architecture Map

This flow represents the decoupled client-server architecture, illustrating how the React frontend coordinates with external gateways before communicating with the Python backend.

```text
 ┌────────────────────────────────────────────────────────┐
 │                 CLIENT LAYER (React.js)                │
 │  Handles UI, Local/Guest Cart State, Firebase SDK      │
 └──────┬───────────────────────────────┬─────────────────┘
        │ 1. Request OTP                │ 3. API Calls (Bearer JWT)
        ▼                               ▼
 ┌──────────────┐               ┌─────────────────────────────────┐
 │   EXTERNAL   │ 2. Return JWT │   BACKEND LAYER (FastAPI)       │
 │   SERVICES   ├──────────────►│ ├─ Auth Middleware (Deps.py)    │
 │ (Firebase)   │               │ ├─ Pincode/Serviceability Engine│
 └──────────────┘               │ ├─ Cart Sync Manager            │
                                │ └─ Payment/Checkout Handler     │
                                └───────┬──────────────────┬──────┘
 ┌──────────────┐ 4. Create Order       │ 6. DB Read/Write │ 5. Render
 │   EXTERNAL   │◄──────────────────────┤                  │    PDF
 │   SERVICES   │ 4b. Async Webhook     │                  ▼
 │ (Razorpay)   ├───────────────────────┤           ┌──────────────┐
 └──────────────┘                       ▼           │  FILE STORE  │
                             ┌────────────────────┐ │ (In-Memory/  │
                             │  DATABASE LAYER    │ │  AWS S3)     │
                             │   (PostgreSQL)     │ └──────────────┘
                             └────────────────────┘

```

---

## 🛠️ Zone 2: Tech Stack Matrix

| Layer | Technology | Primary Responsibility |
| --- | --- | --- |
| **Frontend** | React.js | UI rendering, state management, Razorpay checkout modal. |
| **Backend API** | Python (FastAPI) | High-performance async endpoints, business logic, webhooks. |
| **Authentication** | Firebase Auth | Phone SMS routing, OTP verification, JWT generation. |
| **Database** | PostgreSQL | Relational storage, ACID compliance for cart and orders. |
| **Payments** | Razorpay SDK | UPI, Cards, NetBanking processing, and webhook callbacks. |
| **Document Gen** | ReportLab / WeasyPrint | Compiling HTML data into downloadable tax invoice PDFs. |

---

## 🔐 Zone 3: Authentication & Provisioning Flow

The authentication system strictly separates identity verification (handled by Firebase) from authorization and data storage (handled by FastAPI/PostgreSQL).

1. **Client OTP Request:** React.js + Firebase SDK.
The user enters a 10-digit mobile number. The client calls `signInWithPhoneNumber(auth, phone)`. Firebase routes an SMS OTP to the user.


2. **Token Generation:** Firebase Authenticator.
The user inputs the 6-digit OTP. Firebase validates the payload and issues a secure, short-lived JSON Web Token (`idToken`) containing the user's UID.


3. **Backend Verification:** FastAPI Dependency Injection.
React passes the JWT in the `Authorization: Bearer` header to the backend. FastAPI intercepts this and uses `firebase_admin.auth.verify_id_token()` to cryptographically verify the signature.


4. **Database Provisioning:** PostgreSQL Upsert.
FastAPI extracts the UID and phone number. It queries the database: if the user exists, it updates `last_login`; if new, it inserts a new record into the `users` table and initializes their session.


---

## 🛒 Zone 4: Checkout & Payment State Machine

Managing the cart state and finalizing a payment requires multi-stage validation to prevent phantom inventory or unserviceable deliveries.

1. **Cart Synchronization:** Database-First Approach.
On login, the frontend dispatches `POST /api/cart/sync` with guest cart items. FastAPI uses an `ON CONFLICT` query to merge local items with the persistent PostgreSQL cart without duplicating rows.


2. **Serviceability Validation:** Pincode Lookup.
Before checkout, the frontend queries `/api/pincode/check/{pincode}`. The database checks the `serviceable_pincodes` table and returns boolean flags for COD availability and estimated delivery days.


3. **Order Initialization:** Razorpay Pre-flight.
If the user selects a prepaid method, FastAPI calls the Razorpay API to generate a `razorpay_order_id`, ensuring the backend maintains a source of truth for the transaction amount.


4. **Dual-Guard Verification:** Synchronous + Webhook.
Upon payment completion, the client sends the `razorpay_payment_id` and `razorpay_signature` to the backend. FastAPI generates an HMAC-SHA256 hash using the secret key to verify authenticity. An async webhook provides a fallback if the user closes their browser early.


5. **Invoice Generation:** Automated PDF Engine.
Once the order transitions to `PAID`, a Python library renders an HTML template into a PDF invoice, stores it in memory (or an S3 bucket), and sends a pre-signed download link to the client.


---

## 🗄️ Zone 5: Database Entity-Relationship (ER) Canvas

The system utilizes a **Product-Variant** architecture, allowing a single parent product to house multiple SKUs for size and color.

> **Key insight:** The `cart_items` table features a `UNIQUE(user_id, variant_id)` constraint to ensure users only increment quantities rather than creating duplicate cart rows.

| Table Name | Primary Key (PK) | Foreign Keys (FK) & Core Fields | Purpose |
| --- | --- | --- | --- |
| **users** | `id` (UUID) | `firebase_uid`, `phone_number` | Stores authorized user profiles. |
| **categories** | `id` (INT) | `parent_id` (Self-referencing) | Supports hierarchical nesting (e.g., Men -> Shirts). |
| **products** | `id` (UUID) | `category_id` (categories) | Base product details (title, description, base_price). |
| **product_variants** | `id` (UUID) | `product_id` (products) | Specific SKUs (size, color, stock_quantity, images). |
| **addresses** | `id` (UUID) | `user_id` (users) | Saves delivery locations (Home/Work) and pincodes. |
| **serviceable_pincodes** | `pincode` (VARCHAR) | None | Logistics lookup for COD toggles and delivery estimates. |
| **cart_items** | `id` (UUID) | `user_id`, `variant_id` | Persists user cart sessions across multiple devices. |
| **orders** | `id` (UUID) | `user_id`, `address_id` | Tracks financials, statuses, and Razorpay IDs. |
| **order_items** | `id` (UUID) | `order_id`, `variant_id` | Locks in the specific variant and purchase price at checkout. |

---

## 🌐 Zone 6: API Gateway Contracts

The internal API structure handles routing between the stateless frontend and the database layer.

| Endpoint | Method | Auth | Role in Ecosystem |
| --- | --- | --- | --- |
| `/api/auth/sync` | POST | 🛡️ Bearer | Provisions the Firebase user inside the PostgreSQL database. |
| `/api/products` | GET | 🌍 Public | Returns the catalog with offset pagination and search filtering. |
| `/api/products/{id}` | GET | 🌍 Public | Expands a product ID to show all attached variant SKUs. |
| `/api/cart` | GET | 🛡️ Bearer | Fetches the synchronized cross-device shopping cart. |
| `/api/pincode/{code}` | GET | 🌍 Public | Logistics check; actively disables COD on frontend if unserviceable. |
| `/api/orders/create` | POST | 🛡️ Bearer | Initializes either a standard COD order or a Razorpay staging order. |
| `/api/orders/verify` | POST | 🛡️ Bearer | Authenticates the Razorpay HMAC signature to transition order to PAID. |
| `/api/webhooks/razorpay` | POST | 🔐 Secret | Backend-to-backend endpoint for asynchronous payment confirmations. |
| `/api/orders/{id}/invoice` | GET | 🛡️ Bearer | Streams the dynamically generated PDF tax document to the browser. |
