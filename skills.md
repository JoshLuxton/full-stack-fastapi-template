# Full-Stack FastAPI Template — AI Assistant Skills Guide

This document describes the architecture, directory structure, naming conventions, and best practices for this template. Use it as a reference when generating, modifying, or reviewing code in this codebase.

---

## Stack Overview

| Layer | Technology |
|---|---|
| Backend framework | FastAPI 0.114+ |
| ORM / data models | SQLModel (Pydantic + SQLAlchemy) |
| Database | PostgreSQL 18 |
| Migrations | Alembic |
| Auth | JWT (PyJWT) + OAuth2PasswordBearer |
| Password hashing | pwdlib (Argon2 primary, bcrypt legacy) |
| Configuration | Pydantic Settings |
| Frontend framework | React 19 + TypeScript 5 |
| Router | TanStack Router (file-based) |
| Server state | TanStack Query |
| Forms | React Hook Form + Zod |
| UI components | shadcn/ui + Radix UI primitives |
| Styling | Tailwind CSS 4 |
| HTTP client | Auto-generated via @hey-api/openapi-ts (Axios) |
| Build tool | Vite 7 |
| E2E tests | Playwright |
| Code quality | Biome (frontend), Ruff + mypy (backend) |
| Container | Docker Compose + Traefik |

---

## Directory Structure

```
/
├── backend/
│   ├── app/
│   │   ├── main.py               # FastAPI app initialization, CORS, Sentry
│   │   ├── models.py             # All SQLModel models and Pydantic schemas
│   │   ├── crud.py               # All database operations
│   │   ├── utils.py              # Shared utilities (email sending, tokens)
│   │   ├── initial_data.py       # Seed first superuser on startup
│   │   ├── backend_pre_start.py  # Wait for DB before startup
│   │   ├── tests_pre_start.py    # Wait for DB before tests
│   │   ├── core/
│   │   │   ├── config.py         # Pydantic Settings (all env vars)
│   │   │   ├── db.py             # Engine creation, init_db()
│   │   │   └── security.py       # JWT, password hash/verify
│   │   ├── api/
│   │   │   ├── main.py           # Aggregates all routers
│   │   │   ├── deps.py           # FastAPI dependencies (session, auth)
│   │   │   └── routes/
│   │   │       ├── login.py      # /login/*, /password-recovery/*
│   │   │       ├── users.py      # /users/*
│   │   │       ├── items.py      # /items/*
│   │   │       ├── utils.py      # /utils/health-check, /utils/test-email
│   │   │       └── private.py    # Local-only dev endpoints
│   │   ├── alembic/
│   │   │   ├── env.py
│   │   │   └── versions/         # One .py file per migration
│   │   └── email-templates/
│   │       ├── src/              # MJML source files
│   │       └── build/            # Compiled HTML (committed)
│   ├── tests/
│   │   ├── conftest.py           # Fixtures: db, client, superuser tokens
│   │   ├── api/routes/           # One test file per route module
│   │   ├── crud/                 # CRUD-level tests
│   │   └── utils/                # Shared test helpers
│   ├── scripts/
│   │   ├── prestart.sh           # Run migrations + seed data
│   │   └── tests-start.sh        # Run pytest
│   ├── Dockerfile
│   ├── alembic.ini
│   └── pyproject.toml
│
├── frontend/
│   ├── src/
│   │   ├── main.tsx              # App entry: QueryClient, Router, themes
│   │   ├── client/               # AUTO-GENERATED — do not hand-edit
│   │   │   ├── sdk.gen.ts        # Service classes (UsersService, etc.)
│   │   │   ├── types.gen.ts      # All request/response types
│   │   │   └── schemas.gen.ts
│   │   ├── routes/
│   │   │   ├── __root.tsx        # Root layout (error boundary, devtools)
│   │   │   ├── _layout.tsx       # Protected layout (auth guard, sidebar)
│   │   │   ├── _layout/
│   │   │   │   ├── index.tsx     # Dashboard home
│   │   │   │   ├── admin.tsx     # User management (superuser only)
│   │   │   │   ├── items.tsx     # Items CRUD
│   │   │   │   └── settings.tsx  # User settings
│   │   │   ├── login.tsx
│   │   │   ├── signup.tsx
│   │   │   ├── recover-password.tsx
│   │   │   └── reset-password.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts        # Login, logout, signup, current user
│   │   │   ├── useCustomToast.ts
│   │   │   ├── useCopyToClipboard.ts
│   │   │   └── useMobile.ts
│   │   ├── components/
│   │   │   ├── ui/               # shadcn/ui primitives — do not hand-edit
│   │   │   ├── Common/           # App-wide shared components
│   │   │   ├── Admin/            # Components for the admin page
│   │   │   ├── Items/            # Components for the items page
│   │   │   ├── UserSettings/     # Components for the settings page
│   │   │   ├── Sidebar/          # Navigation sidebar
│   │   │   └── Pending/          # Suspense skeleton/loading states
│   │   ├── lib/
│   │   │   └── utils.ts          # cn() class merging utility
│   │   └── utils.ts              # handleError() and other app utilities
│   ├── tests/                    # Playwright E2E tests
│   ├── public/
│   ├── Dockerfile
│   ├── vite.config.ts
│   ├── playwright.config.ts
│   ├── biome.json
│   └── package.json
│
├── .env                          # Local defaults (never commit secrets)
├── compose.yml                   # Production Docker Compose
├── compose.override.yml          # Dev overrides (hot reload, ports)
├── compose.traefik.yml           # Traefik reverse proxy config
└── .github/workflows/            # CI: backend tests, playwright, deploy
```

