---
share_link: https://share.note.sx/wpgl9xfj#1L+kqBHRfWyQ1R1nruKZr8oNhPYHO7nXI3I15lrSgWU

---
## Government Queen Mary College, Lahore

**University of the Punjab, Lahore**

## Final Year Project – Deliverable 2

### Requirement Specification & Design Document

---

## StuFlux: Peer-to-Peer Rental Marketplace

**Project Supervisor:** Miss Rida

**Project Members:**

| Roll No. | Name             | Email                       |
| -------- | ---------------- | --------------------------- |
| 093080   | Hajra Mubasher   | hajramubasher8110@gmail.com |
| 093078   | Wajeeha Shahzaib | wajeehashahzeb56@gmail.com  |

**Date:** January 29, 2026

---

### Contents

1. Analysis Phase Use Case Diagram
2. Domain Model
3. Sequence Diagrams for High-Level Use Cases
4. Data Model Using Entity-Relationship Diagram (ERD)
5. Design Class Diagram Using Design Patterns

---

## 1. Analysis Phase Use Case Diagram

The analysis-level use case diagram shows how each actor completes core tasks and how shared behavior is factored via include/extend relations.

### System Actors

- **Guest User**: Unauthenticated visitor who can browse listings and sign up
- **Authenticated User**: Base role after login; generalizes into renter and owner
- **Renter**: Authenticated user who books items from others
- **Owner**: Authenticated user who lists items for rental

### Use Case Diagram

[[usecaseDiagram]]
![[usecaseDiagram.png]]
**Figure 1:** Analysis-level use case diagram showing actor generalization (Authenticated User → Renter/Owner) and include/extend relations between booking, calendar updates, and reviews.

### Key Use Case Descriptions

1. **UC1: Sign Up** - Create account (email/Google); profile stub created.
2. **UC2: Login** - Authenticate and receive session/JWT.
3. **UC3: Browse Listings** - View active listings with filters.
4. **UC4: Search Items** - Keyword search with instant results.
5. **UC5: View Item Details** - Listing detail with photos, availability, owner.
6. **UC6: Create Listing** - Multi-step form: basics, pricing, photos, publish.
7. **UC7: Manage Listing** - Edit details, photos, status.
8. **UC8: Manage Profile** - Update profile and history.
9. **UC9: Request Booking** - Select dates, price calc, create pending booking.
10. **UC10: Check Availability** - Validate dates before booking.
11. **UC11: Confirm/Reject Booking** - Owner decision blocks/frees dates.
12. **UC12: Update Calendar** - Auto-reflect confirmed bookings and blocks.
13. **UC13: Complete Booking** - Close rental and update records.
14. **UC14: Send Message** - Real-time chat for listings/bookings.
15. **UC15: Write Review** - Two-way rating (1-5) and text after completion.
---
## 2. Domain Model

The domain model groups concepts into aggregates: `User` (with verification/reputation), `Listing` (pricing, availability, photos, category), `Booking` (rental period, payment, status), `Conversation`/`Message`, `Review` (rating value object), and `Dispute`. Domain services (`BookingService`, `ReviewService`, `NotificationService`) handle cross-aggregate rules.
[[domain diagram]]

![[domaindiagram.svg]]

**Figure 2:** Domain model showing aggregates, value objects, and domain services with their associations.

### Key Relationships

- User owns listings, rents bookings, writes/receives reviews, and joins conversations.
- Listing belongs to a category; has pricing/availability/photos; links to bookings and conversations.
- Booking references one listing and renter/owner; can spawn two reviews and an optional dispute.
- Review targets a user about a booking with a rating value object.
- Conversation ties renter/owner around a listing; holds messages.
- Domain services enforce booking rules, reputation updates, and notifications.

---

## 3. Sequence Diagrams for High-Level Use Cases (Minimal Set)

Minimal flows aligned to aggregates and tables: listing lifecycle, booking lifecycle, messaging, and reviews.

### 3.1 Listing Creation and Publication

