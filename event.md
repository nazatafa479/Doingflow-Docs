# Event API Documentation

This document describes the RESTful API endpoints for event management in the Event Management App backend. All endpoints require JWT authentication unless otherwise specified. Use the `Authorization: Bearer <token>` header for all requests.

---

## Email Verification (All Roles)
- After registration, all users (user, admin, community admin) receive both a verification link and a 6-digit code by email.
- Verify email by:
  - Clicking the link for your role:
    - User: `GET /api/auth/verify-email?token=<token>`
    - Admin: `GET /api/auth/verify-email-admin?token=<token>`
    - Community Admin: `GET /api/auth/verify-email-community-admin?token=<token>`
  - Or submitting the code:
    - User/Admin: `POST /api/auth/verify-email-code` with `{ "email": "...", "code": "123456" }`
    - Community Admin: `POST /api/auth/verify-email-code-community-admin` with `{ "email": "...", "code": "123456" }`
- **Resend Verification:**
  - If not verified, request a new code/link via `POST /api/auth/resend-verification` with `{ "email": "..." }` (max once every 2 minutes).
- Login and notifications are blocked until email is verified.
- After successful verification, the verification code and token are removed from the database.

---

## Notifications
- All in-app notifications are also sent to the user's/admin's/community admin's verified email address.

---

## Endpoints

### 1. Create Event
- **URL:** `/api/events`
- **Method:** `POST`
- **Auth:** Required (Admin, Community Admin, or User)
- **Request Body:**
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
  - **Note:**
    - You can include `imageUrl`, `contact`, and `location` (all optional) to specify image, contact info, and location for the event. If omitted, those fields will be empty by default.
    - If the creator is an **admin** or **community admin**, the event will be associated with their community (`communityId` is set automatically from authentication).
    - If the creator is a **user**, the event will be associated with their user ID (`userId` is set automatically from authentication). You do not need to send `communityId` or `userId` in the request body.
    - The creator's identity and type (`createdBy`, `createdByType`) are always set from authentication and not from the request body.
- **Response:**
  - `201 Created`
  ```json
  {
    "message": "Event created",
    "event": { ...eventObject }
  }
  ```

---

### 2. List Events
- **URL:** `/api/events`
- **Method:** `GET`
- **Auth:** Not required (public feed)
- **Query Params:**
  - `page` (default: 1)
  - `limit` (default: 10)
  - `category`, `type`, `status` (`active`, `upcoming`, `completed`), `communityId`, `search`
- **Response:**
  - `200 OK`
  ```json
  {
    "events": [
      {
        "_id": "...",
        "title": "...",
        "description": "...",
        // ...other fields...
        "participatedUserCount": 12,
        "ticketConfirmedUserCount": 5,
        "creatorName": "Creator Name"
      }
    ],
    "total": 42,
    "page": 1,
    "limit": 10,
    "totalPages": 5
  }
  ```
- Each event object now includes:
  - `participatedUserCount`: Number of users who participated in the event
  - `ticketConfirmedUserCount`: Number of users with confirmed tickets (if `isTicketed` is true)

---

### 3. Approve Event
- **URL:** `/api/events`
- **Method:** `PATCH`
- **Auth:** Required (Admin or Community Admin)
- **Request Body:**
  ```json
  {
    "eventId": "<event_id>"
  }
  ```
- **Response:**
  - `200 OK`
  ```json
  {
    "message": "Event approved",
    "event": { ...eventObject }
  }
  ```

---

### 4. Participate in Event
- **URL:** `/api/events/participate`
- **Method:** `POST`
- **Auth:** Required (User)
- **Request Body:**
  ```json
  {
    "eventId": "<event_id>",
    "status": "interested" // or "attending"
  }
  ```
- **Response:**
  - `201 Created` or `200 OK`
  ```json
  {
    "message": "Participation recorded",
    "participant": { ...participantObject }
  }
  ```

---

### 5. Book Ticket
- **URL:** `/api/events/ticket`
- **Method:** `POST`
- **Auth:** Required (User)
- **Request Body:**
  ```json
  {
    "eventId": "<event_id>"
  }
  ```