---

## Backend Conventions

### Model and Schema Layout (`app/models.py`)

All SQLModel table models and Pydantic schema classes live in a single `models.py` file. Group related classes together with this pattern:

```python
# Base fields shared across variants
class ItemBase(SQLModel):
    title: str = Field(min_length=1, max_length=255)
    description: str | None = Field(default=None, max_length=255)

# DB table model
class Item(ItemBase, table=True):
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    created_at: datetime = Field(default_factory=get_datetime_utc)
    owner_id: uuid.UUID = Field(foreign_key="user.id", ondelete="CASCADE")
    owner: "User" = Relationship(back_populates="items")

# Request schemas (no id, no server-set fields)
class ItemCreate(ItemBase):
    pass

class ItemUpdate(SQLModel):
    title: str | None = Field(default=None, min_length=1, max_length=255)
    description: str | None = Field(default=None, max_length=255)

# Response schemas (what the API returns to clients)
class ItemPublic(ItemBase):
    id: uuid.UUID
    owner_id: uuid.UUID

class ItemsPublic(SQLModel):
    data: list[ItemPublic]
    count: int
```

**Rules:**
- Primary keys are always `uuid.UUID` generated with `uuid.uuid4`, never integers.
- Timestamps are `datetime` using `get_datetime_utc()` (timezone-aware UTC).
- `max_length` must be set on all `str` fields that map to a database column.
- Public response models never expose `hashed_password` or other internal fields.
- Paginated list responses always use a wrapper class with `data: list[T]` and `count: int`.

### CRUD Layer (`app/crud.py`)

All database operations live in `crud.py`. No SQL or ORM queries belong in route handlers.

```python
def create_item(*, session: Session, item_in: ItemCreate, owner_id: uuid.UUID) -> Item:
    db_item = Item.model_validate(item_in, update={"owner_id": owner_id})
    session.add(db_item)
    session.commit()
    session.refresh(db_item)
    return db_item
```

**Rules:**
- All functions take `session: Session` as a keyword-only argument.
- Use `Model.model_validate(schema, update={...})` to merge in server-set fields.
- Always `commit()` then `refresh()` before returning a created or updated object.

### Route Handlers (`app/api/routes/`)

One file per resource. The file registers a single `APIRouter` with a `prefix` and `tags`.

```python
router = APIRouter(prefix="/items", tags=["items"])

@router.get("/", response_model=ItemsPublic)
def read_items(session: SessionDep, current_user: CurrentUser, skip: int = 0, limit: int = 100) -> Any:
    ...
```

**Rules:**
- Use `response_model=` on every endpoint.
- Pagination uses `skip: int = 0, limit: int = 100` query parameters.
- Permission checks go at the top of the function body (raise `HTTPException(403)` or check deps).
- Superuser-only actions use `get_current_active_superuser` dependency.
- Never expose whether an email exists in error messages (security: safe enumeration).

### Dependencies (`app/api/deps.py`)

Use `Annotated` types to create reusable, self-documenting dependencies:

