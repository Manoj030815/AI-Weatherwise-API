# AI WeatherWise API — Technical Documentation

**Package:** `ai-weatherwise-api` v1.0.0
**Stack:** Node.js · Express 4 · MongoDB (Mongoose 8) · JWT · bcryptjs · Google Gemini (`@google/generative-ai`)
**Entry point:** `src/server.js`

---

## 1. Overview

AI WeatherWise is a REST API that lets users register, log in, save favorite cities, look up current weather, and get short AI-generated weather summaries and recommendations.

Two design choices worth knowing up front:

- **Graceful fallbacks.** If `OPENWEATHER_API_KEY` or `GEMINI_API_KEY` is missing or a placeholder, the API returns deterministic mock weather and rule-based text instead of failing. The whole API can be tested with no external keys.
- **MVC-style layout** with a separate `services/` layer for anything that talks to third parties.

---

## 2. Project Structure

```
Backend/
├── package.json
├── postman_collection.json
├── README.md
├── .env / .gitignore
└── src/
    ├── server.js                 # Connects DB, then starts HTTP server
    ├── app.js                    # Express app, middleware, route mounting, error handlers
    ├── config/db.js              # Mongoose connection
    ├── models/
    │   ├── User.js               # name, email, password (hashed on save)
    │   └── Location.js           # city, country, user ref
    ├── middleware/authMiddleware.js   # `protect` – JWT guard
    ├── routes/                   # auth, locations, weather, ai
    ├── controllers/              # request validation + response shaping
    └── services/
        ├── weatherService.js     # OpenWeatherMap + mock fallback
        └── aiService.js          # Gemini + rule-based fallback
```

---

## 3. Setup

**Requirements:** Node.js 18+ (the weather service uses the built-in global `fetch`), npm, and a MongoDB instance (local or Atlas).

```bash
npm install
npm run dev     # node --watch src/server.js
npm start       # node src/server.js
```

### Environment variables (`.env`)

| Variable | Purpose | Default if unset |
| :-- | :-- | :-- |
| `PORT` | HTTP port | `5000` |
| `MONGO_URI` | MongoDB connection string | `mongodb://127.0.0.1:27017/weatherwise` |
| `JWT_SECRET` | Secret for signing/verifying JWTs | Hardcoded fallback string (see §9) |
| `OPENWEATHER_API_KEY` | OpenWeatherMap key | Mock weather used |
| `GEMINI_API_KEY` | Google Gemini key | Rule-based text used |

---

## 4. Startup Flow

1. `server.js` requires `app.js`, which loads `dotenv`, registers middleware (`cors()`, `express.json()`), mounts routes, then adds a 404 handler and a central error handler.
2. `server.js` calls `connectDB()`. On failure the process exits with code 1.
3. Only after the DB connects does `app.listen(PORT)` run.

---

## 5. Data Models

### User
| Field | Type | Rules |
| :-- | :-- | :-- |
| `name` | String | required, trimmed |
| `email` | String | required, unique, lowercased, regex-validated |
| `password` | String | required, min 6 chars, **bcrypt-hashed (10 salt rounds) in a `pre('save')` hook** |
| `createdAt` / `updatedAt` | Date | automatic timestamps |

Method: `comparePassword(candidate)` — bcrypt comparison.

### Location
| Field | Type | Rules |
| :-- | :-- | :-- |
| `city` | String | required, trimmed |
| `country` | String | required, trimmed |
| `user` | ObjectId → `User` | required |
| `createdAt` / `updatedAt` | Date | automatic timestamps |

Compound **unique index** on `(user, city, country)` prevents duplicate favorites per user. Controllers additionally do a case-insensitive check before writing, since the index alone is case-sensitive.

---

## 6. Authentication

- Login/register return a JWT signed with payload `{ id }`, valid for **30 days**.
- Protected routes expect `Authorization: Bearer <token>`.
- The `protect` middleware verifies the token, loads the user (password excluded), and attaches it to `req.user`. It returns **401** if the header is missing, the token is invalid/expired, or the user no longer exists.

---

## 7. API Reference

Base URL: `http://localhost:5000`
All responses use `{ "success": boolean, ... }`. Errors include a `message`.

### 7.1 Auth — `/api/auth`

| Method | Path | Auth | Body |
| :-- | :-- | :-- | :-- |
| POST | `/register` | Public | `name`, `email`, `password` |
| POST | `/login` | Public | `email`, `password` |
| GET | `/profile` | JWT | — |

**POST /api/auth/register** → `201`
```json
{ "success": true, "data": { "_id": "...", "name": "Manoj", "email": "m@example.com", "token": "<jwt>" } }
```
Errors: `400` missing fields / password < 6 chars / user exists; `500` server error.

**POST /api/auth/login** → `200` (same shape as above). Errors: `400` missing fields; `401` invalid credentials (same message for unknown email and wrong password, which avoids leaking which one was wrong).

**GET /api/auth/profile** → `200`
```json
{ "success": true, "data": { "_id": "...", "name": "...", "email": "...", "createdAt": "..." } }
```

### 7.2 Locations — `/api/locations` (all require JWT)

| Method | Path | Body | Description |
| :-- | :-- | :-- | :-- |
| POST | `/` | `city`, `country` | Add favorite → `201` |
| GET | `/` | — | List user's favorites, newest first → `200` with `count` and `data` |
| PUT | `/:id` | `city`, `country` | Update a favorite |
| DELETE | `/:id` | — | Remove a favorite |

Error codes: `400` missing fields or duplicate; `404` not found; `401` if the location belongs to another user (update/delete); `500` server error.

