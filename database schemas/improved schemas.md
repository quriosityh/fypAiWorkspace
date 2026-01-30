---
aliases:
  - updated schemas
---
Looking at your database schema from the Month 1 plan compared to your project specification, I found **several critical issues** that need correction. Here's my detailed review:

## 🚨 **CRITICAL SCHEMA PROBLEMS IDENTIFIED**

### **1. MISSING CORE TABLES (High Priority)**
Your schema is missing essential tables required for the MVP:

```sql
-- ❌ MISSING FROM YOUR SCHEMA:
-- 1. Reviews table (two-way reviews are CORE feature)
-- 2. Messages table (real-time messaging is CORE feature)  
-- 3. User profiles/relationships (following, blocking)
-- 4. Notifications table
-- 5. Favorites/wishlist table
```

### **2. AUTHENTICATION MISMATCH**
**Problem:** Your `users` table has `clerk_id` but Clerk handles all auth - this creates duplication.

**Solution:** Remove Clerk-specific fields and treat Clerk as external identity provider:

```sql
-- ✅ CORRECTED users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- ❌ REMOVE: clerk_id, email, avatar_url (Clerk manages these)
  -- ✅ KEEP: Platform-specific fields only
  display_name TEXT, -- Different from Clerk's name
  bio TEXT,
  phone TEXT, -- For Pakistani users
  city TEXT NOT NULL, -- Required for location-based features
  verification_status TEXT DEFAULT 'unverified' CHECK (verification_status IN ('unverified', 'pending', 'verified')),
  average_rating DECIMAL(3,2) DEFAULT 0.0,
  total_reviews INTEGER DEFAULT 0,
  response_rate DECIMAL(5,2) DEFAULT 0.0, -- Important for trust
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ✅ Add user_verification table for Pakistani ID verification
CREATE TABLE user_verifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  cnic_number TEXT, -- Pakistani National ID
  cnic_front_url TEXT,
  cnic_back_url TEXT, 
  selfie_url TEXT,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'verified', 'rejected')),
  verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### **3. LISTINGS SCHEMA INCOMPLETE**
**Missing critical Pakistani marketplace fields:**

```sql
-- ✅ CORRECTED listings table
CREATE TABLE listings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  category_id INTEGER NOT NULL REFERENCES categories(id),
  
  -- ✅ IMPROVED: Pakistani pricing structure
  daily_rate INTEGER NOT NULL CHECK (daily_rate > 0),
  weekly_rate INTEGER, -- Discount for weekly rental
  monthly_rate INTEGER, -- Discount for monthly rental
  security_deposit INTEGER DEFAULT 0,
  
  -- ✅ IMPROVED: Pakistani location fields
  province TEXT NOT NULL CHECK (province IN ('Punjab', 'Sindh', 'KPK', 'Balochistan', 'Islamabad', 'Gilgit-Baltistan', 'AJK')),
  city TEXT NOT NULL,
  area TEXT, -- Specific area/neighborhood (DHA, Gulberg, etc.)
  exact_location TEXT, -- Optional for privacy
  
  -- ✅ NEW: Pakistani-specific fields
  delivery_available BOOLEAN DEFAULT FALSE,
  delivery_fee INTEGER DEFAULT 0,
  min_rental_days INTEGER DEFAULT 1,
  max_rental_days INTEGER DEFAULT 30,
  
  -- ✅ IMPROVED: Status for Pakistani workflow
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'inactive', 'blocked', 'under_review')),
  
  -- ✅ KEEP your good fields:
  specs JSONB DEFAULT '{}',
  view_count INTEGER DEFAULT 0,
  booking_count INTEGER DEFAULT 0,
  average_rating DECIMAL(3,2) DEFAULT 0.0,
  review_count INTEGER DEFAULT 0,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### **4. MISSING CORE FEATURE TABLES**

