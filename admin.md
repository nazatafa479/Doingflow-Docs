# Admin API Usage Guide

This document details all API endpoints and JSON data formats for admin users in the Event Management App backend. All endpoints require `Authorization: Bearer <token>` except registration/login.

## Authentication
- Register: `POST /api/auth/admin-register`
  - JSON:
    ```json
    {
      "firstName": "Admin",
      "lastName": "User",
      "email": "admin@example.com",
      "phone": "",
      "dateOfBirth": "1990-01-01",
      "occupation": "",
      "country": "US",
      "password": "yourpassword"
    }
    ```
  - **Email Verification:** After registration, an email is sent to the admin. The account must be verified via the link in the email before login is allowed or notifications are sent.
- Login: `POST /api/auth/login`
  - JSON:
    ```json
    {
      "email": "admin@example.com",
      "password": "yourpassword"
    }
    ```
  - Response: `{ token, role, userId, expiresAt }`

---

## Email Verification
- After registration, an email is sent to the admin with both a verification link and a 6-digit code.
- Verify email by:
  - Clicking the link: `GET /api/auth/verify-email-admin?token=<token>`
  - Or submitting the code: `POST /api/auth/verify-email-code` with `{ "email": "...", "code": "123456" }`
- **Resend Verification:**
  - If not verified, admins can request a new code/link via `POST /api/auth/resend-verification` with `{ "email": "..." }` (max once every 2 minutes).
- Admins cannot log in or receive notifications until their email is verified.

---

## User Management
- List users: `GET /api/admin/users?page=1&limit=10&role=user&search=John`
- Update user role: `PATCH /api/admin/users`
  - JSON:
    ```json
    { "userId": "<userId>", "role": "community_admin" }
    ```
- Delete user: `DELETE /api/admin/users`
  - JSON:
    ```json
    { "userId": "<userId>" }
    ```

---

## Event Management
- Approve event: `PATCH /api/events`
  - JSON:
    ```json
    { "eventId": "<eventId>" }
    ```
- List all events: `GET /api/events?page=1&limit=10&status=active`
  - Response includes: `events`, `total`, `page`, `limit`, `totalPages`
- Create event: `POST /api/events`
  - JSON:
    ```json
    {
      "title": "Event Title",
      "description": "Desc",
      "category": "Type",
      "communityId": "<communityId>",
      "eventTime": "2024-07-01T10:00:00Z",
      "isTicketed": false,
      "seatCount": 100,
      "type": "public"
    }
    ```
  - You can include an `imageUrl` field (string, optional) when creating an event to specify an image for the event.
- Event participation report: `GET /api/admin/report?type=event&eventId=<eventId>&format=json|csv`
- Book free/paid ticket: `POST /api/events/ticket`
  - JSON:
    ```json
    { "eventId": "<eventId>" }
    ```

---

## Community Management
- List communities: `GET /api/communities?page=1&limit=10&search=MyCommunity`
- View community details: `GET /api/communities?search=MyCommunity`

---

## View Own Community (Community Admin)
- **URL:** `/api/communities/own`
- **Method:** `GET`
- **Auth:** Required (Community Admin)
- **Response:**
  - `200 OK`
  ```json
  {
    "community": { ...communityObject }
  }
  ```
- Returns the community associated with the currently authenticated community admin.

---

## View Community Users (Community Admin)
- **URL:** `/api/communities/users`
- **Method:** `GET`
- **Auth:** Required (Community Admin)
- **Response:**
  - `200 OK`
  ```json
  {
    "users": [ ...userObjects ]
  }
  ```
- Returns only the users who have joined the currently authenticated community admin's community.

---

## Analytics & Dashboard
- Analytics: `GET /api/admin/analytics` (shows only active events, i.e., events starting today or in the future)
- Approval dashboard: `GET /api/admin/dashboard`

---

