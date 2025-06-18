# API Endpoint Reference

This document lists all main API endpoints, their purpose, supported HTTP methods, which user roles can access them, and the types of information they provide or accept.

---

## Authentication & User Management

- **POST /api/auth/register**
  - Register a new user (User)
  - Accepts user details, sends verification email

- **POST /api/auth/login**
  - Login for users, admins, and community admins
  - Returns JWT and user info

- **POST /api/auth/admin-register**
  - Register a new admin (Admin)
  - Accepts admin details, sends verification email

- **POST /api/auth/community-admin-register**
  - Register a new community admin (Community Admin)
  - Accepts community admin details, sends verification email

- **POST /api/auth/community-admin-login**
  - Login for community admins
  - Returns JWT and admin info

- **POST /api/auth/resend-verification**
  - Resend email verification link/code (All roles)
  - Accepts email, enforces cooldown

- **GET /api/auth/verify-email**
  - Verify user email by link (User)

- **GET /api/auth/verify-email-admin**
  - Verify admin email by link (Admin)

- **GET /api/auth/verify-email-community-admin**
  - Verify community admin email by link (Community Admin)

- **POST /api/auth/verify-email-code**
  - Verify user/admin email by code (User/Admin)
  - Accepts email and code

- **POST /api/auth/verify-email-code-community-admin**
  - Verify community admin email by code (Community Admin)
  - Accepts email and code

- **POST /api/user/change-password**
  - Change password (Authenticated User)

- **GET /api/user/profile**
  - Get user profile (Authenticated User)

---

## Events

- **GET /api/events**
  - List events (All roles)
  - Supports filters, pagination; returns event details
  - Response includes: `events`, `total`, `page`, `limit`, `totalPages`, and for each event: `participatedUserCount`, `ticketConfirmedUserCount`, `creatorName`

- **POST /api/events**
  - Create event (User, Admin, Community Admin)
  - Accepts event details (title, description, imageUrl, etc.)

- **PATCH /api/events**
  - Approve event (Admin, Community Admin)
  - Approves a pending event

- **POST /api/events/participate**
  - Participate in an event (User)
  - Mark interest/attendance

- **GET /api/events/participated**
  - List events a user has participated in (User)

- **GET /api/events/accepted-volunteers**
  - List accepted volunteers for an event (Admin, Community Admin)

- **GET /api/events/assigned-tasks**
  - List assigned tasks for an event (Admin, Community Admin)

- **GET /api/events/feed**
  - Get personalized event feed for the user (User)
  - Returns upcoming events from communities the user joined, categories the user likes, and creators the user has engaged with, excluding events already joined. Paginated and sorted by soonest event.

- **GET /api/events/purchased-tickets**
  - List purchased tickets for an event (User)

- **GET /api/events/volunteer-requests**
  - List volunteer requests for an event (Admin, Community Admin)

- **POST /api/events/ticket**
  - Book a ticket for an event (User)
  - Any authenticated user can book a ticket for a public event. Duplicate ticket booking for the same event by the same user is not allowed and will return an error.
  - All error handling and validation are enforced.

- **POST /api/events/approve-payment**
  - Approve manual payment for an event (Admin, Event Owner)

- **GET /api/events/details**
  - Get basic details for a single event by eventId. Returns: title, description, category, event time, ticket price, seat count, image, contact, location, community name, creator info (name, email, role), participatedUserCount, and ticketConfirmedUserCount.

---

## Communities

- **GET /api/communities**
  - List all communities (All roles)
  - Supports search, pagination

- **GET /api/communities/own**
  - List communities owned by the authenticated community admin

- **GET /api/communities/users**
  - List users in a community (Community Admin)

- **POST /api/communities**
  - Join a community (User)

---

## Notifications

- **GET /api/notifications**
  - List notifications for the authenticated user

- **POST /api/notifications/email**
  - Send notification email (Admin, Community Admin)

- **POST /api/notifications/push**
  - Register device for push notifications (All roles)

---

## Payments & Subscriptions

- **POST /api/payment/ipn**
  - Payment gateway IPN callback (System)

- **POST /api/stripe/webhook**
  - Stripe webhook endpoint (System)

- **GET /api/subscription**
  - Get subscription plans (All roles)

- **POST /api/subscription**
  - Subscribe to a plan (User)

---

## Admin

- **GET /api/admin/users**
  - List all users (Admin)

- **GET /api/admin/dashboard**
  - Get admin dashboard stats (Admin)

- **GET /api/admin/analytics**
  - Get analytics data (Admin)
  - Returns all of the following fields:
    - totalEvents
    - totalParticipants
    - totalVolunteers
    - activeEvents
    - completedEvents
    - totalUsers
    - totalCommunities
    - pendingCommunities
    - totalCommunityAdmins
    - pendingCommunityAdmins
    - totalTicketsSold
    - totalRevenue
    - activeUsers
    - eventsByCategory
    - volunteerRequestsPending
    - pendingEvents
    - totalTicketedEvents

- **GET /api/admin/report**
  - Get admin reports (Admin)

---

## Miscellaneous

- **GET /api/user/profile**
  - Get authenticated user's profile

- **PATCH /api/user/profile**
  - Update authenticated user's profile

---

## POST Request JSON Formats