### 7.3 Weather — `/api/weather`

| Method | Path | Auth |
| :-- | :-- | :-- |
| GET | `/:city` | Public |

```json
{
  "success": true,
  "data": { "city": "Hyderabad", "temperature": 29.4, "humidity": 62, "windSpeed": 3.6, "condition": "Clouds", "isMock": false }
}
```

- Data comes from OpenWeatherMap (`units=metric`, so °C and m/s).
- `isMock: true` means the mock generator was used. This happens with no/placeholder key, an invalid key (HTTP 401 from OpenWeatherMap), or any non-404 upstream failure.
- Mock values are derived from a hash of the city name, so the same city always returns the same result (temp 15–34 °C, humidity 40–89 %, wind 3–17 m/s).
- Unknown city → `404`.

### 7.4 AI Insights — `/api/ai` (all require JWT)

**POST /api/ai/weather-summary**
Body: `city`, `temperature`, `humidity`, `condition` (all required; `0` is accepted for numbers).
```json
{ "success": true, "summary": "Hyderabad is warm and humid today with cloudy skies." }
```

**POST /api/ai/weather-recommendation**
Body: `temperature`, `condition`.
```json
{ "success": true, "recommendation": "Stay hydrated, wear light cotton clothes, and a light jacket might be handy." }
```

Errors: `400` missing fields; `500` server error.

**How generation works** (`aiService.js`): if a real `GEMINI_API_KEY` is set, it calls model `gemini-2.5-flash` with a prompt asking for 1–2 plain-text sentences. On any error or empty response it falls back to a rule-based generator (thresholds: ≥30 °C warm, ≤15 °C chilly; humidity ≥70 high, ≤40 low; extra advice for rain, clouds, snow). The response does not indicate which path was used.

---

## 8. Error Handling

- Controllers wrap logic in `try/catch` and return JSON errors.
- `app.js` adds a **404 handler** (`Route not found: METHOD /path`) and a **central error handler** that logs the stack and returns `err.status || 500`.

---

## 9. Review Notes: Security & Improvements

Things I noticed while reading the code, roughly in priority order.

**Security**

1. **Real secrets in the zip.** The uploaded `.env` contains what look like real values for `MONGO_URI`, `OPENWEATHER_API_KEY`, and `GEMINI_API_KEY`. Since the archive has been shared, rotate those keys and never include `.env` in archives (it is already in `.gitignore`, which protects git only).
2. **Weak JWT secret.** `JWT_SECRET` in `.env` appears to be the placeholder from the README, and the code also falls back to a hardcoded secret if the variable is missing. Anyone who knows that string can forge tokens. Use a long random value and make the app refuse to start without it.
3. **Regex built from user input** (`locationController.js`). `new RegExp(\`^${city}$\`, 'i')` doesn't escape special characters, so input like `.*` or a crafted pattern can match unintended documents or cause slow queries. Escape the input, or store a lowercased `normalized` field and rely on the unique index.
4. **No rate limiting.** `/login`, `/register`, and especially the AI endpoints (which cost money per call) should be rate-limited (e.g. `express-rate-limit`).
5. **Open CORS.** `cors()` allows every origin. Restrict it to your frontend's origin in production.
6. **Add `helmet`** for standard security headers.

**Correctness / robustness**

7. **Email regex rejects valid addresses.** The pattern only allows TLDs of 2–3 characters, so `.info`, `.tech`, `.museum` etc. fail. Consider `validator.isEmail`.
8. **Status codes.** Updating/deleting another user's location returns `401`; `403` is the conventional code. Duplicate resources are usually `409`.
9. **Invalid ObjectId** in `/api/locations/:id` throws a CastError that surfaces as a generic `500`. Validate with `mongoose.isValidObjectId` and return `400`.
10. **`protect` middleware** checks `startsWith('Bearer')` without the trailing space; harmless but sloppy.
11. **Mock fallback is silent for AI.** Weather exposes `isMock`; AI responses don't say whether Gemini or the rule-based path answered.
12. **AI endpoints trust client-supplied weather data** and put `city`/`condition` straight into the prompt (prompt injection surface, no length limits). Validate types/lengths, or have the server fetch weather itself.
13. **`nodemon` is in `dependencies`** but the `dev` script uses `node --watch`. Remove it or move it to `devDependencies`.
14. **README mentions `.env.example`, but none exists.** Add one with placeholder values.
15. **No tests, no request-body schema validation** (consider `zod`/`joi`), and no `/health` endpoint.

---

## 10. Testing with Postman

Import `postman_collection.json`. It uses variables `{{base_url}}`, `{{token}}` (auto-saved after register/login) and `{{location_id}}`. It covers 10 requests:

Auth (register, login, profile) · Locations (add, list, update, delete) · Weather (current weather) · AI Insights (summary, recommendation)

Suggested flow: Register → (token saved automatically) → Add location → copy its `_id` into `location_id` → call the rest.

---

## 11. Quick cURL Examples

```bash
# Register
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Manoj","email":"m@example.com","password":"secret123"}'

# Weather (public)
curl http://localhost:5000/api/weather/Hyderabad

# Add favorite (replace TOKEN)
curl -X POST http://localhost:5000/api/locations \
  -H "Authorization: Bearer TOKEN" -H "Content-Type: application/json" \
  -d '{"city":"Hyderabad","country":"India"}'

# AI summary
curl -X POST http://localhost:5000/api/ai/weather-summary \
  -H "Authorization: Bearer TOKEN" -H "Content-Type: application/json" \
  -d '{"city":"Hyderabad","temperature":31,"humidity":60,"condition":"Clouds"}'
```
