# PROJECT REQUIREMENTS DOCUMENT
# MINIMODA - E-COMMERCE PLATFORM

**Version:** 1.0  
**Date:** December 2024  
**Status:** Draft  
**Classification:** Confidential

---

## DOCUMENT CONTROL

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | Dec 2024 | Development Team | Initial Draft |

## APPROVAL SIGN-OFF

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Project Sponsor | | | |
| Project Manager | | | |
| Technical Lead | | | |
| Business Owner | | | |

---

## TABLE OF CONTENTS

1. [Executive Summary](#1-executive-summary)
2. [Project Overview](#2-project-overview)
3. [Business Requirements](#3-business-requirements)
4. [Technical Architecture](#4-technical-architecture)
5. [Database Design](#5-database-design)
6. [System Flow & Diagrams](#6-system-flow--diagrams)
7. [Functional Requirements](#7-functional-requirements)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Third-Party Integrations](#9-third-party-integrations)
10. [Project Timeline](#10-project-timeline)
11. [Risk Assessment](#11-risk-assessment)
12. [Appendices](#12-appendices)

---

## 1. EXECUTIVE SUMMARY

### 1.1 Purpose
This document outlines the complete requirements for developing Minimoda, a comprehensive e-commerce platform specializing in children's clothing. The platform will consist of two main Laravel applications: a backend system (API + Admin Panel) and a frontend storefront.

### 1.2 Scope
The project encompasses:
- Full e-commerce functionality with product management
- Integrated payment gateway system
- Real-time shipping calculation and tracking
- Comprehensive admin backoffice
- Customer-facing responsive storefront

### 1.3 Objectives
- Launch a scalable e-commerce platform within 3-4 months
- Support 10,000+ concurrent users
- Process 1,000+ daily transactions
- Achieve <2 second page load time
- Maintain 99.9% uptime

---

## 2. PROJECT OVERVIEW

### 2.1 Business Context
**Company:** Minimoda  
**Industry:** Children's Fashion E-commerce  
**Target Market:** Parents aged 25-40 in Indonesia  
**Product Range:** Children's clothing (0-12 years)

### 2.2 Project Deliverables
1. Backend Application (API + Admin Panel)
2. Frontend Storefront Application
3. Database Architecture
4. Integration with Payment & Shipping Providers
5. Documentation & Training Materials

### 2.3 Stakeholders

| Stakeholder | Role | Responsibility |
|-------------|------|----------------|
| Business Owner | Decision Maker | Final approval, requirements validation |
| Project Manager | Coordinator | Timeline, resource management |
| Development Team | Implementation | System development and testing |
| End Users | Customers | Platform usage and feedback |
| Admin Users | Operations | Daily management and operations |

---

## 3. BUSINESS REQUIREMENTS

### 3.1 Functional Business Requirements

#### 3.1.1 Customer Requirements
- **Account Management**
  - Registration with email/phone verification
  - Social login (Google, Facebook)
  - Profile management
  - Multiple delivery addresses
  - Order history tracking

- **Shopping Experience**
  - Product browsing with filters
  - Advanced search functionality
  - Product comparison
  - Wishlist management
  - Guest checkout option

- **Transaction Process**
  - Shopping cart management
  - Real-time shipping cost calculation
  - Multiple payment methods
  - Order tracking
  - Invoice generation

#### 3.1.2 Admin Requirements
- **Product Management**
  - Bulk product upload/update
  - Inventory tracking
  - Category management
  - Price management
  - Discount/promotion setup

- **Order Processing**
  - Order status management
  - Payment verification
  - Shipping label generation
  - Return/refund processing
  - Customer communication

- **Analytics & Reporting**
  - Sales reports
  - Product performance
  - Customer analytics
  - Financial reports
  - Export capabilities

### 3.2 Business Rules
1. Minimum order value: Rp 50,000
2. Maximum cart items: 50 products
3. Order cancellation allowed within 2 hours
4. Return period: 7 days from delivery
5. Stock reservation: 2 hours for unpaid orders

---

## 4. TECHNICAL ARCHITECTURE

### 4.1 System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     MINIMODA ARCHITECTURE                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐         ┌──────────────┐            │
│  │   Frontend   │ ──API──▶│   Backend    │            │
│  │  Storefront  │         │  API + Admin  │            │
│  └──────────────┘         └──────────────┘            │
│         │                        │                      │
│         ▼                        ▼                      │
│  ┌──────────────┐         ┌──────────────┐            │
│  │     CDN      │         │  PostgreSQL   │            │
│  │   (Assets)   │         │   Database    │            │
│  └──────────────┘         └──────────────┘            │
│                                  │                      │
│                           ┌──────────────┐            │
│                           │    Redis      │            │
│                           │    Cache      │            │
│                           └──────────────┘            │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Technology Stack

#### Backend Application
| Component | Technology | Version | Purpose |
|-----------|------------|---------|----------|
| Framework | Laravel | 11.x | Core backend framework |
| Database | PostgreSQL | 16 | Primary data storage |
| Cache | Redis | 7.x | Session & cache storage |
| Queue | Laravel Horizon | Latest | Background job processing |
| API Auth | Laravel Sanctum | Latest | API authentication |
| Admin Panel | Filament PHP | 3.x | Admin interface (optional) |

#### Frontend Application
| Component | Technology | Version | Purpose |
|-----------|------------|---------|----------|
| Framework | Laravel | 11.x | Frontend framework |
| JavaScript | Inertia.js + Vue | 3.x | Interactive components |
| CSS Framework | Tailwind CSS | 3.x | Styling |
| Build Tool | Vite | 5.x | Asset bundling |
| State Management | Pinia | Latest | Frontend state |

### 4.3 Infrastructure Requirements

#### Development Environment
- Docker with Docker Compose
- PostgreSQL 16
- Redis 7
- Node.js 20 LTS
- PHP 8.3

#### Production Environment
- **Server:** VPS with minimum 8GB RAM
- **OS:** Ubuntu 22.04 LTS
- **Web Server:** Nginx
- **Process Manager:** Supervisor
- **SSL:** Let's Encrypt
- **Monitoring:** New Relic / Datadog

---

## 5. DATABASE DESIGN

### 5.1 Database Schema Overview

```sql
-- Schema Structure
schemas:
  ├── public (main business logic)
  ├── auth (authentication related)
  ├── audit (logging and tracking)
  └── analytics (reporting data)
```

### 5.2 Core Tables Structure

#### Authentication Schema
```
auth.admins
├── id (UUID PRIMARY KEY)
├── name (VARCHAR 255)
├── email (VARCHAR 255 UNIQUE)
├── password (VARCHAR 255)
├── role_id (FK)
├── is_active (BOOLEAN)
├── last_login (TIMESTAMP)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)

auth.customers
├── id (UUID PRIMARY KEY)
├── name (VARCHAR 255)
├── email (VARCHAR 255 UNIQUE)
├── phone (VARCHAR 20)
├── password (VARCHAR 255)
├── email_verified_at (TIMESTAMP)
├── phone_verified_at (TIMESTAMP)
├── is_active (BOOLEAN)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)
```

#### Product Schema
```
public.products
├── id (UUID PRIMARY KEY)
├── sku (VARCHAR 100 UNIQUE)
├── name (VARCHAR 255)
├── slug (VARCHAR 255 UNIQUE)
├── description (TEXT)
├── category_id (FK)
├── attributes (JSONB)
├── meta_data (JSONB)
├── is_active (BOOLEAN)
├── created_by (UUID)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)

public.product_variants
├── id (UUID PRIMARY KEY)
├── product_id (FK)
├── sku_variant (VARCHAR 100)
├── size (VARCHAR 50)
├── color (VARCHAR 50)
├── price (DECIMAL 12,2)
├── stock_quantity (INTEGER)
├── reserved_quantity (INTEGER)
└── attributes (JSONB)
```

#### Order Schema
```
public.orders
├── id (UUID PRIMARY KEY)
├── order_number (VARCHAR 50 UNIQUE)
├── customer_id (FK)
├── status (VARCHAR 50)
├── subtotal (DECIMAL 12,2)
├── shipping_cost (DECIMAL 12,2)
├── discount_amount (DECIMAL 12,2)
├── tax_amount (DECIMAL 12,2)
├── total_amount (DECIMAL 12,2)
├── payment_method (VARCHAR 50)
├── payment_status (VARCHAR 50)
├── shipping_address (JSONB)
├── notes (TEXT)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)
```

### 5.3 PostgreSQL Specific Features

- **UUID Primary Keys:** Using gen_random_uuid() for distributed systems
- **JSONB Columns:** Flexible attributes and metadata storage
- **Full-Text Search:** Using tsvector for product search
- **Partial Indexes:** Optimizing queries on active records
- **Table Partitioning:** For orders table by date range

---

## 6. SYSTEM FLOW & DIAGRAMS

### 6.1 Use Case Diagram

```
┌─────────────────────────────────────────────┐
│              MINIMODA USE CASES             │
├─────────────────────────────────────────────┤
│                                             │
│     Customer                   Admin        │
│        ○                         ○          │
│        │                         │          │
│    ┌───┼───┐                ┌───┼───┐      │
│    │Browse │                │Manage │      │
│    │Product│                │Product│      │
│    └───────┘                └───────┘      │
│    ┌───────┐                ┌───────┐      │
│    │Add to │                │Process│      │
│    │Cart   │                │Orders │      │
│    └───────┘                └───────┘      │
│    ┌───────┐                ┌───────┐      │
│    │Checkout│                │Generate│     │
│    │       │                │Reports │      │
│    └───────┘                └───────┘      │
│    ┌───────┐                ┌───────┐      │
│    │Track  │                │Manage │      │
│    │Order  │                │Users  │      │
│    └───────┘                └───────┘      │
└─────────────────────────────────────────────┘
```

### 6.2 Customer Purchase Flow

```mermaid
stateDiagram-v2
    [*] --> Browse
    Browse --> ProductDetail
    ProductDetail --> AddToCart
    AddToCart --> Cart
    Cart --> Checkout
    Checkout --> SelectShipping
    SelectShipping --> Payment
    Payment --> Processing
    Processing --> Success
    Processing --> Failed
    Failed --> Payment
    Success --> [*]
```

### 6.3 Order Status Flow

```
NEW → PAYMENT_PENDING → PAYMENT_CONFIRMED → PROCESSING → 
SHIPPED → DELIVERED → COMPLETED

Alternative flows:
- PAYMENT_PENDING → CANCELLED (timeout/user action)
- DELIVERED → RETURN_REQUESTED → REFUNDED
```

---

## 7. FUNCTIONAL REQUIREMENTS

### 7.1 Customer Portal Features

#### F-CP-001: User Registration
**Priority:** High  
**Description:** Users can register using email or phone number  
**Acceptance Criteria:**
- Email/phone validation
- Password strength requirements
- Verification process
- Social login option

#### F-CP-002: Product Catalog
**Priority:** High  
**Description:** Browse and search products  
**Acceptance Criteria:**
- Filter by category, size, color, price
- Sort by relevance, price, newest
- Product quick view
- Image zoom functionality

#### F-CP-003: Shopping Cart
**Priority:** High  
**Description:** Manage shopping cart  
**Acceptance Criteria:**
- Add/remove items
- Update quantities
- Apply discount codes
- Save for later

#### F-CP-004: Checkout Process
**Priority:** High  
**Description:** Complete purchase transaction  
**Acceptance Criteria:**
- Guest checkout option
- Address validation
- Shipping method selection
- Payment processing
- Order confirmation

### 7.2 Admin Portal Features

#### F-AP-001: Dashboard
**Priority:** High  
**Description:** Overview of business metrics  
**Acceptance Criteria:**
- Real-time sales data
- Order statistics
- Low stock alerts
- Recent activities

#### F-AP-002: Product Management
**Priority:** High  
**Description:** Complete product lifecycle management  
**Acceptance Criteria:**
- CRUD operations
- Bulk import/export
- Variant management
- Image management
- SEO metadata

#### F-AP-003: Order Management
**Priority:** High  
**Description:** Process and track orders  
**Acceptance Criteria:**
- Order status updates
- Shipping label generation
- Invoice generation
- Refund processing

#### F-AP-004: Customer Management
**Priority:** Medium  
**Description:** Manage customer accounts  
**Acceptance Criteria:**
- View customer details
- Order history
- Communication log
- Account status management

---

## 8. NON-FUNCTIONAL REQUIREMENTS

### 8.1 Performance Requirements

| Metric | Target | Measurement |
|--------|--------|-------------|
| Page Load Time | < 2 seconds | 95th percentile |
| API Response Time | < 500ms | Average |
| Database Query Time | < 100ms | 95th percentile |
| Concurrent Users | 10,000+ | Peak capacity |
| Transactions/Day | 1,000+ | Processing capacity |

### 8.2 Security Requirements

- **Authentication:** Multi-factor authentication for admin
- **Encryption:** TLS 1.3 for data in transit
- **Data Protection:** AES-256 for sensitive data
- **PCI DSS:** Compliance for payment processing
- **OWASP:** Top 10 vulnerability mitigation
- **Audit Logs:** All admin actions logged
- **Session Management:** Secure session handling

### 8.3 Reliability Requirements

- **Uptime:** 99.9% availability (excluding maintenance)
- **Backup:** Daily automated backups
- **Recovery Time:** RTO < 4 hours, RPO < 1 hour
- **Failover:** Automatic failover for critical services

### 8.4 Scalability Requirements

- **Horizontal Scaling:** Application servers
- **Database Scaling:** Read replicas for reporting
- **Caching Strategy:** Multi-layer caching
- **CDN:** Global content delivery
- **Queue System:** Asynchronous job processing

---

## 9. THIRD-PARTY INTEGRATIONS

### 9.1 Payment Gateway

#### Primary: Midtrans
**Integration Type:** Server-to-Server API  
**Supported Methods:**
- Bank Transfer (All major banks)
- E-Wallets (GoPay, OVO, DANA, LinkAja)
- Credit/Debit Cards
- Convenience Store
- QRIS

**Requirements:**
- Merchant account setup
- API credentials (Server Key, Client Key)
- Webhook endpoint for notifications
- 3D Secure implementation

### 9.2 Shipping Integration

#### RajaOngkir Pro
**Integration Type:** REST API  
**Features:**
- Real-time shipping cost calculation
- Multiple courier support (JNE, J&T, SiCepat, Anteraja)
- Tracking API
- Waybill generation

**Requirements:**
- Pro account subscription
- API Key
- Origin city configuration
- Weight calculation system

### 9.3 Communication Services

#### WhatsApp Business API (Fonnte/Wablas)
**Use Cases:**
- Order confirmation
- Shipping notifications
- Abandoned cart reminders
- Customer support

#### Email Service (Amazon SES)
**Use Cases:**
- Transactional emails
- Order invoices
- Password reset
- Marketing campaigns

### 9.4 Analytics

#### Google Analytics 4
- E-commerce tracking
- User behavior analysis
- Conversion tracking
- Custom events

#### Facebook Pixel
- Retargeting campaigns
- Conversion tracking
- Audience building

---

## 10. PROJECT TIMELINE

### 10.1 Development Phases

```
┌──────────────────────────────────────────────────────┐
│                  PROJECT TIMELINE                     │
├──────────────────────────────────────────────────────┤
│                                                       │
│ Phase 0: Setup & Planning          [Week 1-2]       │
│ ├── Environment setup                                │
│ ├── Database design                                  │
│ └── Architecture finalization                        │
│                                                       │
│ Phase 1: Core Development          [Week 3-8]       │
│ ├── Authentication system                            │
│ ├── Product management                               │
│ ├── Shopping cart & checkout                         │
│ └── Admin dashboard                                  │
│                                                       │
│ Phase 2: Integration               [Week 9-11]      │
│ ├── Payment gateway                                  │
│ ├── Shipping services                                │
│ └── Communication services                           │
│                                                       │
│ Phase 3: Testing & Optimization   [Week 12-13]      │
│ ├── Unit & integration testing                       │
│ ├── Performance optimization                         │
│ └── Security audit                                   │
│                                                       │
│ Phase 4: Deployment               [Week 14]         │
│ ├── Production setup                                 │
│ ├── Data migration                                   │
│ └── Go-live preparation                              │
│                                                       │
│ Phase 5: Stabilization           [Week 15-16]       │
│ ├── Bug fixes                                        │
│ ├── Performance monitoring                           │
│ └── User training                                    │
└──────────────────────────────────────────────────────┘
```

### 10.2 Milestones

| Milestone | Date | Deliverable |
|-----------|------|-------------|
| M1: Project Kickoff | Week 1 | Setup complete |
| M2: Alpha Release | Week 8 | Core features ready |
| M3: Beta Release | Week 11 | Integrations complete |
| M4: UAT Complete | Week 13 | Testing signoff |
| M5: Go Live | Week 14 | Production launch |

### 10.3 Resource Allocation

| Role | Allocation | Duration |
|------|------------|----------|
| Project Manager | 50% | 16 weeks |
| Lead Developer | 100% | 16 weeks |
| Backend Developer (2) | 100% | 14 weeks |
| Frontend Developer | 100% | 14 weeks |
| UI/UX Designer | 50% | 8 weeks |
| QA Tester | 100% | 6 weeks |
| DevOps Engineer | 50% | 16 weeks |

---

## 11. RISK ASSESSMENT

### 11.1 Risk Matrix

| Risk | Probability | Impact | Mitigation Strategy |
|------|------------|--------|---------------------|
| **Technical Risks** |
| Payment gateway downtime | Medium | High | Implement fallback gateway |
| Database performance issues | Low | High | Optimize queries, add indexes |
| Security breach | Low | Critical | Regular security audits |
| **Business Risks** |
| Scope creep | High | Medium | Clear change management process |
| Budget overrun | Medium | High | Phased development approach |
| Timeline delay | Medium | Medium | Buffer time in schedule |
| **Operational Risks** |
| Staff turnover | Low | High | Knowledge documentation |
| Third-party API changes | Medium | Medium | Version control, monitoring |

### 11.2 Contingency Plans

1. **Payment Gateway Failure**
   - Secondary payment provider ready
   - Manual payment verification process
   - Clear communication to customers

2. **High Traffic Spike**
   - Auto-scaling configuration
   - CDN implementation
   - Queue system for order processing

3. **Data Loss**
   - Daily automated backups
   - Point-in-time recovery
   - Disaster recovery site

---

## 12. APPENDICES

### Appendix A: Glossary

| Term | Definition |
|------|------------|
| API | Application Programming Interface |
| CDN | Content Delivery Network |
| CRUD | Create, Read, Update, Delete |
| ERD | Entity Relationship Diagram |
| JSONB | JSON Binary (PostgreSQL data type) |
| MVP | Minimum Viable Product |
| PCI DSS | Payment Card Industry Data Security Standard |
| REST | Representational State Transfer |
| SKU | Stock Keeping Unit |
| UAT | User Acceptance Testing |
| UUID | Universally Unique Identifier |

### Appendix B: References

1. Laravel Documentation: https://laravel.com/docs
2. PostgreSQL Documentation: https://www.postgresql.org/docs/
3. Midtrans API Docs: https://docs.midtrans.com
4. RajaOngkir API Docs: https://rajaongkir.com/dokumentasi
5. OWASP Top 10: https://owasp.org/www-project-top-ten/

### Appendix C: Document Templates

- User Story Template
- Test Case Template
- Bug Report Template
- Change Request Form
- Deployment Checklist

### Appendix D: Contact Information

| Role | Name | Email | Phone |
|------|------|-------|-------|
| Project Sponsor | TBD | | |
| Project Manager | TBD | | |
| Technical Lead | TBD | | |
| Business Analyst | TBD | | |

---

## DOCUMENT REVISION HISTORY

| Version | Date | Changes | Author |
|---------|------|---------|---------|
| 1.0 | Dec 2024 | Initial document creation | Development Team |
| | | | |

---

## SIGN-OFF

By signing below, stakeholders acknowledge they have reviewed and approved this Project Requirements Document:

**Business Owner:**  
Signature: _________________________ Date: _____________

**Technical Lead:**  
Signature: _________________________ Date: _____________

**Project Manager:**  
Signature: _________________________ Date: _____________

---

**END OF DOCUMENT**

*This document is confidential and proprietary to Minimoda. Distribution is limited to authorized personnel only.*

---

© 2024 Minimoda. All Rights Reserved.