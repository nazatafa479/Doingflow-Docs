# Community Admin API Usage Guide

This document details all API endpoints and JSON data formats for community admins in the Event Management App backend. All endpoints require `Authorization: Bearer <token>` except registration/login.

## Authentication
- Register: `POST /api/auth/community-admin-register`
  - JSON:
    ```json
    {
      "communityName": "My Community",
      "email": "admin@community.com",
      "phone": "",
      "communityStartDate": "2024-01-01",
      "communityType": "free",
      "country": "US",
      "password": "yourpassword"
    }
    ```
  - **Email Verification:** After registration, an email is sent to the community admin with both a verification link and a 6-digit code. The account must be verified via the link or code in the email before login is allowed or notifications are sent.
- Login: `POST /api/auth/login`
  - JSON:
    ```json
    {
      "email": "admin@community.com",
      "password": "yourpassword"
    }
    ```
  - Response: `{ token, role, userId, expiresAt }`

---

## Email Verification
- After registration, an email is sent to the community admin with both a verification link and a 6-digit code.
- Verify email by:
  - Clicking the link: `GET /api/auth/verify-email-community-admin?token=<token>`
  - Or submitting the code:
    - `POST /api/auth/verify-email-code-community-admin` with `{ "email": "...", "code": "123456" }`
    - (Legacy: `POST /api/auth/verify-email-code` also works if your client is not role-specific)
- **Resend Verification:**
  - If not verified, community admins can request a new code/link via `POST /api/auth/resend-verification` with `{ "email": "..." }` (max once every 2 minutes).
- Community admins cannot log in or receive notifications until their email is verified.
- After successful verification, the verification code and token are removed from the database.

---

## Community Management
- View own community: `GET /api/communities?search=My%20Community`
- List communities: `GET /api/communities?page=1&limit=10&search=MyCommunity`
- Join community: `POST /api/communities`
  - JSON:
    ```json
    { "communityId": "<communityId>" }
    ```
- Manage volunteers: `PATCH /api/events/volunteer`
  - JSON:
    ```json
    { "requestId": "<volunteerRequestId>", "status": "approved" }
    ```
- Assign tasks: `POST /api/events/task`
  - JSON:
    ```json
    { "eventId": "<eventId>", "description": "Task details", "dueDate": "2024-06-01", "volunteerId": "<userId>" }
    ```

---

## Event Management
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
- List events: `GET /api/events?communityId=<communityId>`
- Approve volunteers: `PATCH /api/events/volunteer`
  - JSON:
    ```json
    { "requestId": "<volunteerRequestId>", "status": "approved" }
    ```
- Assign tasks: `POST /api/events/task`
  - JSON:
    ```json
    { "eventId": "<eventId>", "description": "Task details", "dueDate": "2024-06-01", "volunteerId": "<userId>" }
    ```
- Event participation report: `GET /api/admin/report?type=event&eventId=<eventId>&format=json|csv`
- Book free/paid ticket: `POST /api/events/ticket`
  - JSON:
    ```json
    { "eventId": "<eventId>" }
    ```

---

## Ticketing
- Book free ticket: `POST /api/events/ticket`
  - JSON:
    ```json
    { "eventId": "<eventId>" }
    ```
- Book paid ticket: `POST /api/events/ticket` (returns Stripe Checkout URL)

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

## View Own Community
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

## View Community Users
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

# Event-Based Payment Handling System

## Overview
- Payment handling is event-based. The community admin who creates a specific event is responsible for handling the payment approval for that event.
- Only the event creator (community admin or user) or an admin can approve manual payments for their event.
- This ensures that payment approval and ticket issuance are managed by the event owner or platform admin, maintaining proper authorization and accountability.

## Manual Payment Workflow (Event-Based)
- When a user books a ticket for a paid event, they submit manual payment details (phone, payment method, transaction ID).
- The payment is saved with status `pending_approval`.
- The event creator (community admin or user) or an admin can approve the payment using the `/api/events/approve-payment` endpoint.
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

## View All Payments for an Event
- **URL:** `/api/events/payments?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required (community admin or event creator)
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
  "email": "admin@community.com",
  "password": "yourpassword"
}
```

### Join Community
```json
{
  "communityId": "<communityId>"
}
```

### Approve Volunteer
```json
{
  "requestId": "<volunteerRequestId>",
  "status": "approved" // or "rejected"
}
```

### Assign Task
```json
{
  "eventId": "<eventId>",
  "description": "Task details",
  "dueDate": "2024-06-01", // optional
  "volunteerId": "<userId>"
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
  "createdByType": "community_admin",
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

# Notes
- All endpoints require `Authorization: Bearer <token>` except registration/login.
- Use pagination and filters as needed.
- For full JSON response examples, see the API responses or test with Postman/cURL.
- If payment is mentioned, only manual payment is supported. No payment gateway integration.
- Payment approval is always handled by the event creator (community admin or user) or an admin, depending on who created the event.
