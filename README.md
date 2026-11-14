# ThreadHive

ThreadHive is a full-stack, Reddit-style forum. Users register and log in, browse subreddits (communities), create threads, comment, and upvote or downvote threads and comments. The backend also exposes AI helpers (thread summaries and text rephrasing) backed by Gemini by default, with OpenAI available as an alternative.

**Stack:** React 19 + Vite + Redux Toolkit + React-Bootstrap (frontend); Node.js + Express 5 + Mongoose/MongoDB (backend); Vitest for tests.

This folder is the Week 10 starter code. The assignment brief is in `WK10_MLS_Problem_Statement.pdf`, and the prompts used for the exercises are in `resources/prompts.md`.

## Project layout

```
.
├── threadhive-backend/      Express API
│   ├── main.js              Entry point: loads .env, connects to MongoDB, starts the server
│   ├── server.js, db.js     Server start/stop and MongoDB connection helpers
│   ├── src/
│   │   ├── app.js           Express app: helmet, rate limit (100 req / 15 min per IP), CORS, routes
│   │   ├── routes/          auth, threads, comments, subreddits, votes
│   │   ├── controllers/     Request handlers
│   │   ├── services/        Business logic
│   │   ├── models/          Mongoose models: User, Subreddit, Thread, Comment
│   │   ├── middleware/      authHandler (JWT), errorHandler
│   │   ├── utils/           aiProvider.js (AI client), createAppError.js
│   │   └── scripts/         populate_db.js and seed-data.js
│   └── tests/               Vitest unit and integration tests (supertest, in-memory MongoDB)
├── threadhive-frontend/     React + Vite single-page app
│   ├── src/
│   │   ├── api/             Axios instance (base URL)
│   │   ├── config/          API endpoint paths (apiConfig.js)
│   │   ├── components/, pages/ (Auth, User), reducers/, store/, services/, utils/
│   └── tests/               Vitest + Testing Library + MSW
├── resources/               Exercise prompts and an image asset
└── WK10_MLS_Problem_Statement.pdf
```

## Prerequisites

- Node.js and npm (the repo does not pin a version; a current LTS release works with Express 5, Vite 6, and Vitest 3)
- A running MongoDB instance (local or hosted) for development. Backend tests use `mongodb-memory-server` and do not need one.
- An API key for Gemini (default) or OpenAI, only if you want the AI features (summary, rephrase)

## Setup

### Backend

```bash
cd threadhive-backend
npm install
cp .env.example .env
```

Edit `.env`:

| Variable                         | Purpose                                                                                                                                         |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `MONGODB_URI`                    | MongoDB connection string (example: `mongodb://localhost:27017/your_database_name`)                                                             |
| `PORT`                           | API port. Set it to `5000`: the frontend is hard-coded to `http://localhost:5000/api`, while the server falls back to `3000` if `PORT` is unset |
| `JWT_SECRET`                     | Secret used to sign auth tokens                                                                                                                 |
| `GEMINI_API_KEY`, `GEMINI_MODEL` | Gemini credentials and model (default model `gemini-2.5-flash`)                                                                                 |
| `OPENAI_API_KEY`, `OPENAI_MODEL` | OpenAI credentials and model (default model `gpt-4o`); only used if you switch providers                                                        |

Do not commit `.env` or real keys.

To switch AI providers, edit `src/utils/aiProvider.js`: comment out the active provider block and uncomment the other, then restart the server. Gemini is active by default.

### Frontend

```bash
cd threadhive-frontend
npm install
```

If your backend runs on a different host or port, change `baseURL` in `src/api/axiosInstance.js` (it is currently `http://localhost:5000/api`).

## Running

Start the backend and frontend in separate terminals.

```bash
# Backend (from threadhive-backend)
npm run dev        # nodemon, auto-restart
npm start          # plain node
npm run populate   # optional: clear and seed the database with sample users, subreddits, threads, comments

# Frontend (from threadhive-frontend)
npm run dev        # Vite dev server (default http://localhost:5173)
npm run build      # production build
npm run preview    # serve the production build
```

`npm run populate` deletes existing data in the database before seeding, so do not run it against data you want to keep.

## Testing and linting

```bash
# Backend (from threadhive-backend)
npm test           # vitest run; unit and integration tests
npm run format     # prettier --write .

# Frontend (from threadhive-frontend)
npm test           # vitest (watch mode)
npm run test:ui    # Vitest UI
npm run test:coverage
npm run lint       # eslint
```

## API overview

All routes are under `/api`. Except for register and login, every route requires a JWT via the `authHandler` middleware.

| Area       | Endpoints                                                                                                                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Auth       | `POST /api/auth/register`, `POST /api/auth/login`                                                                                                                                          |
| Threads    | `GET /api/threads`, `GET /api/threads/:id`, `POST /api/threads`, `PUT /api/threads/:id`, `DELETE /api/threads/:id`, `GET /api/threads/:id/summary` (AI), `POST /api/threads/rephrase` (AI) |
| Comments   | `GET /api/comments/thread/:threadId`, `POST /api/comments`                                                                                                                                 |
| Subreddits | `GET /api/subreddits`, `GET /api/subreddits/:id`, `POST /api/subreddits`                                                                                                                   |
| Votes      | `POST /api/threads/:id/upvote`, `POST /api/threads/:id/downvote`, `POST /api/comments/:id/upvote`, `POST /api/comments/:id/downvote`                                                       |

## Known limitations

- `threadhive-frontend/src/config/apiConfig.js` lists endpoints that have no matching backend route: `USER_API` (`/users/...`) and `/subreddits/:id/threads`. The backend serves a subreddit's threads from `GET /api/subreddits/:id`.
- The backend `main.js` defines `stopApp` but never calls it, so there is no graceful shutdown handler.
