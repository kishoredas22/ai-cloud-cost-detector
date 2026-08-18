# Prompt 4: React Frontend + Custom JWT Auth

Build a React frontend and add custom JWT authentication to the existing FastAPI backend.

## What to build

### Backend auth endpoints

Add two endpoints to the FastAPI backend:

- `POST /api/auth/signup` — accepts email and password, hashes the password with bcrypt, stores the user in the `users` table, returns a JWT.
- `POST /api/auth/login` — validates credentials, returns a JWT.

Store `JWT_SECRET` in the environment and add it to `.env.example`. Update `backend/requirements.txt` with `PyJWT` and `bcrypt`.

Use a sensible token expiry and include the user ID in the claims — `GET /api/history` in prompt 03 scopes results by user.

### Frontend

Vite + React + TypeScript, styled with Tailwind CSS in a dark theme.

Auth handling:

- Store the JWT in `localStorage` after login or signup.
- Attach it to every API request as `Authorization: Bearer <token>`.
- Redirect unauthenticated users to the login page.
- Clear the token and redirect on a `401`, so an expired token doesn't leave the app in a broken half-authenticated state.

### Pages

**Login / Signup** — email and password forms with inline validation and error display.

**Dashboard** — a dropdown of **GCP projects** (from `GET /api/projects`), a button to run the analysis, and a live status area. This is the only page that differs structurally from the Azure original: the user picks a project, not a resource group.

Show `projectId` as the value and `name (projectId)` as the label — display names are not unique across projects and the ID is what the scan actually uses.

**Analysis Report** — renders the AI analysis: summary, issues with colour-coded severity badges, estimated savings, and copyable code blocks for the `gcloud` fix commands.

Distinguish the two savings sources visually. Findings with `savings_source: "recommender"` carry a real figure from Google and should read as measured; findings marked `"estimated"` are the model's own inference and should be visibly labelled as estimates. Presenting both with the same confidence would misrepresent the data — and a cost tool that overstates its certainty gets ignored the first time someone checks a number.

**History** — a list of past analyses showing project, date, issues found, and savings. Clicking an entry opens the full report.

### Components

- `ProgressTracker` — an animated step list, wired to the WebSocket in prompt 05.
- `Navbar` — navigation and logout.

## Project structure

```
frontend/
├── index.html
├── package.json
├── tailwind.config.js
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── pages/
│   │   ├── Login.tsx
│   │   ├── Signup.tsx
│   │   ├── Dashboard.tsx
│   │   ├── Report.tsx
│   │   └── History.tsx
│   └── components/
│       ├── ProgressTracker.tsx
│       └── Navbar.tsx
```

```
backend/
├── main.py            (updated — auth endpoints, JWT issuing)
├── auth.py            (new — password hashing, token creation and verification)
├── requirements.txt   (updated — add PyJWT, bcrypt)
├── .env.example       (updated — add JWT_SECRET)
```

Refer to `Architecture.MD` and `RequestFlow.MD`. This covers step ① of the request flow.
