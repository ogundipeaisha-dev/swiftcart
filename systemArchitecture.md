# SwiftCart System Architecture

**Technical Blueprint for One-Link Checkout Platform**

---

## Architecture Overview

SwiftCart follows a **modern three-tier architecture** with clear separation between presentation, business logic, and data layers. The system is designed for scalability, maintainability, and rapid iteration during the MVP phase.

### **High-Level Architecture Diagram**

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                         │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────────┐  │
│  │  Seller    │  │   Buyer    │  │   Mobile Browsers    │  │
│  │ Dashboard  │  │  Checkout  │  │  (iOS/Android Web)   │  │
│  └────────────┘  └────────────┘  └──────────────────────┘  │
│           React.js + Tailwind CSS (SPA)                      │
└─────────────────────────────────────────────────────────────┘
                            ↓ HTTPS
┌─────────────────────────────────────────────────────────────┐
│                      APPLICATION LAYER                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Express.js REST API Server                  │  │
│  │  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌────────┐ │  │
│  │  │  Auth   │  │ Payment │  │ Checkout │  │ Orders │ │  │
│  │  │ Service │  │ Service │  │ Service  │  │Service │ │  │
│  │  └─────────┘  └─────────┘  └──────────┘  └────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│                   Node.js Runtime                            │
└─────────────────────────────────────────────────────────────┘
            ↓                    ↓                    ↓
┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐
│  EXTERNAL APIs   │  │   DATA LAYER     │  │  NOTIFICATIONS  │
│  ┌────────────┐  │  │  PostgreSQL DB   │  │  ┌───────────┐  │
│  │  Paystack  │  │  │  ┌────────────┐  │  │  │  Twilio   │  │
│  │    API     │  │  │  │   Users    │  │  │  │   (SMS)   │  │
│  └────────────┘  │  │  │   Orders   │  │  │  └───────────┘  │
│                  │  │  │   Links    │  │  │  ┌───────────┐  │
│  (Future:        │  │  │ Payments   │  │  │  │ SendGrid  │  │
│   Logistics APIs)│  │  └────────────┘  │  │  │  (Email)  │  │
└──────────────────┘  └──────────────────┘  └─────────────────┘
```

---

## Technology Stack

### **Frontend**
| Technology | Purpose | Justification |
|------------|---------|---------------|
| **React.js** | UI Framework | Component-based architecture enables rapid iteration and reusable UI elements |
| **Tailwind CSS** | Styling | Utility-first CSS for mobile-first responsive design without custom CSS overhead |
| **React Router** | Client-side routing | SPA navigation between seller dashboard and checkout flows |
| **Axios** | HTTP Client | Promise-based HTTP requests to backend API with interceptor support for auth |
| **React Hook Form** | Form Management | Lightweight form validation with minimal re-renders (critical for mobile) |

**Hosting:** Vercel (automatic deployments from GitHub, global CDN, free tier)

---

### **Backend**
| Technology | Purpose | Justification |
|------------|---------|---------------|
| **Node.js** | Runtime Environment | Non-blocking I/O ideal for payment webhooks and concurrent user sessions |
| **Express.js** | Web Framework | Minimal, flexible framework for building RESTful APIs quickly |
| **JWT** | Authentication | Stateless authentication for seller sessions (token-based, no session storage) |
| **bcrypt** | Password Hashing | Industry-standard hashing for secure credential storage |
| **Joi** | Request Validation | Schema-based validation to prevent malformed data reaching database |
| **node-cron** | Scheduled Tasks | Automated link expiration checks and payment status polling |

**Hosting:** Railway (automatic scaling, PostgreSQL add-on, free tier)

---

### **Database**
| Technology | Purpose | Justification |
|------------|---------|---------------|
| **PostgreSQL** | Primary Database | ACID compliance for payment transactions; strong relational integrity |
| **Prisma ORM** | Database Client | Type-safe query builder, automatic migrations, simplified schema management |

**Hosting:** Supabase (managed PostgreSQL, real-time subscriptions, free tier 500MB)

**Schema Design (Core Tables):**

```sql
-- Users (Sellers)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  business_name VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Checkout Links
CREATE TABLE checkout_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  short_code VARCHAR(10) UNIQUE NOT NULL, -- e.g., "xK9mP2"
  product_name VARCHAR(255) NOT NULL,
  product_price DECIMAL(10, 2) NOT NULL,
  product_image_url TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  expires_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  link_id UUID REFERENCES checkout_links(id),
  buyer_name VARCHAR(255) NOT NULL,
  buyer_phone VARCHAR(20) NOT NULL,
  delivery_address TEXT NOT NULL,
  order_status VARCHAR(50) DEFAULT 'pending', -- pending, paid, shipped, delivered
  payment_reference VARCHAR(100) UNIQUE,
  amount_paid DECIMAL(10, 2),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Payment Transactions
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id),
  paystack_reference VARCHAR(100) UNIQUE NOT NULL,
  status VARCHAR(50) NOT NULL, -- success, failed, pending
  amount DECIMAL(10, 2) NOT NULL,
  payment_method VARCHAR(50), -- card, bank_transfer, ussd
  paid_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