### Register User
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "phone": "0123456789", // optional
  "dateOfBirth": "1995-01-01", // optional
  "occupation": "Engineer", // optional
  "country": "US", // optional
  "password": "yourpassword"
}
```

### Register Admin
```json
{
  "firstName": "Admin",
  "lastName": "User",
  "email": "admin@example.com",
  "phone": "0123456789", // optional
  "dateOfBirth": "1990-01-01", // optional
  "occupation": "Manager", // optional
  "country": "US", // optional
  "password": "yourpassword"
}
```

### Register Community Admin
```json
{
  "communityName": "My Community",
  "email": "admin@community.com",
  "phone": "0123456789", // optional
  "communityStartDate": "2024-01-01", // optional
  "communityType": "free", // or "non-profit", etc.
  "country": "US", // optional
  "password": "yourpassword"
}
```

### Login
```json
{
  "email": "user@example.com",
  "password": "yourpassword"
}
```

### Create Event
```json
{
  "title": "Event Title",
  "description": "Event description",
  "category": "Category",
  "eventTime": "2025-06-01T18:00:00.000Z",
  "isTicketed": true,
  "ticketPrice": 20, // required if isTicketed is true
  "seatCount": 100,
  "type": "public",
  "needVolunteer": true, // optional
  "imageUrl": "https://example.com/image.jpg", // optional
  "contact": "0123456789", // optional
  "location": "Dhaka, Bangladesh" // optional
}
```

### Book Ticket (Free or Paid)
// Free event:
```json
{
  "eventId": "<eventId>"
}
```
// Paid event (manual payment):
```json
{
  "eventId": "<eventId>",
  "payment_method": "bkash",
  "manual_tran_id": "TX123456",
  "phone": "017XXXXXXXX"
}
```

### Participate in Event
```json
{
  "eventId": "<eventId>",
  "status": "attending" // or "interested"
}
```

### Volunteer for Event
```json
{
  "eventId": "<eventId>"
}
```

### Add Comment
```json
{
  "eventId": "<eventId>",
  "text": "This event is awesome!"
}
```

### Send Chat Message
```json
{
  "eventId": "<eventId>",
  "message": "Hello!"
}
```

### Mark Notification as Read
```json
{
  "notificationId": "<notificationId>"
}
```

### Register Device for Push
```json
{
  "token": "<deviceToken>",
  "platform": "android" // or "ios"
}
```

### Change Password
```json
{
  "oldPassword": "oldpass",
  "newPassword": "newpass"
}
```

## User Object Example
```json
{
  "_id": "<user_id>",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "phone": "0123456789",
  "dateOfBirth": "1995-01-01T00:00:00.000Z",
  "occupation": "Engineer",
  "country": "US",
  "role": "user",
  "createdAt": "2025-06-17T12:00:00.000Z",
  "emailVerified": true
}
```

## Event Object Example
```json
{
  "_id": "<event_id>",
  "title": "Event Title",
  "description": "Event description",
  "category": "Category",
  "communityId": "<community_id>",
  "eventTime": "2025-07-01T10:00:00.000Z",
  "isTicketed": true,
  "ticketPrice": 100,
  "seatCount": 100,
  "createdBy": "<user_id>",
  "approvedBy": "<admin_id>",
  "type": "public",
  "createdAt": "2025-06-17T12:00:00.000Z",
  "createdByType": "user",
  "needVolunteer": true,
  "imageUrl": "https://example.com/image.jpg",
  "contact": "0123456789",
  "location": "Dhaka, Bangladesh"
}
```

## Ticket Object Example
```json
{
  "_id": "<ticket_id>",
  "eventId": "<event_id>",
  "userId": "<user_id>",
  "purchasedAt": "2025-06-17T12:00:00.000Z",
  "status": "confirmed"
}
```

## Payment Object Example
```json
{
  "_id": "<payment_id>",
  "userId": "<user_id>",
  "eventId": "<event_id>",
  "communityId": "<community_id>",
  "amount": 100,
  "currency": "BDT",
  "paymentMethod": "bkash",
  "manualTranId": "TX123456",
  "phone": "017XXXXXXXX",
  "status": "completed",
  "createdAt": "2025-06-17T12:00:00.000Z"
}
```

## Volunteer Request Object Example
```json
{
  "_id": "<volunteer_request_id>",
  "userId": "<user_id>",
  "eventId": "<event_id>",
  "status": "approved",
  "requestedAt": "2025-06-17T12:00:00.000Z",
  "reviewedBy": "<admin_id>",
  "reviewedAt": "2025-06-17T13:00:00.000Z"
}
```

## Community Object Example
```json
{
  "_id": "<community_id>",
  "communityName": "My Community",
  "email": "community@example.com",
  "phone": "0123456789",
  "communityStartDate": "2024-01-01T00:00:00.000Z",
  "communityType": "non-profit",
  "country": "US",
  "adminUser": "<community_admin_id>",
  "createdAt": "2025-06-17T12:00:00.000Z",
  "subscription": "<subscription_id>"
}
```

## Task Object Example
```json
{
  "_id": "<task_id>",
  "eventId": "<event_id>",
  "description": "Task description",
  "status": "pending",
  "dueDate": "2025-07-01T10:00:00.000Z",
  "volunteerId": "<user_id>",
  "createdAt": "2025-06-17T12:00:00.000Z"
}
```

---

> **Note:** All endpoints require JWT authentication unless otherwise specified. Role-based access is enforced for sensitive endpoints.
