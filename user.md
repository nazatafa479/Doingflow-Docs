# User API Usage Guide

This document details all API endpoints and JSON data formats for regular users in the Event Management App backend. All endpoints require `Authorization: Bearer <token>` except registration/login.

## Authentication
- Register: `POST /api/auth/register`
  - JSON:
    ```json
    {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "phone": "",
      "dateOfBirth": "1995-01-01",
      "occupation": "Engineer",
      "country": "US",
      "password": "yourpassword"
    }
    ```
  - **Email Verification:** After registration, an email is sent to the user. The account must be verified via the link in the email before login is allowed or notifications are sent.
- Login: `POST /api/auth/login`
  - JSON:
    ```json
    {
      "email": "john@example.com",
      "password": "yourpassword"
    }
    ```
  - Response: `{ token, role, userId, expiresAt }`

---

## Email Verification
- After registration, an email is sent to the user with both a verification link and a 6-digit code.
- Verify email by:
  - Clicking the link: `GET /api/auth/verify-email?token=<token>`
  - Or submitting the code: `POST /api/auth/verify-email-code` with `{ "email": "...", "code": "123456" }`
- **Resend Verification:**
  - If not verified, users can request a new code/link via `POST /api/auth/resend-verification` with `{ "email": "..." }` (max once every 2 minutes).
- Users cannot log in or receive notifications until their email is verified.

---

## Community
- List communities: `GET /api/communities?page=1&limit=10&search=MyCommunity`
- Join community: `POST /api/communities`
  - JSON:
    ```json
    { "communityId": "<communityId>" }
    ```

---

## Events
- List events: `GET /api/events?page=1&limit=10&status=active`
  - Returns paginated events with fields: `events`, `total`, `page`, `limit`, `totalPages`, and for each event: `participatedUserCount`, `ticketConfirmedUserCount`, `creatorName`.
- Personalized feed: `GET /api/events/feed`
  - Returns a personalized feed of upcoming events for you, including:
    - Events from communities you have joined
    - Events in categories you have previously participated in
    - New events from users who created events you joined in the past
  - Excludes events you already joined
  - Results are deduplicated, sorted by soonest event, and paginated
- Participate in event: `POST /api/events/participate`
  - JSON:
    ```json
    { "eventId": "<eventId>", "status": "attending" }
    ```
- Volunteer for event: `POST /api/events/volunteer`
  - JSON:
    ```json
    { "eventId": "<eventId>" }
    ```
- Book free/paid ticket: `POST /api/events/ticket`
  - JSON:
    ```json
    { "eventId": "<eventId>" }
    ```
  - Any authenticated user can book a ticket for a public event. Duplicate ticket booking for the same event by the same user is not allowed and will return an error.
  - All error handling and validation are enforced.
- Get event details: `GET /api/events/details?eventId=<eventId>`
  - Returns basic event info: title, description, category, event time, ticket price, seat count, image, contact, location, community name, creator info (name, email, role), participatedUserCount, and ticketConfirmedUserCount.

### Event Participation & Volunteer Logic
- If `isTicketed: true`, only ticket booking is allowed. Participation and volunteer requests are not accepted.
- If `isTicketed: false` and `needVolunteer: true`, both participation and volunteer requests are accepted.
- If `isTicketed: false` and `needVolunteer: false`, only participation requests are accepted. Volunteer requests are not accepted.

---

## Create Event
- You can include an `imageUrl` field (string, optional) when creating an event to specify an image for the event. This should be a publicly accessible URL to an image file. The image will be displayed on the event page and in listings.

---

## Notifications
- Get notifications: `GET /api/notifications`
- Mark as read: `PATCH /api/notifications`
  - JSON:
    ```json
    { "notificationId": "<notificationId>" }
    ```
- Register device for push: `POST /api/notifications/push/register`
  - JSON:
    ```json
    { "token": "<deviceToken>", "platform": "android" }
    ```
- All in-app notifications for users are also sent to the user's verified email address.

---

## Profile
- Get profile: `GET /api/user/profile`
- Update profile: `PATCH /api/user/profile`
- Change password: `PATCH /api/user/change-password`
  - JSON:
    ```json
    { "oldPassword": "oldpass", "newPassword": "newpass" }
    ```

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

### Login
```json
{
  "email": "john@example.com",
  "password": "yourpassword"
}
```

### Join Community
```json
{
  "communityId": "<communityId>"
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
  "payment_method": "bkash", // or "nagad", "bank", etc.
  "manual_tran_id": "TX123456",
  "phone": "017XXXXXXXX"
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

# Event-Based Payment Handling System

## Overview
- Payment handling is event-based. The user, community admin, or admin who creates a specific event is responsible for handling the payment approval for that event.
- Only the event creator (user, community admin, or admin) or an admin can approve manual payments for their event.
- This ensures that payment approval and ticket issuance are managed by the event owner or platform admin, maintaining proper authorization and accountability.

## Manual Payment Workflow (Event-Based)
- When a user books a ticket for a paid event, they submit manual payment details (phone, payment method, transaction ID).
- The payment is saved with status `pending_approval`.
- The event creator (user, community admin, or admin) or an admin can approve the payment using the `/api/events/approve-payment` endpoint.
- On approval, the payment status is set to `completed` and the ticket is issued if seats are available.

## Example Approval Request
```json
POST /api/events/approve-payment
{
  "paymentId": "<payment_id>",
  "eventId": "<event_id>"
}
```

## View All Payments for an Event
- **URL:** `/api/events/payments?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required (event creator or admin)
- **Response:**
  - `200 OK`
  ```json
  {
    "payments": [ ...paymentObjectsWithUserInfo ]
  }
  ```
- Returns all payments for the specified event, including user info for each payment. Only the event creator or an admin can access this data.

---

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

## Notes
- All endpoints require `Authorization: Bearer <token>` except registration/login.
- Use pagination and filters as needed.
- For full JSON response examples, see the API responses or test with Postman/cURL.
- If payment is mentioned, only manual payment is supported. No payment gateway integration.
- Payment approval is always handled by the event creator (user, community admin, or admin), depending on who created the event.