### **Third-Party Integrations**

#### **Payment Processing: Paystack**
- **Why Paystack:** Dominant payment gateway in Nigeria, supports cards/bank transfer/USSD, robust webhook system
- **Integration Points:**
  - Initialize Transaction API (backend creates payment session)
  - Verify Transaction API (confirm payment completion)
  - Webhooks (real-time payment notifications)

**Flow:**
1. Backend calls Paystack Initialize API with order details
2. Returns payment URL to frontend
3. Buyer completes payment on Paystack-hosted page
4. Paystack sends webhook to backend on payment success
5. Backend verifies transaction, updates order status, triggers SMS

#### **SMS Notifications: Twilio**
- **Why Twilio:** Reliable Nigerian SMS delivery, programmable messaging API
- **Use Cases:**
  - Payment confirmation to seller
  - Delivery details to seller for courier handoff
  - Order status updates to buyer (future)

#### **Email Notifications: SendGrid**
- **Why SendGrid:** High deliverability, transactional email templates, free tier (100 emails/day)
- **Use Cases:**
  - Order receipt to buyer
  - Payment confirmation to seller
  - Delivery details backup (in case SMS fails)

---

## System Workflows

### **Workflow 1: Create Checkout Link**

```
Seller (Frontend) → Backend API → Database
      ↓
1. Seller fills form: product name, price, image
2. POST /api/links
   Body: { productName, productPrice, productImage }
   Headers: { Authorization: "Bearer <JWT>" }
3. Backend validates JWT, extracts user_id
4. Generates unique short_code (e.g., "xK9mP2")
5. Inserts record into checkout_links table
6. Returns: { linkUrl: "swiftcart.ng/c/xK9mP2", linkId }
7. Frontend displays shareable link
```

---

### **Workflow 2: Buyer Checkout**

```
Buyer (Frontend) → Backend API → Paystack → Database → SMS
      ↓
1. Buyer clicks link: swiftcart.ng/c/xK9mP2
2. Frontend: GET /api/links/xK9mP2
3. Backend returns: { productName, price, sellerName, imageUrl }
4. Buyer fills form: name, phone, address
5. Buyer clicks "Pay Now"
6. POST /api/checkout
   Body: { linkId, buyerName, buyerPhone, deliveryAddress }
7. Backend:
   a. Creates order record (status: pending)
   b. Calls Paystack Initialize API
   c. Returns { paystackUrl, orderReference }
8. Frontend redirects to Paystack payment page
9. Buyer completes payment
10. Paystack webhook: POST /api/webhooks/paystack
    Body: { event: "charge.success", data: { reference, amount } }
11. Backend:
    a. Verifies webhook signature
    b. Calls Paystack Verify API to confirm
    c. Updates order status → "paid"
    d. Creates payment record
    e. Sends SMS to seller with delivery details
    f. Sends email receipt to buyer
12. Frontend shows success page
```

---

### **Workflow 3: Payment Verification (Webhook)**

```
Paystack → Backend Webhook Endpoint → Database → Notifications
      ↓
1. Paystack POST /api/webhooks/paystack
   Headers: { x-paystack-signature }
   Body: { event, data: { reference, status, amount } }
2. Backend verifies signature using Paystack secret key
3. If valid:
   a. Extract reference, lookup order in database
   b. If order exists and status != "paid":
      - Update order.status = "paid"
      - Create payment record
      - Trigger SMS: "New order from [buyer]. Delivery: [address]"
      - Trigger Email: Order receipt to buyer
   c. Return 200 OK to Paystack
4. If invalid signature: Return 401 Unauthorized
```

---

## Security Considerations

### **Authentication & Authorization**
- **JWT Tokens:** Signed with HMAC SHA-256, 24-hour expiration
- **Password Storage:** bcrypt with 10 salt rounds (industry standard)
- **API Authorization:** Middleware validates JWT on protected routes

### **Payment Security**
- **No Card Storage:** All payment data handled by Paystack (PCI-DSS compliant)
- **Webhook Verification:** HMAC signature validation prevents spoofed webhooks
- **HTTPS Only:** All API endpoints enforce TLS 1.2+

### **Data Protection**
- **Input Validation:** Joi schemas prevent SQL injection, XSS
- **Rate Limiting:** 100 requests/15min per IP (prevents brute force)
- **CORS:** Whitelist only swiftcart.ng origin

### **Link Security**
- **Short Codes:** 6-character alphanumeric (56 billion combinations)
- **Expiration:** Links can be set to expire after X hours (prevents stale links)
- **Soft Delete:** Deactivated links remain in DB for audit trail

---

## API Endpoints (MVP)

### **Authentication**
```
POST /api/auth/register
POST /api/auth/login
POST /api/auth/refresh
```