- **Response:**
  - For free events:
    ```json
    {
      "message": "Free ticket booked",
      "event": { ...eventObject }
    }
    ```
  - For paid events:
    ```json
    {
      "message": "Manual payment submitted. Awaiting admin approval.",
      "paymentId": "...",
      "paymentStatus": "pending_approval",
      "manualPaymentResult": { ... }
    }
    ```
- **Access:** Any authenticated user can book a ticket for a public event. Duplicate ticket booking for the same event by the same user is not allowed and will return an error.
- **Error Handling:**
  - If a user tries to book more than one ticket for the same event, a `400 Bad Request` error is returned.
  - All other errors (e.g., no seats available, missing payment info) are handled with appropriate status codes and messages.

---

### 6. Get User Event Feed
- **URL:** `/api/events/feed`
- **Method:** `GET`
- **Auth:** Required (User)
- **Query Params:**
  - `page` (default: 1)
  - `limit` (default: 10)
- **Response:**
  - `200 OK`
  ```json
  {
    "events": [ ... ],
    "total": 42,
    "page": 1,
    "limit": 10,
    "totalPages": 5
  }
  ```
- **How it works:**
  - Returns a personalized feed of upcoming events for the user, including:
    - Events from communities the user has joined
    - Events in categories the user has previously participated in
    - New events from users who created events the user joined in the past
  - Excludes events the user already joined
  - Results are deduplicated, sorted by soonest event, and paginated

---

### Event Participation & Volunteer Logic
- If `isTicketed: true`, only ticket booking is allowed. Participation and volunteer requests are not accepted.
- If `isTicketed: false` and `needVolunteer: true`, both participation and volunteer requests are accepted.
- If `isTicketed: false` and `needVolunteer: false`, only participation requests are accepted. Volunteer requests are not accepted.

### 7. Get User Participated Events
- **URL:** `/api/events/participated`
- **Method:** `GET`
- **Auth:** Required (User)
- **Response:**
  - `200 OK`
  ```json
  {
    "events": [
      {
        "_id": "...",
        "title": "...",
        // ...other event fields...
        "participated": true, // true if user participated or sent a volunteer request
        "volunteerRequested": true, // true if user sent a volunteer request for this event
        "volunteer": true // true if user's volunteer request was approved for this event
      }
    ]
  }
  ```
- **Logic:**
  - If a user sends a volunteer request for an event, they are automatically considered as having participated in that event (`participated: true`).
  - If their volunteer request is approved, `volunteer: true`.
  - If they only participated (not volunteered), `volunteerRequested: false` and `volunteer: false`.
  - Participation and volunteering are strictly separated: a participation request is not a volunteer request, and vice versa.

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
- Returns all volunteer requests for the specified event.

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
- Returns all accepted (approved) volunteers for the specified event.

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

### Analytics
- **URL:** `/api/admin/analytics`
- **Method:** `GET`
- **Auth:** Required (Admin)
- **Response:**
  - `200 OK`
  ```json
  {
    "totalEvents": 10,
    "totalParticipants": 100,
    "totalVolunteers": 20,
    "activeEvents": 5,
    "completedEvents": 5,
    "totalUsers": 50,
    "totalCommunities": 8,
    "pendingCommunities": 2,
    "totalCommunityAdmins": 6,
    "pendingCommunityAdmins": 1,
    "totalTicketsSold": 120,
    "totalRevenue": 5000,
    "activeUsers": 30,
    "eventsByCategory": { "Music": 4, "Tech": 3 },
    "volunteerRequestsPending": 7,
    "pendingEvents": 3,
    "totalTicketedEvents": 4
  }
  ```
- **Field Descriptions:**
  - See `admin.md` for full field breakdown.
- **Note:** Only events with `eventTime` today or in the future are counted as `activeEvents`. Past events are counted as `completedEvents`.

---

### X. Get Event Details
- **URL:** `/api/events/details?eventId=<eventId>`
- **Method:** `GET`
- **Auth:** Required
- **Query Params:**
  - `eventId` (required): The ID of the event to fetch
