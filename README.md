# PairUp 🤝

> A location-based social companion platform — find people to attend events, explore nightlife, and share experiences with. Not a dating app.

![PairUp Dashboard](https://img.shields.io/badge/Status-In%20Development-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)
![Stack](https://img.shields.io/badge/Stack-Next.js%20%7C%20Node.js%20%7C%20MongoDB-brightgreen?style=flat-square)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Deployment](#deployment)
- [Roadmap](#roadmap)
- [Safety & Disclaimer](#safety--disclaimer)
- [License](#license)

---

## Overview

**PairUp** is a social matching platform that connects people looking for companions for real-world activities — live concerts, clubbing nights, food crawls, travel, and more. Users create a profile, set their interests, discover nearby people, and match with those who share mutual interest. Once matched, they can chat in real time and plan meetups.

PairUp is **strictly a social companionship platform**. It has no affiliation with dating services, escort services, or any form of transactional companionship.

---

## Features

### Core
- 🔐 **Auth** — Email/password signup with bcrypt hashing; JWT-based sessions; optional mobile OTP verification
- 👤 **Profiles** — Name, gender, bio, city, interests (multi-select), photo gallery
- 💡 **Discovery** — Swipe-style card deck; like/pass/super-like actions
- 💞 **Matching** — Mutual likes create a match; matched users unlock chat
- 💬 **Real-time Chat** — Socket.io powered messaging; emoji support; read receipts; typing indicators
- 📍 **Nearby Users** *(Premium)* — Geolocation-based discovery within a configurable radius (5–20 km)

### Premium (₹100/month via Razorpay)
- Unlimited likes
- Nearby user discovery
- Priority visibility in feeds
- Advanced interest filters
- Premium badge on profile

### Safety
- Report & block users
- Image moderation (AWS Rekognition)
- Profile verification badge
- Account deactivation / data deletion (GDPR-compliant)
- Platform disclaimer on first login

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, Tailwind CSS, Framer Motion |
| Backend | Node.js, Express.js |
| Database | MongoDB Atlas (Mongoose ODM) |
| Real-time | Socket.io + Redis Pub/Sub |
| Auth | JWT, bcrypt, Twilio (OTP) |
| Cache & Presence | Redis Cloud |
| Search | Elasticsearch (geo-distance queries) |
| Media | AWS S3 + CloudFront CDN |
| Payments | Razorpay |
| Notifications | Firebase Cloud Messaging |
| Deployment | Vercel (frontend), Railway (backend) |

---

## System Architecture

```
Client (Next.js)
    │
    ▼
API Gateway — JWT Auth middleware, rate limiting
    │
    ├── Auth Service        (register, login, OTP)
    ├── User Service        (profiles, photos, geo)
    ├── Match Engine        (swipe, mutual like detection)
    ├── Chat Service        (message history, REST)
    ├── Payment Service     (Razorpay, subscriptions)
    ├── Safety Service      (report, block, moderation)
    ├── Notification Svc    (FCM push, email, SMS)
    └── Media Service       (S3 upload, CDN URLs)
            │
            ▼
    Socket.io ←→ Redis Pub/Sub   (real-time presence & chat)
            │
            ▼
    MongoDB ── Redis Cache ── Elasticsearch
            │
            ▼
    AWS S3 / CloudFront · Vercel · Firebase
```

---

## Database Schema

### `users`
```js
{
  _id:           ObjectId,
  full_name:     String,
  email:         String,       // unique
  password_hash: String,
  mobile:        String,
  gender:        String,       // 'male' | 'female' | 'other'
  bio:           String,
  city:          String,
  location: {
    type:        'Point',
    coordinates: [lng, lat]    // GeoJSON for 2dsphere index
  },
  interests:     [String],
  photos:        [String],     // S3 URLs
  status:        String,       // 'online' | 'offline'
  is_premium:    Boolean,
  is_verified:   Boolean,
  is_active:     Boolean,
  created_at:    Date
}
```

### `matches`
```js
{
  _id:        ObjectId,
  user_a_id:  ObjectId,  // ref: users
  user_b_id:  ObjectId,  // ref: users
  status:     String,    // 'active' | 'unmatched'
  matched_at: Date
}
```

### `likes`
```js
{
  _id:        ObjectId,
  from_user:  ObjectId,  // ref: users
  to_user:    ObjectId,  // ref: users
  action:     String,    // 'like' | 'pass' | 'super'
  created_at: Date
}
```

### `messages`
```js
{
  _id:       ObjectId,
  match_id:  ObjectId,  // ref: matches
  sender_id: ObjectId,  // ref: users
  content:   String,
  read:      Boolean,
  sent_at:   Date
}
```

### `subscriptions`
```js
{
  _id:        ObjectId,
  user_id:    ObjectId,  // ref: users
  plan:       String,    // 'free' | 'premium'
  payment_id: String,    // Razorpay payment ID
  status:     String,    // 'active' | 'expired' | 'cancelled'
  starts_at:  Date,
  expires_at: Date
}
```

### `reports`
```js
{
  _id:         ObjectId,
  reporter_id: ObjectId,  // ref: users
  reported_id: ObjectId,  // ref: users
  reason:      String,
  status:      String,    // 'pending' | 'reviewed' | 'resolved'
  created_at:  Date
}
```

### `blocks`
```js
{
  _id:        ObjectId,
  blocker_id: ObjectId,  // ref: users
  blocked_id: ObjectId,  // ref: users
  created_at: Date
}
```

---

## API Endpoints

### Auth  `POST /api/auth/`
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/register` | Create account |
| POST | `/login` | Get JWT token |
| POST | `/verify-otp` | Verify mobile OTP |
| POST | `/refresh` | Refresh access token |
| POST | `/logout` | Invalidate session |

### Users  `/api/users/`
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/me` | Get own profile |
| PUT | `/me` | Update profile |
| POST | `/me/photos` | Upload photo to S3 |
| DELETE | `/me/photos/:id` | Remove a photo |
| GET | `/nearby` | 🔒 Premium — nearby users |
| GET | `/discover` | Discovery feed (paginated) |

### Matching  `/api/matches/`
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/like/:id` | Like a user |
| POST | `/pass/:id` | Pass on a user |
| GET | `/` | Get all matches |
| DELETE | `/:id` | Unmatch |

### Chat  `/api/chat/`
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/:matchId` | Fetch message history |
| POST | `/:matchId` | Send a message |
| PUT | `/:matchId/read` | Mark messages as read |

### Premium  `/api/premium/`
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/plans` | List available plans |
| POST | `/subscribe` | Create Razorpay order |
| POST | `/webhook` | Razorpay webhook handler |
| GET | `/status` | Check premium status |

### Safety  `/api/safety/`
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/report/:id` | Report a user |
| POST | `/block/:id` | Block a user |
| DELETE | `/block/:id` | Unblock a user |
| GET | `/blocked` | Get block list |

> 🔒 = Premium feature. All routes except `/auth/*` require `Authorization: Bearer <token>` header.

---

## Project Structure

```
pairup/
├── frontend/                  # Next.js app
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   ├── register/
│   │   │   └── onboarding/
│   │   └── (app)/
│   │       ├── discover/
│   │       ├── matches/
│   │       ├── chat/[matchId]/
│   │       ├── profile/
│   │       └── premium/
│   ├── components/
│   │   ├── auth/
│   │   ├── profile/           # ProfileCard, PhotoGallery, InterestTags
│   │   ├── discover/          # SwipeCard, SwipeDeck, FilterPanel
│   │   ├── chat/              # ChatList, ChatWindow, MessageBubble
│   │   ├── premium/           # PlanCard, PaymentForm, PremiumBadge
│   │   └── shared/            # Navbar, Avatar, Modal, Toast
│   ├── hooks/
│   │   ├── useAuth.js
│   │   ├── useSocket.js
│   │   ├── useLocation.js
│   │   ├── useMatches.js
│   │   └── useChat.js
│   ├── store/                 # Redux slices
│   │   ├── authSlice.js
│   │   ├── matchSlice.js
│   │   └── chatSlice.js
│   └── lib/
│       ├── api.js             # Axios instance
│       └── socket.js          # Socket.io client
│
├── backend/                   # Express API
│   ├── src/
│   │   ├── routes/
│   │   │   ├── auth.js
│   │   │   ├── users.js
│   │   │   ├── matches.js
│   │   │   ├── chat.js
│   │   │   ├── premium.js
│   │   │   └── safety.js
│   │   ├── models/            # Mongoose schemas
│   │   ├── middleware/
│   │   │   ├── auth.js        # JWT verification
│   │   │   ├── premium.js     # Premium gate
│   │   │   └── rateLimit.js
│   │   ├── services/
│   │   │   ├── matchService.js
│   │   │   ├── socketService.js
│   │   │   └── paymentService.js
│   │   └── server.js
│   └── package.json
│
├── README.md
└── .env.example
```

---

## Getting Started

### Prerequisites
- Node.js 18+
- MongoDB Atlas account (free tier works)
- Redis Cloud account (free tier works)

### 1. Clone the repo
```bash
git clone https://github.com/yourusername/pairup.git
cd pairup
```

### 2. Install dependencies
```bash
# Frontend
cd frontend && npm install

# Backend
cd ../backend && npm install
```

### 3. Set up environment variables
```bash
cp .env.example .env
# Fill in all values (see Environment Variables section)
```

### 4. Run in development
```bash
# Terminal 1 — backend
cd backend && npm run dev

# Terminal 2 — frontend
cd frontend && npm run dev
```

App runs at `http://localhost:3000` (frontend) and `http://localhost:5000` (backend).

---

## Environment Variables

Create a `.env` file in `/backend`:

```env
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/pairup

# Auth
JWT_SECRET=your_super_secret_key_here
JWT_EXPIRES_IN=7d

# Redis
REDIS_URL=redis://default:<pass>@<host>:<port>

# AWS S3
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_REGION=ap-south-1
S3_BUCKET_NAME=pairup-media

# Razorpay
RAZORPAY_KEY_ID=rzp_live_xxxx
RAZORPAY_KEY_SECRET=your_secret
RAZORPAY_WEBHOOK_SECRET=your_webhook_secret

# Twilio (OTP)
TWILIO_ACCOUNT_SID=ACxxxx
TWILIO_AUTH_TOKEN=xxxx
TWILIO_PHONE_NUMBER=+1xxxxxxxxxx

# Firebase (push notifications)
FIREBASE_PROJECT_ID=pairup-app
FIREBASE_PRIVATE_KEY=-----BEGIN PRIVATE KEY-----\n...
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@pairup-app.iam.gserviceaccount.com
```

---

## Deployment

### Frontend → Vercel
```bash
cd frontend
npx vercel --prod
```
Set all `NEXT_PUBLIC_*` environment variables in the Vercel dashboard.

### Backend → Railway
1. Push backend to GitHub
2. Create a new project at [railway.app](https://railway.app)
3. Connect your repo and add environment variables
4. Railway auto-deploys on every push to `main`

### Database → MongoDB Atlas
1. Create a free M0 cluster at [mongodb.com/atlas](https://www.mongodb.com/atlas)
2. Whitelist `0.0.0.0/0` for Railway (or use a static IP)
3. Create a database user and copy the connection string to `MONGODB_URI`

---

## Roadmap

- [x] Dashboard UI (prototype)
- [ ] Next.js project scaffolding
- [ ] Auth system (JWT + bcrypt)
- [ ] User onboarding flow
- [ ] Discovery + swipe engine
- [ ] Match detection logic
- [ ] Socket.io real-time chat
- [ ] AWS S3 photo uploads
- [ ] Razorpay subscription integration
- [ ] Geolocation-based nearby users
- [ ] Image moderation pipeline
- [ ] Push notifications
- [ ] Admin moderation dashboard
- [ ] AI-based match suggestions
- [ ] Event-based pairing ("Find partner for tonight")
- [ ] Rating/review system

---

## Safety & Disclaimer

PairUp is built with user safety as a core principle:

- All profiles are moderated before going live
- Users can report or block anyone at any time
- Images are scanned via AWS Rekognition for explicit content
- No transactional or escort-related activity is permitted
- Violations result in immediate account suspension

> **This platform is for social companionship and shared activities only. Any use of the platform for illegal, explicit, or transactional purposes is strictly prohibited and will be reported to the appropriate authorities.**

---

## License

MIT License — see [LICENSE](./LICENSE) for details.

---

<p align="center">Built by Sparsh · PairUp is a portfolio project</p>