## Reporting
- Event participation: `GET /api/admin/report?type=event&eventId=<eventId>&format=json|csv`
- Financial: `GET /api/admin/report?type=financial&startDate=2024-01-01&endDate=2024-12-31&format=json|csv`
- Volunteer activity: `GET /api/admin/report?type=volunteer&eventId=<eventId>&format=json|csv`

---

## Notifications
- Send in-app: `POST /api/notifications`
  - JSON:
    ```json
    { "userId": "<userId>", "type": "event_update", "message": "Event updated!" }
    ```
- Send email: `POST /api/notifications/email`
  - JSON:
    ```json
    { "userId": "<userId>", "subject": "Subject", "message": "Body text" }
    ```
- Send push: `POST /api/notifications/push/send`
  - JSON:
    ```json
    { "userId": "<userId>", "title": "Title", "body": "Message body", "data": { "key": "value" } }
    ```
- Register device for push: `POST /api/notifications/push/register`
  - JSON:
    ```json
    { "token": "<deviceToken>", "platform": "android" }
    ```
- Get notifications: `GET /api/notifications`
- Mark notification as read: `PATCH /api/notifications`
  - JSON:
    ```json
    { "notificationId": "<notificationId>" }
    ```

---

## Scheduled Reports
- Daily financial report is sent to the admin email (set `ADMIN_EMAIL` in `.env.local`).

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

## Event Participation & Volunteer Logic
- If `isTicketed: true`, only ticket booking is allowed. Participation and volunteer requests are not accepted.
- If `isTicketed: false` and `needVolunteer: true`, both participation and volunteer requests are accepted.
- If `isTicketed: false` and `needVolunteer: false`, only participation requests are accepted. Volunteer requests are not accepted.

## User Participated Events
- **URL:** `/api/events/participated`
- **Method:** `GET`
- **Auth:** Required (User)
- **Response:**
  - `200 OK`
  ```json
  {
    "events": [ ... ]
  }
  ```

---

## View Volunteer Requests (Event Based)
- **URL:** `/api/events/volunteer-requests?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required
- **Response:**
  - `200 OK`
  ```json
  {
    "requests": [ ...volunteerRequestObjects ]
  }
  ```

## View Accepted Volunteers (Event Based)
- **URL:** `/api/events/accepted-volunteers?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required
- **Response:**
  - `200 OK`
  ```json
  {
    "volunteers": [ ...volunteerRequestObjects ]
  }
  ```

---

## View Purchased Tickets (Event Based)
- **URL:** `/api/events/purchased-tickets?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required
- **Response:**
  - `200 OK`
  ```json
  {
    "tickets": [ ...ticketObjects ]
  }
  ```
- Returns all purchased tickets for the specified event, including user info for each ticket.

---

## View Participated Users (Event Based)
- **URL:** `/api/events/participated-users?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required
- **Response:**
  - `200 OK`
  ```json
  {
    "participants": [ ...participantObjects ]
  }
  ```
- Returns all users who have participated in the specified event, including user info for each participant.

---

## View Assigned Task Users (Event Based)
- **URL:** `/api/events/assigned-tasks?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required
- **Response:**
  - `200 OK`
  ```json
  {
    "tasks": [ ...taskObjectsWithVolunteerUser ]
  }
  ```
- Returns all assigned tasks for the specified event, including user info for each assigned volunteer.

---

# Admin Guide: Manual Payment Approval

# Event-Based Payment Handling System

## Overview
- Payment handling is event-based. The admin, community admin, or user who creates a specific event is responsible for handling the payment approval for that event.
- Only the event creator (admin, community admin, or user) or an admin can approve manual payments for their event.
- This ensures that payment approval and ticket issuance are managed by the event owner or platform admin, maintaining proper authorization and accountability.

## Manual Payment Workflow (Event-Based)
- When a user books a ticket for a paid event, they submit manual payment details (phone, payment method, transaction ID).
- The payment is saved with status `pending_approval`.
- The event creator (admin, community admin, or user) or an admin can approve the payment using the `/api/events/approve-payment` endpoint.
- On approval, the payment status is set to `completed` and the ticket is issued if seats are available.

## Approving Payments
- Use the `/api/events/approve-payment` endpoint (POST) with `paymentId` and `eventId` to approve a payment.
- Only admins or the event owner can approve payments.
- On approval, the payment status is set to `completed` and the ticket is issued if seats are available.

## Example Approval Request
```json
POST /api/events/approve-payment
{
  "paymentId": "<payment_id>",
  "eventId": "<event_id>"
}
```

---

# Notes
- If payment is mentioned, only manual payment is supported. No payment gateway integration.
- Payment approval is always handled by the event creator (admin, community admin, or user) or an admin, depending on who created the event.

---

## View All Payments for an Event
- **URL:** `/api/events/payments?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required (admin or event creator)
- **Response:**
  - `200 OK`
  ```json
  {
    "payments": [ ...paymentObjectsWithUserInfo ]
  }
  ```
