# Airbnb Clone – Backend Feature Requirements Specification

## 1. User Authentication & Authorization

### Functional Requirements
- Support registration, login, logout, password reset, and profile management
- Issue secure JWT tokens with refresh mechanism
- Role-based access: Guest, Host (Property Owner), Admin
- Email verification on registration

### Non-Functional Requirements
- Passwords hashed with **bcrypt** or **Argon2**
- JWT tokens valid for 1 hour (access), 7 days (refresh)
- Rate limiting: max 10 login attempts per IP per 15 mins
- Secure password reset via time-limited token (expires in 15 mins)

### API Endpoints

| Method | Endpoint                        | Description                      | Auth Required |
|--------|----------------------------------|----------------------------------|---------------|
| POST   | `/api/v1/auth/register/`         | Register new user                | No            |
| POST   | `/api/v1/auth/login/`            | Login & get tokens               | No            |
| POST   | `/api/v1/auth/logout/`           | Invalidate refresh token         | Yes           |
| POST   | `/api/v1/auth/token/refresh/`    | Get new access token             | Yes (refresh) |
| POST   | `/api/v1/auth/password/reset/`   | Request password reset           | No            |
| POST   | `/api/v1/auth/password/reset/confirm/` | Confirm new password       | No (token)    |
| GET    | `/api/v1/auth/me/`               | Get current user profile         | Yes           |
| PATCH  | `/api/v1/auth/me/`               | Update profile (name, phone, etc)| Yes           |

### Request/Response Examples

**POST /api/v1/auth/register/**
```json
{
  "email": "john@example.com",
  "password": "SecurePass2025!",
  "password2": "SecurePass2025!",
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+2348012345678",
  "role": "guest"  // guest | host | admin
}
```
**Response (201 Created):**
```json
{
  "user": { "id": 42, "email": "john@example.com", "role": "guest" },
  "message": "Registration successful. Verification email sent."
}
```

**POST /api/v1/auth/login/**
```json
{
  "email": "john@example.com",
  "password": "SecurePass2025!"
}
```
**Response (200 OK):**
```json
{
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6...",
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6...",
  "user": { "id": 42, "email": "...", "role": "host" }
}
```

### Validation Rules
- Email: unique, valid format, max 254 chars
- Password: min 8 chars, 1 uppercase, 1 number, 1 special char
- Phone: E.164 format, validated with `phonenumbers` lib
- Role: choices = ['guest', 'host', 'admin']

### Performance Criteria
- Registration/login < 300ms (95th percentile)
- JWT verification < 10ms per request
- Support 10,000 concurrent authenticated users

---

## 2. Property Management (Host Features)

### Functional Requirements
- Hosts can create, update, delete, and publish/unpublish properties
- Upload up to 20 high-res photos per property
- Set pricing, availability calendar, house rules
- View booking history and revenue analytics

### API Endpoints

| Method | Endpoint                                | Description                        | Auth + Role       |
|--------|------------------------------------------|------------------------------------|-------------------|
| POST   | `/api/v1/properties/`                    | Create new property                | Host only         |
| GET    | `/api/v1/properties/`                    | List my properties                 | Host only         |
| GET    | `/api/v1/properties/{id}/`               | Retrieve property details          | Host or Public    |
| PATCH  | `/api/v1/properties/{id}/`               | Update property                    | Owner only        |
| DELETE | `/api/v1/properties/{id}/`               | Delete property (soft delete)      | Owner only        |
| POST   | `/api/v1/properties/{id}/photos/`       | Upload photos                      | Owner only        |
| POST   | `/api/v1/properties/{id}/availability/`  | Block/unblock dates                | Owner only        |

### Request Body (Create Property)

```json
{
  "title": "Cozy Studio in Lagos Island",
  "description": "Modern studio with ocean view...",
  "property_type": "apartment",
  "room_type": "entire_place",
  "guest_capacity": 2,
  "bedrooms": 1,
  "beds": 1,
  "bathrooms": 1,
  "amenities": ["wifi", "kitchen", "ac", "tv"],
  "price_per_night": 45000.00,
  "location": {
    "address": "123 Victoria Island",
    "city": "Lagos",
    "state": "Lagos",
    "country": "Nigeria",
    "lat": 6.4281,
    "lng": 3.4210
  },
  "house_rules": "No smoking, No parties"
}
```

### Validation Rules
- Price: > 5,000 NGN, < 50,000,000 NGN
- Guest capacity: 1–16
- Photos: max 20, each ≤ 10MB, JPEG/PNG only
- Geolocation: valid lat/lng within country bounds

### Performance & Scalability
- Property search with filters < 400ms (even at 100k listings)
- Image upload to Cloudinary/S3 with async processing (Celery)
- Availability calendar uses range indexing (PostgreSQL GIST)

---

## 3. Booking System

### Functional Requirements
- Guests can search available properties and book dates
- Instant booking or request-to-book (host approval)
- Automatic calendar blocking on successful booking
- Cancellation policies with refund rules
- Booking status: pending, confirmed, cancelled, completed

### API Endpoints

| Method | Endpoint                              | Description                          | Auth Required |
|--------|----------------------------------------|--------------------------------------|---------------|
| GET    | `/api/v1/bookings/availability/`       | Check date availability              | Yes           |
| POST   | `/api/v1/bookings/`                    | Create booking (instant or request)  | Yes           |
| GET    | `/api/v1/bookings/my-bookings/`        | List guest’s bookings                | Yes           |
| GET    | `/api/v1/bookings/host-bookings/`      | List host’s incoming bookings        | Host only     |
| PATCH  | `/api/v1/bookings/{id}/status/`        | Host approve/decline (request-to-book)| Host only     |
| POST   | `/api/v1/bookings/{id}/cancel/`        | Cancel booking                       | Yes           |

### Request Body (Create Booking)

```json
{
  "property": 123,
  "check_in": "2025-12-20",
  "check_out": "2025-12-27",
  "guests": 2,
  "total_price": 315000.00,
  "booking_type": "instant"  // or "request"
}
```

### Response (201 Created – Instant Booking)

```json
{
  "id": 889,
  "status": "confirmed",
  "guest": { "id": 42, "name": "Chidi Okoye" },
  "property": { "id": 123, "title": "Cozy Studio..." },
  "check_in": "2025-12-20",
  "check_out": "2025-12-27",
  "total_price": 315000.00,
  "created_at": "2025-11-25T10:30:00Z"
}
```

### Business Logic & Validation
- Prevent double-booking using DB-level locking (`SELECT FOR UPDATE`)
- Enforce minimum 1-night stay, maximum 30-night stay
- Auto-calculate total price = nights × price_per_night + cleaning fee + service fee
- Cancellation policy:
  - > 48h before check-in → 100% refund
  - < 48h → 50% refund
  - After check-in → no refund

### Performance Criteria
- 99.9% of availability checks return in < 200ms
- Booking creation < 500ms (including payment intent)
- Support 5,000 concurrent booking attempts during peak

### Security Measures
- Only property owner or admin can change booking status
- Guests can only cancel their own bookings
- All actions logged to audit trail (who, what, when)