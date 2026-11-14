# Issues

Verified against the code. See [DEVELOPMENT.md](DEVELOPMENT.md#known-issues-and-gotchas) for background.

1. **Port mismatch.** Server defaults to 3000; the frontend hard-codes 5000.
   `threadhive-backend/server.js`, `threadhive-frontend/src/api/axiosInstance.js`
   Fix: default the server to 5000, or read the frontend base URL from `import.meta.env.VITE_API_URL`.

2. **Dead endpoints.** `USER_API` and `SUBREDDIT_API.GET_WITH_THREADS` have no backend route and no callers.
   `threadhive-frontend/src/config/apiConfig.js`
   Fix: delete both.

3. **Missing thread endpoints.** Backend has `PUT` and `DELETE /api/threads/:id`; `THREAD_API` has no entries for them.
   `threadhive-frontend/src/config/apiConfig.js`, `threadhive-backend/src/routes/threads.js`
   Fix: add `UPDATE` and `DELETE` entries (`(id) => \`/threads/${id}\``).

4. **Unused `stopApp`.** Never called and not wired to `SIGINT`/`SIGTERM`.
   `threadhive-backend/main.js`
   Fix: register it on both signals, or remove it.

5. **`stopServer` does not await `server.close`.** The callback is ignored, so the promise resolves early.
   `threadhive-backend/server.js`
   Fix: wrap `server.close` in a `new Promise` that resolves in its callback.

6. **Stale `"main": "app.js"`.** No such file at the backend root.
   `threadhive-backend/package.json`
   Fix: set it to `"main.js"` or remove the field.

7. **Test and dev tooling in `dependencies`.** `vitest`, `supertest`, `mongodb-memory-server`, `nodemon`.
   `threadhive-backend/package.json`
   Fix: `npm install -D vitest supertest mongodb-memory-server nodemon`.

8. **`npm run format` rewrites everything with no Prettier config.**
   `threadhive-backend/package.json`
   Fix: add a `.prettierrc` and a `.prettierignore`, or scope the script to `src tests`.

9. **Rate limit applies in development.** 100 requests per 15 minutes per IP, always on.
   `threadhive-backend/src/app.js`
   Fix: skip or raise the limit when `NODE_ENV === "development"`.

10. **`dotenv.config()` called in many files.** `main.js`, `src/middleware/authHandler.js`, `src/services/authService.js`, `src/services/threadService.js`, `src/utils/aiProvider.js`, `src/scripts/populate_db.js`.
    Fix: load once at the entry point (`main.js`, `populate_db.js`) and remove the rest.

11. **`NODE_ENV` set by no script.** `errorHandler.js` only adds stack traces when it equals `development`, so `npm run dev` never does.
    `threadhive-backend/package.json`, `threadhive-backend/src/middleware/errorHandler.js`
    Fix: set it in the `dev` script (for example `NODE_ENV=development nodemon main.js`).