```python
SessionDep = Annotated[Session, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

Add new resources by adding new `Annotated` dependency types here; never inline `Depends(...)` directly in function signatures.

### Configuration (`app/core/config.py`)

All environment variables are accessed through the `settings` singleton (Pydantic `BaseSettings`). Never use `os.environ` directly.

```python
from app.core.config import settings
settings.POSTGRES_SERVER
settings.SECRET_KEY
```

- Add new env vars as fields on the `Settings` class with sensible defaults.
- Use `@field_validator` or `@computed_field` for derived values.
- Non-local environments enforce non-default secrets via validators.

### New Resource Checklist (Backend)

When adding a new resource (e.g., `Project`):

1. Add `ProjectBase`, `Project` (table), `ProjectCreate`, `ProjectUpdate`, `ProjectPublic`, `ProjectsPublic` to `models.py`.
2. Add `create_project`, `get_project`, `update_project`, `delete_project` to `crud.py`.
3. Create `app/api/routes/projects.py` with router prefix `/projects`.
4. Register the router in `app/api/main.py`.
5. Create a migration: `alembic revision --autogenerate -m "Add project table"`.
6. Add `tests/api/routes/test_projects.py` and `tests/utils/project.py`.

---

## Frontend Conventions

### Route Files (`src/routes/`)

TanStack Router uses file-based routing. Key conventions:

| File pattern | Purpose |
|---|---|
| `__root.tsx` | Root layout; wraps everything |
| `_layout.tsx` | Authenticated layout; contains `beforeLoad` auth guard |
| `_layout/foo.tsx` | Protected page at `/foo` |
| `foo.tsx` | Public page at `/foo` |

**Auth guard pattern** (used in `_layout.tsx`):
```tsx
export const Route = createFileRoute('/_layout')({
  beforeLoad: ({ location }) => {
    if (!isLoggedIn()) {
      throw redirect({ to: '/login', search: { redirect: location.href } })
    }
  },
})
```

**Superuser guard pattern** (used in `_layout/admin.tsx`):
```tsx
beforeLoad: ({ context }) => {
  if (!context.queryClient.getQueryData(['currentUser'])?.is_superuser) {
    throw redirect({ to: '/' })
  }
}
```

### Components (`src/components/`)

| Folder | Contents |
|---|---|
| `ui/` | Auto-added by shadcn/ui — do not edit manually |
| `Common/` | Shared across multiple features |
| `Admin/` | Only used by the admin route |
| `Items/` | Only used by the items route |
| `UserSettings/` | Only used by the settings route |
| `Sidebar/` | Navigation components |
| `Pending/` | Suspense skeleton loaders |

Add a new feature folder (e.g., `Projects/`) when a feature has more than one component. Otherwise, put single-component features in `Common/`.

For each resource that needs CRUD UI, create these files:
- `columns.tsx` — TanStack Table column definitions
- `AddProject.tsx` — dialog/form for creation
- `EditProject.tsx` — dialog/form for editing
- `DeleteProject.tsx` — confirmation dialog for deletion
- `ProjectActionsMenu.tsx` — row action dropdown

### API Client (`src/client/`)

**This entire directory is auto-generated. Never edit it manually.**

Regenerate after any backend model or endpoint change:
```bash
# From repo root (requires running backend)
bun run generate-client
```

Import generated services like:
```tsx
import { ItemsService, type ItemPublic } from '@/client'
```

### Data Fetching Pattern

Use `useSuspenseQuery` for page-level data (pairs with `<Suspense>` boundary):

```tsx
const { data } = useSuspenseQuery({
  queryKey: ['items'],
  queryFn: () => ItemsService.readItems({ skip: 0, limit: 100 }),
})
```

Use `useMutation` for writes, with query invalidation on success:

```tsx
const mutation = useMutation({
  mutationFn: (data: ItemCreate) => ItemsService.createItem({ requestBody: data }),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['items'] })
    showToast('Item created')
  },
  onError: handleError,
})
```

### Form Pattern

```tsx
const form = useForm<ItemCreate>({
  resolver: zodResolver(itemCreateSchema),
  defaultValues: { title: '', description: '' },
})

// In JSX:
<Form {...form}>
  <FormField control={form.control} name="title" render={({ field }) => (
    <FormItem>
      <FormLabel>Title</FormLabel>
      <FormControl><Input {...field} /></FormControl>
      <FormMessage />
    </FormItem>
  )} />
  <LoadingButton type="submit" loading={mutation.isPending}>Save</LoadingButton>