[[listinglifecyclesq]]
![[correct listing lifecyle sq diagram-2026-01-30-104016.svg]]
**Figure 3:** Listing lifecycle from draft creation to photo upload and publication using LISTINGS and LISTING_PHOTOS.

### 3.2 Booking Request and Owner Decision

[[bookingflow sq]]
![[bookingflow.svg]]
**Figure 4:** Booking lifecycle from renter request to owner decision with notifications stored in NOTIFICATIONS.

### 3.3 Real-time Messaging (Conversations + Messages)

[[mesagingsqcode]]
![[messaging.svg]]
**Figure 5:** Messaging flow tied to CONVERSATIONS and MESSAGES tables with SSE delivery and read receipts.

### 3.4 Review Submission and Reputation Updates

[[reviewsq]]
![[reviewsq.svg]]
**Figure 6:** Review lifecycle after booking completion with eligibility checks, single-review enforcement per booking, and SSE notifications to the target user alongside reputation aggregate updates.

---

## 4. Data Model Using Entity-Relationship Diagram (ERD)

The ERD specifies the database schema for PostgreSQL, showing tables, attributes, data types, and relationships with cardinality constraints.
[[erd diagram]]
![[erd.svg]]

**Figure 7:** Entity-Relationship Diagram showing database schema with tables, attributes, primary keys (PK), foreign keys (FK), and unique keys (UK).

### Key Database Design Decisions

- 3NF schema with lookup tables for categories and listing images; blocked_dates stores single-day blocks.
- UUID keys, JSONB specs, DECIMAL money, ENUM statuses, timestamptz for auditability.
- Indexes on FK columns, bookings date range, blocked_dates, and unread messages.
- Constraints: NOT NULL on critical fields; rating 1-5; end_date > start_date; unique emails/slugs.
- RLS: users access only their data; listings limited to owners; messages scoped to participants; public read for active listings/reviews.

---

## 5. Design Class Diagram Using Design Patterns

The design class diagram shows platform layers with controllers, middleware, services/policies, repositories, entities, and adapters.

[[designclass]]
![[classdiagram-2026-01-30-143359.svg]]
**Figure 8:** Design class diagram applying MVC (controllers/services), Repository, Adapter (Clerk/Cloudinary/Email/Cache), Strategy (availability policy), and Singleton (SSE hub) patterns across the StuFlux platform layers.

### Design Patterns Applied

**1. Model-View-Controller (MVC)**
- Controllers validate requests and delegate to services; services host business logic; models carry validation.

**2. Repository Pattern**
- Repositories isolate data access for users, listings, bookings, messages, reviews, notifications.

**3. Adapter Pattern**
- Clerk/Cloudinary/Email/Cache adapters wrap external APIs behind stable interfaces.

**4. Singleton Pattern**
- `SSEHub` centralizes real-time delivery to users.

**5. Strategy Pattern**
- `AvailabilityPolicy` encapsulates date validation and blocking logic.

**6. Middleware Chain**
- `AuthMiddleware` and `RateLimitMiddleware` provide cross-cutting enforcement.

### Class Responsibilities

**Controllers**: Accept requests and return responses via services.
**Services**: Execute business logic and coordinate repositories.
**Repositories**: Encapsulate CRUD and query logic.
**Models**: Define structure, validation, derived values.
**Middleware**: Handle authentication, throttling, and logging.


---

## Summary

This document outlines StuFlux requirements and design: actors (Guest, Renter, Owner), fifteen core use cases, and aggregates for users, listings, bookings, conversations/messages, reviews, and disputes. Sequence diagrams cover listing creation, booking decision, messaging, and reviews; the ERD captures a normalized PostgreSQL schema with availability and notifications.

The design class diagram shows a layered Next.js + Express stack using MVC, repositories, adapters, an availability strategy, and an SSE singleton hub. Next.js handles read-heavy flows; Express manages writes with Clerk-authenticated APIs, RLS-protected data, and real-time delivery.

---

**End of Document**
