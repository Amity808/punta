# 08 — Security & Input Validation

This document outlines the security architecture of the **CareerLift** application, focusing on authentication, request verification, input validation, and rate limiting.

---

## 1. Authentication & Session Control

CareerLift uses stateless JSON Web Token (JWT) authentication.

```
┌──────────┐              Credentials             ┌──────────┐
│          ├─────────────────────────────────────▶│          │
│          │                                      │          │
│  Client  │         JWT (Expires in 24h)         │  Server  │
│          │◀─────────────────────────────────────┤          │
│          │                                      │          │
│          │          Authorization Header        │          │
│          ├─────────────────────────────────────▶│          │
└──────────┘       Bearer <eyJhbGciOiJIUzI1...>   └──────────┘
```

### Authentication Logic
* **Password Hashing:** Passwords are encrypted before database insertion using **argon2id**, which provides strong defense against GPU-based brute-force attacks.
* **Token Issuance:** Upon successful authentication (`/auth/login` or `/auth/register`), the server generates a JWT signed with a high-entropy secret (`JWT_SECRET`).
* **Expiration:** Tokens are set to expire in 24 hours. If a token is compromised, its impact is limited by this short lifespan.
* **Storage:** The client SPA stores the token in memory, caching it securely in the Zustand store which syncs to local memory.

---

## 2. Input Validation (Zod Validation Middleware)

All client inputs are validated at the API boundary using **Zod** schemas. Requests that do not match the expected schema are rejected before reaching database controllers.

```typescript
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export const validateRequest = (schema: AnyZodObject) => 
  async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      // Validate body, query, and params
      const parsed = await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      // Replace request with typed data
      req.body = parsed.body;
      req.query = parsed.query;
      req.params = parsed.params;
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        res.status(400).json({
          status: 'error',
          message: 'Validation failed',
          errors: error.errors.map(err => ({
            field: err.path.join('.'),
            message: err.message
          }))
        });
        return;
      }
      next(error);
    }
  };
```

### Example Schema: Registration Input Validation
```typescript
import { z } from 'zod';

export const registerSchema = z.object({
  body: z.object({
    fullName: z.string().min(2, 'Name must be at least 2 characters'),
    email: z.string().email('Invalid email address format'),
    password: z.string()
      .min(8, 'Password must be at least 8 characters long')
      .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
      .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
      .regex(/[0-9]/, 'Password must contain at least one number')
  })
});
```

---

## 3. Rate Limiting Middleware

To prevent automated denial-of-service (DoS) attacks and brute-force credential stuffing, the server implements IP-based rate limiting using `express-rate-limit`.

```typescript
import rateLimit from 'express-rate-limit';

// Global API limiter
export const apiRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per window
  standardHeaders: true, // Return rate limit info in the `RateLimit-*` headers
  legacyHeaders: false, // Disable the `X-RateLimit-*` headers
  message: {
    status: 'error',
    message: 'Too many requests from this IP, please try again after 15 minutes.'
  }
});

// Stricter limiter for authentication endpoints
export const authRateLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 10, // Limit each IP to 10 login/register requests per hour
  standardHeaders: true,
  legacyHeaders: false,
  message: {
    status: 'error',
    message: 'Too many login attempts from this IP, please try again after an hour.'
  }
});
```

---

## 4. Database Security & Relational Isolation

* **SQL Injection Prevention:** CareerLift uses **Drizzle ORM** for query building, which automatically parameterizes inputs, preventing SQL injection vulnerabilities.
* **Relational Safety:** The database schema enforces strict cascading checks (`ON DELETE CASCADE`) to clean up records and prevent orphan rows in bookmarks, responses, and user profile tables.
* **Anonymous Submissions:** To protect candidate privacy, users can choose to submit company interview reports anonymously. If `is_anonymous` is checked, the backend sets the `user_id` field to `NULL` to decouple the contribution from the user profile, while still awarding points to the author's internal account ledger.