```sql
-- ✅ REVIEWS table (Two-way reviews - CORE FEATURE)
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  reviewer_id UUID NOT NULL REFERENCES users(id),
  reviewee_id UUID NOT NULL REFERENCES users(id),
  listing_id UUID NOT NULL REFERENCES listings(id),
  
  rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
  comment TEXT,
  
  -- ✅ Airbnb-style two-way reviews
  type TEXT NOT NULL CHECK (type IN ('owner_to_renter', 'renter_to_owner')),
  status TEXT DEFAULT 'hidden' CHECK (status IN ('hidden', 'visible')), -- Both reviews visible together
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(booking_id, reviewer_id, type) -- One review per booking per type
);

-- ✅ MESSAGES table (Real-time messaging - CORE FEATURE)
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL, -- Will reference conversations table
  sender_id UUID NOT NULL REFERENCES users(id),
  recipient_id UUID NOT NULL REFERENCES users(id),
  listing_id UUID REFERENCES listings(id),
  
  content TEXT NOT NULL,
  message_type TEXT DEFAULT 'text' CHECK (message_type IN ('text', 'image', 'system')),
  image_url TEXT,
  
  read_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ✅ CONVERSATIONS table (Organize messages)
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  participant1_id UUID NOT NULL REFERENCES users(id),
  participant2_id UUID NOT NULL REFERENCES users(id),
  listing_id UUID REFERENCES listings(id),
  
  last_message_at TIMESTAMPTZ DEFAULT NOW(),
  last_message_preview TEXT,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(participant1_id, participant2_id, listing_id)
);

-- ✅ FAVORITES table (Wishlist feature)
CREATE TABLE favorites (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  listing_id UUID NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, listing_id)
);
```

### **5. BOOKINGS SCHEMA IMPROVEMENTS**

```sql
-- ✅ CORRECTED bookings table
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  listing_id UUID NOT NULL REFERENCES listings(id),
  renter_id UUID NOT NULL REFERENCES users(id),
  
  -- ✅ Pakistani rental specifics
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  total_days INTEGER NOT NULL,
  
  -- ✅ Pricing breakdown (transparent for Pakistani users)
  daily_rate INTEGER NOT NULL,
  total_amount INTEGER NOT NULL,
  security_deposit INTEGER DEFAULT 0,
  platform_fee INTEGER DEFAULT 0, -- Commission (10-15%)
  owner_payout INTEGER, -- total_amount - platform_fee
  
  -- ✅ MVP booking flow (optimistic booking)
  status TEXT DEFAULT 'pending' CHECK (status IN (
    'pending',      -- Request sent
    'confirmed',    -- Owner accepted
    'rejected',     -- Owner rejected  
    'cancelled',    -- Either party cancelled
    'completed'     -- Rental completed
  )),
  
  -- ✅ Pakistani payment context
  payment_method TEXT DEFAULT 'cash', -- MVP: cash only
  payment_status TEXT DEFAULT 'pending' CHECK (payment_status IN (
    'pending',      -- Will pay cash
    'paid_cash',    -- Paid in person
    'commission_pending', -- Owner owes commission
    'commission_paid', -- Commission collected
    'refunded'
  )),
  
  -- ✅ Communication fields
  renter_notes TEXT,
  owner_notes TEXT,
  
  -- ✅ Meeting logistics (Pakistani context)
  meeting_location TEXT,
  meeting_instructions TEXT,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  CHECK (end_date > start_date),
  CHECK (total_days = (end_date - start_date) + 1)
);
```

## 📊 **CORRECTED MIGRATION ORDER**

```sql
-- Migration 001: Core tables
001_create_users.sql
002_create_categories.sql
003_create_listings.sql
004_create_listing_images.sql

-- Migration 002: Booking system  
005_create_bookings.sql
006_create_unavailable_dates.sql

-- Migration 003: Core features (REQUIRED FOR MVP)
007_create_reviews.sql
008_create_conversations.sql
009_create_messages.sql
010_create_favorites.sql

-- Migration 004: Pakistani-specific features
011_create_user_verifications.sql
012_create_notifications.sql
```

##