### **Checkout Links**
```
POST   /api/links              (Create new link - requires auth)
GET    /api/links              (List seller's links - requires auth)
GET    /api/links/:shortCode   (Public - fetch link details)
PATCH  /api/links/:id          (Update link - requires auth)
DELETE /api/links/:id          (Deactivate link - requires auth)
```

### **Orders**
```
POST /api/checkout             (Initialize order + payment)
GET  /api/orders               (List seller's orders - requires auth)
GET  /api/orders/:id           (Get order details - requires auth)
PATCH /api/orders/:id/status   (Update order status - requires auth)
```

### **Webhooks**
```
POST /api/webhooks/paystack    (Payment status updates)
```

---

## Deployment Architecture

### **CI/CD Pipeline**
```
GitHub Push → Automated Tests → Build → Deploy
      ↓
1. Developer pushes to main branch
2. GitHub Actions triggers:
   a. Run ESLint (code quality)
   b. Run Jest tests (unit + integration)
   c. Build frontend (npm run build)
3. If tests pass:
   a. Vercel deploys frontend (automatic)
   b. Railway deploys backend (automatic)
4. Database migrations via Prisma CLI (manual for MVP)
```

### **Environment Variables**
```
# Backend (.env)
DATABASE_URL=postgresql://...
JWT_SECRET=<random-256-bit-key>
PAYSTACK_SECRET_KEY=sk_live_...
PAYSTACK_PUBLIC_KEY=pk_live_...
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...
SENDGRID_API_KEY=...
NODE_ENV=production
```

### **Monitoring & Logging**
- **Error Tracking:** Sentry (frontend + backend exceptions)
- **Logging:** Winston (structured JSON logs)
- **Uptime Monitoring:** UptimeRobot (ping API every 5 minutes)
- **Analytics:** Google Analytics (checkout funnel tracking)

---

## Scalability Considerations

### **Current MVP Limits**
- **Concurrent Users:** ~500 simultaneous checkouts (single Railway instance)
- **Database:** 500MB (Supabase free tier)
- **API Rate Limit:** 100 req/15min per IP

### **Scaling Plan (Post-MVP)**
- **Horizontal Scaling:** Add Railway instances behind load balancer
- **Database:** Upgrade to Supabase Pro (8GB) or migrate to managed AWS RDS
- **Caching:** Redis for frequently accessed links (reduce DB queries)
- **CDN:** Cloudflare for static assets + DDoS protection
- **Async Processing:** Bull queue for SMS/email (prevent webhook timeouts)

---

## Testing Strategy

### **Unit Tests (Jest)**
- Services: Payment validation, link generation logic
- Utilities: Short code generation, JWT signing/verification

### **Integration Tests (Supertest)**
- API endpoints: POST /api/links, POST /api/checkout
- Webhook handling: Mock Paystack webhook payloads

### **End-to-End Tests (Future - Playwright)**
- Full checkout flow: Create link → Share → Pay → Confirm

### **Manual Testing Checklist**
- [ ] Create link with valid/invalid inputs
- [ ] Checkout flow on mobile Safari/Chrome
- [ ] Payment success/failure scenarios
- [ ] Webhook retries (simulate Paystack downtime)
- [ ] SMS/Email delivery confirmation

---

## Development Setup

### **Prerequisites**
- Node.js 18+
- PostgreSQL 14+
- Git

### **Local Installation**
```bash
# Clone repository
git clone https://github.com/yourusername/swiftcart.git
cd swiftcart

# Install dependencies
npm install

# Setup environment variables
cp .env.example .env
# Fill in API keys in .env

# Run database migrations
npx prisma migrate dev

# Start development servers
npm run dev:backend  # Express server on :5000
npm run dev:frontend # React app on :3000
```

---

## Future Technical Enhancements

### **Phase 2 (Q2 2025)**
- **Real-time Dashboard:** WebSocket for live order updates
- **Saved Addresses:** Encrypt stored buyer addresses (AES-256)
- **Multi-tenancy:** Separate database schemas per high-volume seller

### **Phase 3 (Q3-Q4 2025)**
- **Logistics API:** Integrate GIG/Kwik delivery tracking APIs
- **Advanced Analytics:** Time-series database (InfluxDB) for metrics
- **Mobile Apps:** React Native iOS/Android apps for sellers

---

## Technical References

- [Paystack API Docs](https://paystack.com/docs/api/)
- [Prisma ORM Guide](https://www.prisma.io/docs/)
- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [PostgreSQL Performance Tuning](https://wiki.postgresql.org/wiki/Performance_Optimization)

---

## Contributing Guidelines

### **Code Standards**
- ES6+ JavaScript (async/await over callbacks)
- ESLint: Airbnb style guide
- Prettier: 2-space indentation
- Commit messages: Conventional Commits format

### **Branch Strategy**
- `main`: Production-ready code
- `develop`: Integration branch
- `feature/*`: New features
- `fix/*`: Bug fixes

---

**Built by Ogundipe Aisha**