- Returns all payments for the specified event, including user info for each payment. Only the event creator or an admin can access this data.

---

## POST Request JSON Formats

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

### Login
```json
{
  "email": "admin@example.com",
  "password": "yourpassword"
}
```

### Approve Event
```json
{
  "eventId": "<eventId>"
}
```

### Update User Role
```json
{
  "userId": "<userId>",
  "role": "community_admin" // or "user", "admin"
}
```

### Delete User
```json
{
  "userId": "<userId>"
}
```

### Create Event
```json
{
  "title": "Event Title",
  "description": "Desc",
  "category": "Type",
  "communityId": "<communityId>",
  "eventTime": "2024-07-01T10:00:00Z",
  "isTicketed": false,
  "seatCount": 100,
  "type": "public",
  "imageUrl": "https://example.com/image.jpg" // optional
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

### Send Notification
```json
{
  "userId": "<userId>",
  "type": "event_update",
  "message": "Event updated!"
}
```

### Send Email Notification
```json
{
  "userId": "<userId>",
  "subject": "Subject",
  "message": "Body text"
}
```

### Send Push Notification
```json
{
  "userId": "<userId>",
  "title": "Title",
  "body": "Message body",
  "data": { "key": "value" }
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

---

## User Object Example
```json
{
  "_id": "<user_id>",
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "phone": "0123456789",
  "dateOfBirth": "1990-01-01T00:00:00.000Z",
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
  "createdByType": "admin",
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

## View Pending Events (Paginated)
- **URL:** `/api/admin/pending-events`
- **Method:** `GET`
- **Query Params:**
  - `page` (default: 1)
  - `limit` (default: 10)
- **Auth:** Required (Admin)
- **Response:**
  - `200 OK`
    ```json
    {
      "pendingEvents": [
        {
          "_id": "<event_id>",
          "title": "Event Title",
          "eventTime": "2025-07-01T10:00:00.000Z",
          "communityId": {
            "_id": "<community_id>",
            "communityName": "Community Name"
          },
          "createdBy": {
            "_id": "<user_id>",
            "firstName": "John",
            "lastName": "Doe",
            "email": "john@example.com",
            "role": "user"
          }
        }
        // ...up to 10 per page
      ],
      "total": 42,
      "page": 1,
      "limit": 10,
      "totalPages": 5
    }
    ```

## View Pending Communities (Paginated)
- **URL:** `/api/admin/pending-communities`
- **Method:** `GET`
- **Query Params:**
  - `page` (default: 1)
  - `limit` (default: 10)
- **Auth:** Required (Admin)
- **Response:**
  - `200 OK`
    ```json
    {
      "pendingCommunities": [
        {
          "_id": "<community_id>",
          "communityName": "My Community",
          "email": "community@example.com",
          "country": "US",
          "communityType": "non-profit"
        }
        // ...up to 10 per page
      ],
      "total": 12,
      "page": 1,
      "limit": 10,
      "totalPages": 2
    }
    ```
