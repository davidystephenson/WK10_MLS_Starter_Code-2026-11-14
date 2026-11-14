# Developer Guide

This guide is for people changing ThreadHive. For what the project is, the full setup steps, the endpoint list, and the layout tree, see [README.md](README.md). This file covers how the pieces fit together and how to work on them.

## Local workflow

1. Follow the setup in [README.md](README.md#setup) (install both packages, create `threadhive-backend/.env` from `.env.example`).
2. Make sure MongoDB is reachable at `MONGODB_URI`.
3. Run the backend and frontend in two terminals:
   - `npm run dev` in `threadhive-backend` (nodemon on `main.js`)
   - `npm run dev` in `threadhive-frontend` (Vite, default `http://localhost:5173`)
4. Optionally run `npm run populate` in `threadhive-backend` to seed sample data. It deletes existing users, subreddits, threads, and comments first (see `src/scripts/populate_db.js`).
5. Register or log in through the UI. The frontend stores `token` and `user` in `localStorage` and sends `Authorization: Bearer <token>` on requests.

Keep `PORT=5000` in `.env` so the backend matches the frontend (see [Known issues](#known-issues-and-gotchas)).

## Architecture and request flow

```
Browser (React)
  page/component -> Redux thunk (src/reducers/*Slice.js)
    -> src/services/*Service.js -> axiosInstance (baseURL http://localhost:5000/api)
      -> Express app (threadhive-backend/src/app.js)
        helmet -> rate limit -> cors -> body parsers
        -> routes/*.js -> authHandler -> controllers/*.js -> services/*.js
          -> models/*.js (Mongoose) -> MongoDB
          -> utils/aiProvider.js -> Gemini or OpenAI   (summary and rephrase only)
      <- errorHandler -> JSON response
```

### Backend

- `main.js` loads `.env`, calls `connectToDB()` (`db.js`), then `startServer()` (`server.js`). A DB connection failure calls `process.exit(1)`.
- `src/app.js` builds the Express app and does not listen, which is why tests can import it directly with supertest. Mount points: `/api/threads`, `/api/subreddits`, `/api/auth`, `/api/comments`, and `/api` for the vote routes (`src/routes/votes.js` defines `/threads/:id/upvote` etc.).
- Layers:
  - Routes only map a path and method to `authHandler` plus a controller.
  - Controllers read `req`, call a service, and send `{ success, message, data }` JSON.
  - Services hold the logic and use the Mongoose models.
- Errors: services throw `createAppError(message, statusCode)` (`src/utils/createAppError.js`). `src/middleware/errorHandler.js` turns any error into `{ success: false, message }` with `err.statusCode` (or 500), and adds `stack` only when `NODE_ENV === "development"`. Controllers and `authHandler` are `async` and have no try/catch; they rely on Express 5 forwarding rejected promises to the error handler.
- Auth: `authHandler` reads the bearer token, verifies it with `JWT_SECRET`, loads the user, and sets `req.user = { userId }`. Tokens are issued by `authService.js` with a 1 day expiry.
- Rate limit: 100 requests per 15 minutes per IP, applied to every route.

### Frontend

- `src/main.jsx` and `src/App.jsx` set up the router and the Redux store (`src/store/store.js`: slices `auth`, `threads`, `selectedThread`, `comments`, `theme`, `subreddits`). `/` and unknown paths redirect to `/home`; `PrivateRoute` guards the signed-in pages.
- `src/api/axiosInstance.js` holds the base URL. `src/config/apiConfig.js` holds the endpoint paths. `src/services/*.js` call the API and unwrap `res.data.data`.
- `src/utils/handleApiError.js` clears `localStorage` and redirects to `/login` on a 401, otherwise returns the error message.

## Adding an endpoint

Example: a new thread-related endpoint. Use the existing files as templates (the comments files are the smallest).

1. **Model** (if new data is needed): add or edit a file in `threadhive-backend/src/models/`. If a new model must be registered at startup, see the model imports at the top of `src/app.js`.
2. **Service**: add a function in `src/services/<area>Service.js`. Validate input and throw `createAppError("...", 400)` for bad requests.
3. **Controller**: add a handler in `src/controllers/<area>Controller.js`:

   ```js
   export const myHandler = async (req, res) => {
     const data = await myService.doThing(req.params.id, req.user.userId);
     res.status(200).json({ success: true, message: "Done", data });
   };
   ```

4. **Route**: register it in `src/routes/<area>.js` with `authHandler` unless it must be public. Order matters in Express: static paths such as `/rephrase` are declared before `/:id` in `routes/threads.js`; do the same for new static paths.
5. **New router only**: create the router file and mount it in `src/app.js`.
6. **Tests**: add a service unit test, a controller unit test, and an integration test (see [Testing](#testing-linting-and-formatting)).
7. **Frontend** (if the UI needs it): add the path to `src/config/apiConfig.js`, a function in the matching `src/services/*Service.js`, and a thunk or reducer in `src/reducers/`. Add a handler to `tests/mocks/handlers.js` so MSW can serve it in tests.
8. Update the endpoint table in [README.md](README.md#api-overview).

## Testing, linting, and formatting

| Where    | Command                                    | Notes                                                                                                                                                                  |
| -------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Backend  | `npm test`                                 | `vitest run`; files matching `tests/**/*.test.js`; runs files serially (`fileParallelism: false`) with 60 s test and hook timeouts                                     |
| Backend  | `npm run format`                           | `prettier --write .` (writes files; no Prettier config file exists in the backend, so Prettier defaults apply)                                                         |
| Frontend | `npm test`                                 | `vitest` in watch mode; use `npx vitest run` for a single pass                                                                                                         |
| Frontend | `npm run test:ui`, `npm run test:coverage` | Vitest UI and coverage. `vitest --ui` needs `@vitest/ui` (a devDependency).  No coverage provider package is listed in `package.json`; this was not run |
| Frontend | `npm run lint`                             | `eslint .` using `eslint.config.js` (ESLint recommended, react-hooks, react-refresh; `no-unused-vars` is an error except names starting with a capital letter or `_`)  |

Backend test layout:

- `tests/setup.js` is the setup shared by the whole backend suite. It starts a `mongodb-memory-server` instance, connects Mongoose to it, and cleans threads and subreddits before and after. The first run may download a MongoDB binary, which is why the timeouts are long.
- `tests/unit/` covers controllers, services, middleware, and utils. `tests/integration/` drives `src/app.js` through supertest.
- Tests that call the AI path mock the module: `vi.mock` of `src/utils/aiProvider.js` with `generateAIContent: vi.fn()`. No real API key or network is needed.
- Several integration tests and `authService` tests read `process.env.JWT_SECRET` via `dotenv`, so a `.env` with `JWT_SECRET` should exist in `threadhive-backend` before running `npm test`.

Frontend test layout: `tests/unit/` (components, pages, reducers) and `tests/integration/` (flows), with MSW handlers in `tests/mocks/` and setup in `tests/setup.js`. The Vitest config (jsdom, globals, `setupFiles`) lives in `vite.config.js`.

## Working with the AI provider

All AI calls go through one function, `generateAIContent(prompt)` exported by `threadhive-backend/src/utils/aiProvider.js`. It takes a prompt string and returns the model's text. It is used only in `src/services/threadService.js`, for the thread summary (`GET /api/threads/:id/summary`) and rephrase (`POST /api/threads/rephrase`) services; both trim the result and throw a 500 `createAppError` when it is empty.

- **Switching provider:** the file contains two blocks, OpenAI (commented out) and Gemini (active). Comment out one block and uncomment the other, then restart. Both export the same `generateAIContent` signature. The OpenAI block's `import OpenAI from 'openai'` is also commented, so uncomment it together with the client code. Both SDKs are already in `package.json`.
- **Configuration:** `GEMINI_API_KEY` and `GEMINI_MODEL` (default `gemini-2.5-flash`), or `OPENAI_API_KEY` and `OPENAI_MODEL` (default `gpt-4o`).
- **Adding an AI feature:** build a prompt in a service, call `generateAIContent`, validate the result, and expose it through a controller and route as described above. Keep SDK-specific code inside `aiProvider.js` so the services stay provider-neutral.
- **Testing:** mock `aiProvider.js` as the existing tests do (see `tests/unit/services/threadService.test.js` and `tests/integration/thread.test.js`).

## Environment variables

Backend only; the frontend reads none (its API base URL is hard-coded). `.env` is gitignored. Template: `threadhive-backend/.env.example`.

| Variable                         | Used by                            | Notes                                                                            |
| -------------------------------- | ---------------------------------- | -------------------------------------------------------------------------------- |
| `MONGODB_URI`                    | `db.js`, `populate_db.js`          | Required for `npm start`, `npm run dev`, `npm run populate`                      |
| `PORT`                           | `server.js`                        | Falls back to `3000` if unset; `.env.example` sets `5000`                        |
| `JWT_SECRET`                     | `authService.js`, `authHandler.js` | Signs and verifies tokens                                                        |
| `GEMINI_API_KEY`, `GEMINI_MODEL` | `aiProvider.js`                    | Active provider; model defaults to `gemini-2.5-flash`                            |
| `OPENAI_API_KEY`, `OPENAI_MODEL` | `aiProvider.js` (commented block)  | Only after switching; model defaults to `gpt-4o`                                 |
| `NODE_ENV`                       | `errorHandler.js`                  | `development` adds the stack trace to error responses. Not set by any npm script |

## Known issues and gotchas

- **Port mismatch.** `server.js` defaults to port 3000, but the frontend's `baseURL` in `src/api/axiosInstance.js` is `http://localhost:5000/api`. Set `PORT=5000` in `.env` (the example file already does), or edit `baseURL`. Because the base URL is hard-coded there is no env-based override.
- **Dead endpoints in `apiConfig.js`.** `SUBREDDIT_API.GET_WITH_THREADS` (`/subreddits/:id/threads`) and all of `USER_API` (`/users/:id`, `/users/profile`) have no backend route, and nothing in `src/` or `tests/` references them. A subreddit's threads come from `GET /api/subreddits/:id` (`getSubredditWithThreads`). Also, the backend has `PUT` and `DELETE /api/threads/:id`, but `apiConfig.js` has no entries for them.
- **Unused `stopApp` in `main.js`.** It is defined but never called and not wired to `SIGINT`/`SIGTERM`, so nothing shuts the server or DB connection down gracefully. Note too that `stopServer()` in `server.js` does not await `server.close`.
- **Stale `main` field.** `threadhive-backend/package.json` has `"main": "app.js"`, which does not exist at the backend root (the app is `src/app.js`; the entry point is `main.js`). Scripts do not use the field, so nothing breaks.
- **Runtime deps in `dependencies`.** The backend lists `vitest`, `supertest`, `mongodb-memory-server`, and `nodemon` under `dependencies`, not `devDependencies`. Only `prettier` is a devDependency.
- **Format scope.** `npm run format` runs `prettier --write .` and rewrites any file it matches, including files you did not touch; existing backend files use mixed quote and indentation styles. Check `git diff` after running it.
- **AI provider edit is a source change.** Switching providers means editing a tracked file; do not commit a switch by accident.
- **Rate limiter in development.** The 100 requests per 15 minutes per-IP limit applies in development too, so heavy manual testing or scripts against the API can return 429.
- **Several files call `dotenv.config()`** (`main.js`, `authHandler.js`, `authService.js`, `threadService.js`, `aiProvider.js`, `populate_db.js`). The first call wins; they all read `.env` from the process working directory, so start the backend from `threadhive-backend`.
