# ⚡ Introduction to Express Middleware

An interactive Reveal.js presentation covering Express middleware — from the request pipeline and built-in middleware through to authentication, validation, error handling, composition patterns, and testing.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Introduction_to_Express_Middleware/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Introduction to Express Middleware |
| 02 | Agenda | Overview of all topics covered |
| 03 | What Is Middleware? | The (req, res, next) pattern, pipeline concept |
| 04 | How the Middleware Stack Works | Execution order, next() chain, request lifecycle diagram |
| 05 | Application-Level Middleware | app.use, app.get, mounting paths |
| 06 | Router-Level Middleware | express.Router(), modular route files |
| 07 | Built-In Middleware | express.json(), express.urlencoded(), express.static() |
| 08 | Error-Handling Middleware | 4-argument signature, centralised error handling |
| 09 | Third-Party Middleware | helmet, cors, morgan, compression, rate-limit, cookie-parser |
| 10 | Writing Custom Middleware | Logging, timing, request ID, feature flags |
| 11 | Authentication Middleware | JWT verification, session-based auth, passport.js strategies |
| 12 | Authorisation Middleware | Role-based access, permission guards, policy patterns |
| 13 | Validation Middleware | express-validator, Joi, Zod schema validation |
| 14 | Request Processing Pipeline | body parsing → auth → validate → handle → error |
| 15 | Async Middleware & Error Propagation | Wrapping async handlers, promise rejection |
| 16 | Middleware Composition Patterns | Factory functions, conditional middleware, chaining |
| 17 | Testing Middleware | Unit testing with mocks, integration testing with supertest |
| 18 | Performance Considerations | Middleware ordering, short-circuiting, avoiding unnecessary work |
| 19 | Summary & Next Steps | Core takeaways, best practices, resources, key packages |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

- [Express.js Using Middleware Guide](https://expressjs.com/en/guide/using-middleware.html) — official middleware documentation
- [Express.js Error Handling Guide](https://expressjs.com/en/guide/error-handling.html) — error-handling middleware patterns
- [Helmet.js](https://helmetjs.github.io/) — security HTTP headers middleware
- [cors](https://github.com/expressjs/cors) — Cross-Origin Resource Sharing middleware
- [express-rate-limit](https://github.com/express-rate-limit/express-rate-limit) — rate limiting middleware
- [express-validator](https://express-validator.github.io/) — validation and sanitisation middleware
- [Zod](https://zod.dev/) — TypeScript-first schema validation
- [Joi](https://joi.dev/) — schema description and data validation
- [Passport.js](https://www.passportjs.org/) — authentication middleware with 500+ strategies
- [supertest](https://github.com/ladjs/supertest) — HTTP integration testing
- [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) — JWT signing and verification

## License

Educational use. Code examples provided as-is.
