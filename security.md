# Security Guide for Developers

This document provides security guidelines, best practices, and implementation details for the Event Management App backend. It covers authentication, authorization, input validation, sensitive data handling, and how to safely test or bypass security for development purposes.

---

## 1. Authentication
- All protected endpoints require a valid JWT in the `Authorization: Bearer <token>` header.
- JWTs are signed with the `JWT_SECRET` environment variable. Never commit secrets to version control.
- Tokens expire after 7 days by default.
- For production, consider using httpOnly cookies for JWT storage to mitigate XSS.
- **Email Verification:** All users (user, admin, community admin) must verify their email via a link or 6-digit code sent to their email address before login is allowed or notifications are sent.
- **Verification Endpoints:**
  - By link: `/api/auth/verify-email`, `/api/auth/verify-email-admin`, `/api/auth/verify-email-community-admin`
  - By code: `POST /api/auth/verify-email-code` (user/admin), `POST /api/auth/verify-email-code-community-admin` (community admin)
  - **Resend:** `POST /api/auth/resend-verification` with `{ "email": "..." }` (max once every 2 minutes, only if not verified)
- After successful verification, the verification code and token are removed from the database for security.

### How to Use
- Obtain a token by logging in via `/api/auth/login`, `/api/auth/admin-register`, or `/api/auth/community-admin-register`.
- Include the token in the `Authorization` header for all API requests.
- After registration, check your email and verify your account before attempting to log in.

### How to Bypass (for local development only)
- You may hardcode a valid JWT for your test user/admin and use it in API requests.
- Never disable authentication in production.

---

## 2. Authorization
- Role-based access control is enforced using the `authorize` middleware.
- Only users with the required role (e.g., `admin`, `community_admin`) can access certain endpoints.
- Event-based access: Only the event creator or an admin can approve payments or view all payments for an event.

### How to Use
- Assign roles at registration or via the admin panel.
- Use the correct token for the user role you want to test.

### How to Bypass (for local development only)
- You may modify the `authorize` middleware to allow all roles, but revert before production.

---

## 3. Input Validation & Sanitization
- All registration and login endpoints sanitize and validate emails.
- Passwords are trimmed and must be at least 8 characters for password changes.
- Never trust client input. Always validate and sanitize on the server.

---

## 4. Sensitive Data Handling
- Passwords are hashed with bcrypt before storage.
- Never return password hashes or sensitive fields in API responses.
- User queries use `.select('-password')` to exclude passwords.

---

## 5. Error Handling
- Error messages do not leak sensitive details (e.g., whether a user exists or not).
- All authentication errors return generic messages.

---

## 6. Rate Limiting & Account Lockout (TODO)
- Not yet implemented. For production, add rate limiting to login and registration endpoints to prevent brute-force attacks.
- Consider account lockout after repeated failed login attempts.

---

## 7. CORS & Security Headers (TODO)
- For production, configure CORS to only allow trusted origins.
- Use security headers (e.g., Helmet) to protect against common web vulnerabilities.

---

## 8. Testing Security
- Use tools like Postman or curl to test endpoints with and without valid tokens.
- Try to access endpoints with insufficient roles to verify authorization.
- Attempt invalid input to ensure validation is enforced.

---

## 9. Known Bypass Methods (for local/dev only)
- Hardcode a valid JWT for testing.
- Temporarily relax the `authorize` middleware.
- Use test users with elevated roles.

**Never use these bypasses in production.**

---

## 10. Reporting Vulnerabilities
- If you discover a security issue, report it to the project maintainer immediately.
- Do not disclose vulnerabilities publicly until they are fixed.

---

## 11. References
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html)

---

## 12. API Usage for All Clients (Apps, Websites, Integrations)
- All API endpoints are designed to be securely consumed by any client: web apps, mobile apps, or third-party integrations.
- Always use the documented authentication (JWT in Authorization header) and authorization (role-based) methods for every request.
- Never bypass authentication or authorization in production, regardless of the client type.
- Validate and sanitize all input on both client and server sides.
- Follow RESTful best practices for all API integrations.
- If building a public API, consider adding rate limiting, CORS restrictions, and API key support.

**For development, you may use the bypass/testing methods described above, but always revert to secure practices before deploying to production.**

---

## 13. Error Handling (Required)
- Every API endpoint and main code must use try/catch for error handling.
- All errors should be logged on the server for debugging.
- API responses must return appropriate HTTP status codes and generic error messages (never leak stack traces or sensitive details).
- See main code for the error handling pattern to follow.

---

## 14. Notification Security
- All in-app notifications are also sent to the user's/admin's/community admin's verified email address.
- Email notifications are only sent to verified email addresses.
