# Complete API Reference & Developer Cheat-Sheet

This reference catalogs every REST endpoint, request/response DTO payload, environment variable, and feature flag in **AuthForge**, with technical descriptions and cURL / HTTP examples.

---

## Table of Contents

1. [Authentication Endpoints (`/api/auth`)](#1-authentication-endpoints-apiauth)
2. [Two-Factor Authentication (`/api/2fa`)](#2-two-factor-authentication-api2fa)
3. [Administration Endpoints (`/api/admin`)](#3-administration-endpoints-apiadmin)
4. [Environment Variables Reference](#4-environment-variables-reference)
5. [Feature Flags Matrix](#5-feature-flags-matrix)
6. [Maven Dependency & Starter Usage](#6-maven-dependency--starter-usage)

---

## 1. Authentication Endpoints (`/api/auth`)

All public authentication routes. Protected against brute-force attacks via token-bucket rate limiting (Bucket4j).

### `POST /api/auth/register`

* **Summary**: Registers a new user account.
* **Headers**: `Content-Type: application/json`
* **Request Body**:
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "password": "SecurePassword123!"
}
```
* **Response (201 Created)**:
```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "d8f1e2c3-...",
  "tokenType": "Bearer",
  "user": {
    "id": 1,
    "name": "Jane Doe",
    "email": "jane@example.com",
    "role": "USER",
    "twoFactorEnabled": false,
    "emailVerified": false
  }
}
```
* **Example cURL**:
```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane Doe","email":"jane@example.com","password":"SecurePassword123!"}'
```

---

### `POST /api/auth/login`

* **Summary**: Authenticates credentials. If the user has 2FA enabled, returns `twoFactorRequired: true` without access tokens.
* **Request Body**:
```json
{
  "email": "jane@example.com",
  "password": "SecurePassword123!"
}
```
* **Response (200 OK - Standard Login)**:
```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "d8f1e2c3-...",
  "tokenType": "Bearer",
  "user": { "id": 1, "name": "Jane Doe", "email": "jane@example.com", "role": "USER" }
}
```
* **Response (200 OK - When 2FA is active)**:
```json
{
  "twoFactorRequired": true,
  "message": "Two-factor authentication code required"
}
```

---

### `POST /api/auth/2fa/verify`

* **Summary**: Completes authentication for 2FA-enabled accounts using a 6-digit TOTP code.
* **Request Body**:
```json
{
  "email": "jane@example.com",
  "code": "481920"
}
```
* **Response (200 OK)**: Returns standard `AuthResponse` with access & refresh tokens.

---

### `POST /api/auth/refresh`

* **Summary**: Rotates and refreshes the access token using a valid refresh token.
* **Request Body**:
```json
{
  "refreshToken": "d8f1e2c3-4a5b-6c7d-8e9f-0a1b2c3d4e5f"
}
```
* **Response (200 OK)**:
```json
{
  "accessToken": "eyJhbGciOi...new-token...",
  "refreshToken": "e9a2b3c4-...rotated-refresh-token...",
  "tokenType": "Bearer"
}
```

---

### `POST /api/auth/logout`

* **Summary**: Invalidates the active refresh token and terminates the session.
* **Security**: `Authorization: Bearer <accessToken>`
* **Response (200 OK)**:
```json
{
  "message": "Logged out successfully"
}
```

---

### `POST /api/auth/forgot-password` & `POST /api/auth/reset-password`

* **Forgot Password Body**: `{"email": "jane@example.com"}` (Dispatches HTML reset link via MailHog/SMTP).
* **Reset Password Body**:
```json
{
  "token": "reset-token-uuid",
  "newPassword": "BrandNewPassword123!"
}
```

---

## 2. Two-Factor Authentication (`/api/2fa`)

All routes require `Authorization: Bearer <accessToken>`.

| Method & Endpoint | Description | Response Model |
| :--- | :--- | :--- |
| **`GET /api/2fa/setup`** | Generates a new secret key & Google Authenticator QR URI. | `{"secret": "JBSWY3DPEHPK3PXP", "qrCodeUri": "otpauth://totp/AuthForge:jane?secret=..."}` |
| **`POST /api/2fa/enable`** | Validates the user's initial code and permanently enables 2FA. | `{"message": "Two-factor authentication enabled successfully"}` |
| **`POST /api/2fa/disable`** | Verifies current TOTP code and disables 2FA. | `{"message": "Two-factor authentication disabled"}` |

---

## 3. Administration Endpoints (`/api/admin`)

Requires `ADMIN` role (`Authorization: Bearer <adminAccessToken>`).

### `GET /api/admin/users`

Returns all registered users, roles, and verification statuses:
```json
[
  { "id": 1, "name": "Admin User", "email": "admin@example.com", "role": "ADMIN" },
  { "id": 2, "name": "Jane Doe", "email": "jane@example.com", "role": "USER" }
]
```

### `PUT /api/admin/users/{id}/role`

Changes a user's authorization role:
```json
{ "role": "ADMIN" }
```

### `GET /api/admin/features`

Returns real-time status of all active feature flags:
```json
{
  "oauth2": true,
  "twoFactor": true,
  "rateLimiting": true,
  "emailVerification": true
}
```

---

## 4. Environment Variables Reference

Configure AuthForge dynamically in Docker, Kubernetes, or cloud deployments:

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `DB_URL` | `jdbc:postgresql://localhost:5432/authforge` | PostgreSQL JDBC connection URL. |
| `DB_USERNAME` | `authforge` | Database user. |
| `DB_PASSWORD` | `authforge` | Database password. |
| `JWT_SECRET` | *(256-bit default string)* | Secret key for signing HMAC-SHA256 JWT tokens. |
| `RATE_LIMIT_RPM` | `30` | Max requests per minute per IP on authentication routes. |
| `GOOGLE_CLIENT_ID` | `google-client-id` | Google OAuth2 credentials. |
| `GOOGLE_CLIENT_SECRET` | `google-client-secret` | Google OAuth2 secret. |
| `GITHUB_CLIENT_ID` | `github-client-id` | GitHub OAuth2 credentials. |
| `GITHUB_CLIENT_SECRET` | `github-client-secret` | GitHub OAuth2 secret. |
| `MAIL_HOST` / `MAIL_PORT` | `localhost` / `1025` | SMTP server coordinates (pre-wired to MailHog in Docker). |
| `CORS_ORIGINS` | `http://localhost:4000` | Allowed origins for web and mobile frontends. |

---

## 5. Feature Flags Matrix

Toggle security subsystems without modifying code via environment variables:

| Flag Name | Env Variable | Default | What happens when `false`? |
| :--- | :--- | :--- | :--- |
| **OAuth2 Social** | `FEATURE_OAUTH2` | `true` | Hides OAuth2 buttons and disables social callbacks. |
| **Two-Factor Auth** | `FEATURE_2FA` | `true` | Disables TOTP setup and bypasses 2FA login checks. |
| **Rate Limiting** | `FEATURE_RATE_LIMIT` | `true` | Bypasses Bucket4j IP throttling on `/api/auth/**`. |
| **Email Service** | `FEATURE_EMAIL` | `true` | Auto-verifies users on registration without SMTP tokens. |

---

## 6. Maven Dependency & Starter Usage

To import AuthForge components into an existing Spring Boot application:

```xml
<dependency>
    <groupId>io.github.frodygr</groupId>
    <artifactId>authforge</artifactId>
    <version>2.0.0</version>
</dependency>
```
