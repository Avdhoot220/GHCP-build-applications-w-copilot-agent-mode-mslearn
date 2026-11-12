# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Keep route-level pages and UI state in React components, and put HTTP details in the shared API client. Use relative `/api` URLs so Vite's proxy remains the browser's API path. Each data page shows an accessible loading state, a useful empty state, and an accessible error alert.
- Logic tier decision: Keep request validation, business rules, and database operations in backend services; routes translate service results into the existing JSON response shapes and forward errors to the app's error handler. Return deliberate 4xx responses for invalid or unknown resources rather than leaking database cast or validation errors as 500s.
- Data tier decision: Enforce required fields in Mongoose schemas: user name, email, and fitness goal; team name and mascot, with members defaulting to an empty array; activity user, team, type, minutes, points, and occurrence time; and workout focus, minutes, and difficulty. Preserve the seed script's idempotence and unrelated records.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and positive finite `minutes`; validate both IDs and confirm both referenced records exist. Persist the references, type, and minutes, calculate one point per minute on the server (consistent with the existing seed data), and set `occurredAt` on the server.
- Team management approach: Require a non-empty name and mascot. Default omitted members to an empty list; when provided, validate member IDs against existing users and store them as user references. Return the submitted name and mascot and retain member IDs in team responses.
- Leaderboard approach: Sum persisted activity points per team, include teams with no activities at zero, order by total points descending, break ties by team name ascending, and assign ranks starting at 1.
- Workout suggestions approach: Find the user by email and return a deliberate 404 when no user matches. Prefer workouts matching the user's fitness goal; when there is no goal match, use their most recent activity pattern. Include a short reason tied to the matched goal or activity, and return a deliberate 4xx if the user has no matching suggestion.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` checks model validation and idempotent seeding. A failure indicates a persistence contract or seed regression. Baseline from `npm test --prefix octofit-tracker/backend`: 4 data-contract tests failed because required schema validators are missing; the seed test passed.
- API command: `npm run test:api --prefix octofit-tracker/backend` checks activity/team creation and validation, leaderboard aggregation/ranking, and user-specific workout suggestions. A failure indicates an API contract or service behavior regression. Baseline from `npm test --prefix octofit-tracker/backend`: all 4 API contract tests failed (501 responses for invalid creates, an empty leaderboard, and a missing workout-suggestion reason).
- Frontend command: `npm test --prefix octofit-tracker/frontend` checks rendered API data and the loading, empty, and error states for each page. Baseline: all 15 presentation contract tests failed, covering missing data rendering, the user-specific workout request, and the required accessible states.
- Backend build command: `npm run build --prefix octofit-tracker/backend` checks backend TypeScript types. Not run in the current baseline.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` checks frontend TypeScript and the Vite production build. Not run in the current baseline.