- **Response:**
  - `200 OK`
  ```json
  {
    "event": {
      "_id": "...",
      "title": "Event Title",
      "description": "Event description",
      "category": "Category",
      "eventTime": "2025-06-01T18:00:00.000Z",
      "isTicketed": true,
      "ticketPrice": 20,
      "seatCount": 100,
      "imageUrl": "https://example.com/image.jpg",
      "contact": "0123456789",
      "location": "Dhaka, Bangladesh",
      "community": "Community Name",
      "createdBy": {
        "_id": "...",
        "name": "Creator Name",
        "email": "creator@example.com",
        "role": "user"
      },
      "createdAt": "2025-05-01T12:00:00.000Z",
      "participatedUserCount": 12,
      "ticketConfirmedUserCount": 5
    }
  }
  ```
- Returns only the basic event details: title, description, category, event time, ticket price, seat count, image, community name, and creator info (name, email, role).

---

### X. Event Comments
- **URL:** `/api/events/comments`
- **Methods:**
  - `GET`: Get all comments for a specific event (paginated, most recent first)
    - **Query Params:**
      - `eventId` (required): The ID of the event
      - `page` (default: 1): Page number
      - `limit` (default: 10): Comments per page (max 10 recommended)
    - **Response:**
      - `200 OK`
      ```json
      {
        "comments": [
          {
            "_id": "<comment_id>",
            "eventId": "<event_id>",
            "userId": "<user_id>",
            "comment": "Comment text",
            "createdAt": "2025-06-16T12:00:00.000Z",
            "user": {
              "firstName": "John",
              "lastName": "Doe",
              "email": "john@example.com"
            }
          }
        ],
        "total": 42,
        "page": 1,
        "limit": 10,
        "totalPages": 5
      }
      ```
  - `POST`: Add a comment to a specific event
    - **Body:**
      ```json
      { "eventId": "...", "text": "Comment text" }
      ```
    - **Response:**
      - `201 Created`
      ```json
      { "message": "Comment added", "comment": {
        "_id": "<comment_id>",
        "eventId": "<event_id>",
        "userId": "<user_id>",
        "comment": "Comment text",
        "createdAt": "2025-06-16T12:00:00.000Z",
        "user": {
          "firstName": "John",
          "lastName": "Doe",
          "email": "john@example.com"
        }
      }}
      ```
- **Access:**
  - Any authenticated user can add or view comments for any event. There is no participation or volunteer restriction.
- **Storage Model:**
  - Each comment is a separate document in the `Comment` collection, with an `eventId` field.
  - This allows unlimited scaling, better sharding, and avoids MongoDB document size limits.
  - Each comment includes sender info, content, and timestamp. Sender info is populated in the API response.
- **How to use:**
  - **Add a comment:**
    - `POST /api/events/comments`
    - Body:
      ```json
      {
        "eventId": "<event_id>",
        "text": "This event is awesome!"
      }
      ```
    - Auth required: Any authenticated user.
  - **Fetch comments:**
    - `GET /api/events/comments?eventId=<event_id>&page=1&limit=10`
    - Auth required: Any authenticated user.
    - Returns paginated, most recent comments first, with sender info populated.
- **Middleware:**
  - All comment endpoints are protected by authentication (JWT).
  - There are no restrictions based on participation or volunteering.
- **Notes:**
  - Comments are always event-specific. Comments for one event are never shown on another event. Most recent comments are shown first. Supports pagination.
  - The storage model is optimized for unlimited scalability: each comment is a separate document with an eventId field.

---

### X. View Purchased Ticket (User)
- **URL:** `/api/events/ticket-details`
- **Method:** `GET`
- **Query Params:**
  - `eventId` (required): The ID of the event
- **Access:**
  - Only the authenticated user can view their own ticket for the specified event.