</Form>
```

Define Zod schemas adjacent to the form component. Mirror backend field constraints (min_length, max_length).

### Naming Conventions (Frontend)

| Thing | Convention | Example |
|---|---|---|
| Component files | PascalCase `.tsx` | `AddItem.tsx` |
| Hook files | camelCase starting with `use` | `useAuth.ts` |
| Route files | lowercase, hyphens | `recover-password.tsx` |
| Utility files | camelCase | `utils.ts` |
| Generated service classes | PascalCase + `Service` | `ItemsService` |
| Generated types | PascalCase | `ItemPublic`, `ItemCreate` |
| Query keys | lowercase string arrays | `['items']`, `['currentUser']` |
| CSS classes | Tailwind utilities via `cn()` | `cn('flex gap-2', isActive && 'font-bold')` |

### New Resource Checklist (Frontend)

When adding a new resource (e.g., `Project`) after regenerating the client:

1. Create `src/components/Projects/` with `columns.tsx`, `AddProject.tsx`, `EditProject.tsx`, `DeleteProject.tsx`, `ProjectActionsMenu.tsx`.
2. Create `src/routes/_layout/projects.tsx` with the page route.
3. Add the route to the sidebar in `src/components/Sidebar/Main.tsx`.
4. Regenerate the client if backend models changed.

---

## Environment Variables

### Backend (`.env` / `app/core/config.py`)

| Variable | Purpose |
|---|---|
| `SECRET_KEY` | JWT signing key — must be changed in production |
| `ENVIRONMENT` | `local` \| `staging` \| `production` |
| `BACKEND_CORS_ORIGINS` | Comma-separated allowed origins |
| `FIRST_SUPERUSER` | Email of the bootstrap superuser |
| `FIRST_SUPERUSER_PASSWORD` | Password of the bootstrap superuser |
| `POSTGRES_SERVER` | DB host |
| `POSTGRES_DB` | Database name |
| `POSTGRES_USER` | DB user |
| `POSTGRES_PASSWORD` | DB password |
| `SMTP_HOST` | Mail server host (optional) |
| `SENTRY_DSN` | Sentry error tracking DSN (optional) |

### Frontend (`.env` / `VITE_*`)

| Variable | Purpose |
|---|---|
| `VITE_API_URL` | Backend base URL, injected into the OpenAPI client at startup |

---

## Database Migrations

Always create a migration after changing `models.py`:

```bash
# Generate migration
docker compose exec backend alembic revision --autogenerate -m "Short description"

# Apply to DB
docker compose exec backend alembic upgrade head

# Rollback one step
docker compose exec backend alembic downgrade -1
```

**Rules:**
- Review auto-generated migration files before committing — Alembic misses some changes (e.g., column renames).
- Never edit an applied migration. Create a new one instead.
- Migrations run automatically at container startup via `scripts/prestart.sh`.

---

## Testing

### Backend

```bash
# Run all tests inside Docker
docker compose exec backend bash scripts/tests-start.sh

# Run specific file
docker compose exec backend pytest tests/api/routes/test_items.py -v
```

- `conftest.py` provides: `db` (session-scoped), `client` (module-scoped TestClient), `superuser_token_headers`, `normal_user_token_headers`.
- Each test file imports helpers from `tests/utils/`.
- 90% coverage required (enforced by CI).

### Frontend (E2E)

```bash
# Run Playwright tests
bun run test
```

E2E tests live in `frontend/tests/`. They test full user flows against a running stack.

---

## Security Practices

- **Never** store plain-text passwords. Always use `get_password_hash()` / `verify_password()` from `app.core.security`.
- Use `DUMMY_HASH` in `crud.py:authenticate()` to prevent timing attacks when a user is not found.
- JWT secrets and DB passwords must be changed from defaults before any non-local deployment. Config validators enforce this.
- Do not reveal whether an email address exists in password-recovery or login error responses.
- Superuser actions must be protected by the `get_current_active_superuser` dependency.
- Cascade deletes are configured at the database level (`ondelete="CASCADE"`) and enforced in Alembic migrations.

---

## Docker Services

| Service | Purpose | Internal port |
|---|---|---|
| `db` | PostgreSQL 18 | 5432 |
| `adminer` | DB admin UI | — (Traefik) |
| `prestart` | Runs migrations + seed | — (job) |
| `backend` | FastAPI app | 8000 |
| `frontend` | React SPA (Nginx) | 80 |

Dev overrides (`compose.override.yml`) mount source directories for hot reload and expose ports directly.

---

## Code Style

### Backend
- **Formatter/linter**: Ruff (configured in `pyproject.toml`)
- **Type checker**: mypy strict mode
- Imports: stdlib → third-party → local, separated by blank lines
- All public functions must have type annotations

### Frontend
- **Formatter/linter**: Biome (configured in `biome.json`)
- No ESLint or Prettier — use Biome exclusively
- TypeScript strict mode enabled
- Prefer `const` over `let`; avoid `any`
