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