- **Response:**
  - `200 OK`
    ```json
    {
      "ticket": {
        "_id": "<ticket_id>",
        "eventId": "<event_id>",
        "userId": "<user_id>",
        "purchasedAt": "2025-06-17T12:00:00.000Z",
        "status": "confirmed"
      },
      "event": {
        "_id": "<event_id>",
        "title": "Event Title",
        "description": "..."
        // ...other event fields
      },
      "payment": {
        "_id": "<payment_id>",
        "userId": "<user_id>",
        "eventId": "<event_id>",
        "amount": 100,
        "currency": "BDT",
        "paymentMethod": "bkash",
        "manualTranId": "TX123456",
        "phone": "017XXXXXXXX",
        "status": "completed",
        "createdAt": "2025-06-17T12:00:00.000Z"
      }
    }
    ```
  - `404 Not Found` if the user has not purchased a ticket for the event.
- **Notes:**
  - This endpoint is for users to view their own ticket details for a specific event.
  - Returns both the ticket, event, and payment details for convenience.

---

## POST Request JSON Formats

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

# ...existing code...

---

# Event Ticketing (Manual Payment)

## Booking a Ticket
- For paid events, users must submit:
  - Phone number
  - Payment method (bkash, nagad, bank, etc.)
  - Transaction/reference ID
- The payment is saved as `pending_approval`.

## Approval
- Admin or event owner must approve the payment using the `/api/events/approve-payment` endpoint.
- On approval, the ticket is issued if seats are available.

---

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

---

# Notes
- If payment is mentioned, only manual payment is supported. No payment gateway integration.
- Payment approval is always handled by the event creator (user, community admin, or admin), depending on who created the event.

---

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

### 8. Event-Based Group Chat (Participated Users Only)
- **Endpoint:** `/api/events/chat`
- **Methods:**
  - `GET`: Get all chat messages for a specific event (paginated, most recent first)
    - **Query Params:**
      - `eventId` (required): The ID of the event
      - `page` (default: 1): Page number
      - `limit` (default: 20): Messages per page (max 20 recommended)
    - **Response:**
      - `200 OK`
      ```json
      {
        "messages": [
          {
            "_id": "<message_id>",
            "eventId": "<event_id>",
            "userId": "<user_id>",
            "message": "Hello!",
            "createdAt": "2025-06-17T12:00:00.000Z",
            "user": {
              "firstName": "John",
              "lastName": "Doe",
              "email": "john@example.com"
            }
          }
        ],
        "total": 42,
        "page": 1,
        "limit": 20,
        "totalPages": 3
      }
      ```
  - `POST`: Send a chat message to a specific event
    - **Body:**
      ```json
      { "eventId": "...", "message": "Hello!" }
      ```
    - **Response:**
      - `201 Created`
      ```json
      { "message": "Message sent", "chat": {
        "_id": "<message_id>",
        "eventId": "<event_id>",
        "userId": "<user_id>",
        "message": "Hello!",
        "createdAt": "2025-06-17T12:00:00.000Z",
        "user": {
          "firstName": "John",
          "lastName": "Doe",
          "email": "john@example.com"
        }
      }}
      ```
- **Access:**
  - Only authenticated event participants can send or view chat messages for an event.
- **Storage Model:**
  - Each chat message is a separate document in the `Chat` collection, with an `eventId` field.
  - This allows unlimited scaling, better sharding, and avoids MongoDB document size limits.
  - Each message includes sender info, content, and timestamp. Sender info is populated in the API response.
- **How to use:**
  - **Send a message:**
    - `POST /api/events/chat`
    - Body:
      ```json
      {
        "eventId": "<event_id>",
        "message": "Hello!"
      }
      ```
    - Auth required: Must be a participant of the event.
  - **Fetch messages:**
    - `GET /api/events/chat?eventId=<event_id>&page=1&limit=20`
    - Auth required: Must be a participant of the event.
    - Returns paginated, most recent messages first, with sender info populated.
- **Middleware:**
  - All chat endpoints are protected by authentication (JWT), rate limiting, and logging.
  - Only event participants can access chat.
- **Notes:**
  - Chat is always event-specific. Messages for one event are never shown on another event. Most recent messages are shown first. Supports pagination.
  - The storage model is optimized for unlimited scalability: each message is a separate document with an eventId field.

---
