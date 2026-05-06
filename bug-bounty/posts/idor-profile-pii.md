# IDOR on Profile Picture Endpoint Exposed Any User's PII

## Summary

A classic but impactful IDOR. The `/api/v1/users/{id}/profile-picture` endpoint accepted any numeric user ID without validating session ownership — meaning any authenticated user could fetch another user's profile photo, display name, and email.

## Steps to Reproduce

1. Log in as **User A** and capture a request to update your profile picture.
2. Note the `user_id` parameter in the request body.
3. Replace it with any other valid numeric ID — try incrementing by 1.
4. The response returns the target user's profile data including their email address.

```http
GET /api/v1/users/10482/profile-picture HTTP/1.1
Host: target.com
Authorization: Bearer <User A token>
```

Response:

```json
{
  "id": 10482,
  "email": "victim@target.com",
  "display_name": "Victim User",
  "avatar_url": "https://cdn.target.com/avatars/10482.jpg"
}
```

## Impact

An attacker could enumerate all user IDs (they were sequential) and harvest the full user database — emails, display names, and avatar URLs. Combined with phishing, the blast radius is significant.

## Root Cause

No server-side ownership check. The backend queried by ID directly without asserting `session.user_id === requested_id`.

## Fix

Validate that the authenticated user's session ID matches the requested resource ID before returning data. Alternatively, derive the user ID from the session token server-side and never accept it as user-supplied input.

---

*Triaged as P1 · Resolved · Bounty awarded.*
