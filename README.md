# MealPlanner

A full-stack web app for planning weekly meals and generating a combined shopping list. Built a .NET 10 controller-based REST API with EF Core and PostgreSQL, and a Next.js frontend in React and TypeScript that uses static rendering, SSR, ISR and CSR according to each page’s needs. Separated server state (TanStack Query) from client state (Zustand), and deployed the full stack locally with Docker Compose.

## Run it

```bash
docker compose up --build
```

| What | URL |
|---|---|
| Web app | http://localhost:3000 |
| API docs (Swagger UI) | http://localhost:5000/swagger |
| OpenAPI JSON | http://localhost:5000/openapi/v1.json |
| PostgreSQL | localhost:5432 (user/password/db: `mealplanner`) |

The API creates the schema and seeds ten recipes on first start.
To reset the database: `docker compose down -v`.

### Running pieces outside Docker

```bash
docker compose up db                  # just Postgres
cd api && dotnet run                  # API on http://localhost:5000
cd web && cp .env.local.example .env.local && npm install && npm run dev
```

`npm run dev` re-renders every request, so ISR caching is only visible in
the Docker build or with `npm run build && npm start`.

## How the pieces fit
<p align="center">
<img width="544" height="288" alt="mealplanner_architecture" src="https://github.com/user-attachments/assets/0470e059-576a-4790-b5bd-3651e905bd9f" />
</p>

| Route | Rendering | File |
|---|---|---|
| `/` | Static | `web/src/app/page.tsx` |
| `/recipes` | SSR + TanStack hydration | `web/src/app/recipes/page.tsx` |
| `/recipes/[slug]` | ISR (60 s + on demand) | `web/src/app/recipes/[slug]/page.tsx` |
| `/planner` | CSR (`ssr: false`) | `web/src/components/planner/*` |
| `/shopping-list` | Static shell + CSR data | `web/src/components/ShoppingList.tsx` |

## Where the state lives

| State | Owner | Why |
|---|---|---|
| Recipes, tags, saved plan, shopping list | TanStack Query | Has a copy on the server: needs caching, refetching, invalidation |
| Search text and tag filter | URL | Shareable, survives refresh, readable by the server for SSR |
| The plan you're editing, servings, unsaved flag | Zustand (`plannerStore`, persisted) | Exists only in the browser until you press Save |
| Which day "Add" targets | Zustand (not persisted) | Pure UI state |
| Ticked shopping items | Zustand (`shoppingStore`, persisted) | Never needs to reach the server |

The rule to remember: **Zustand stores IDs and intent; TanStack Query stores
what the server knows.** The planner looks up recipe titles from the query
cache by ID rather than copying them into the store.

## Things to try

1. **Server-to-client handoff.** Load `/recipes?tag=vegetarian` and open the
   Network tab: no `/api/recipes` request from the browser on first load.
   Click another tag: now there is one. Open React Query Devtools (bottom
   left) to see the cache keys.
2. **ISR.** Open a recipe page and note the "Page built" time at the bottom.
   Refresh: same time. Wait 60 s and refresh twice: the first refresh shows
   the old page and triggers a rebuild; the second shows the new time.
3. **On-demand ISR.** In Swagger, call `PUT /api/recipes/{slug}` with a new
   title. Refresh the recipe page: it changes immediately, because the API
   called `/api/revalidate`.
4. **Persisted client state.** Add meals to the plan, then close the tab and
   come back. The draft (and the yellow "unsaved" badge) is still there.
5. **Optimistic update and rollback.** Run `docker compose stop api`, press
   Save plan, and watch the cache roll back while your draft stays intact.
   Start the API again and save.
6. **Fine-grained re-renders.** Turn on "Highlight updates" in React
   DevTools and change servings on one day: only that row re-renders.
7. **Hydration.** In `plannerStore.ts`, remove `skipHydration: true` and the
   `rehydrate()` calls in `providers.tsx`. With meals in the plan, reload
   `/recipes` and look for the hydration-mismatch warning in the console.

## Next steps

* **Migrations.** The API uses `EnsureCreated` until migrations exist:
  ```bash
  cd api
  dotnet tool install --global dotnet-ef
  dotnet ef migrations add InitialCreate
  docker compose down -v && docker compose up --build   # switches to Migrate()
  ```
* **Drag and drop** in the planner with `@dnd-kit/core`, calling `moveMeal`.
* **Redux Toolkit comparison.** Rewrite `plannerStore.ts` as a slice
  (`createSlice`, `configureStore`, `redux-persist`) and compare the two.
* **Auth.** Replace `CurrentUser.Id` with a JWT claim; give each user a plan.
* **Tests.** Vitest for store actions (they're plain functions:
  `usePlannerStore.getState().addMeal(...)`), xUnit + Testcontainers for the API.
* **Generated types.** Generate `web/src/lib/types.ts` from
  `/openapi/v1.json` with `openapi-typescript`.
