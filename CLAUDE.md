# CLAUDE.md

Guidance for Claude Code and other AI assistants working in this repository.

## Project overview

**Aqua Trajectory AI** is a two-service application:

- **backend** — Python 3.10 / FastAPI, served by `uvicorn` on port **8000**
- **frontend** — Node.js 18, built and served by npm scripts on port **3000**

The two services are orchestrated with Docker Compose. There is no database, no
message broker, and no other infrastructure at present.

## Current state: scaffolding only — read this first

**The repository contains infrastructure files only. No application source code
exists yet**, in any branch or any commit in history. Every tracked file is
listed below:

```
aqua-trajectory-ai/
├── .gitignore
├── README.md
├── docker-compose.yml
├── backend/
│   └── Dockerfile
└── frontend/
    └── Dockerfile
```

The `README.md` "Project Structure" section describes files that do **not**
exist (`backend/main.py`, `backend/requirements.txt`, `frontend/package.json`,
`frontend/server.js`). Treat that section as the *intended* target layout, not a
description of what is on disk.

**Consequence: `docker-compose up --build` currently fails.** Both Dockerfiles
reference files that are absent — the backend `COPY requirements.txt .` has
nothing to copy, and the frontend `RUN npm install` has no `package.json`. Do
not report the stack as runnable until the files in the next section exist.

Do not silently invent these files as a side effect of an unrelated task, and do
not "fix" the README by deleting the structure section — it is the spec. When
asked to make the stack run, create the missing files to the contracts below.

## Contracts the missing files must satisfy

These are hard requirements dictated by the existing Dockerfiles and
`docker-compose.yml`. Anything you add must honour them, or the container build
and start commands break.

### `backend/requirements.txt`
- Must exist at `backend/` root; installed with plain `pip install -r
  requirements.txt` (no lockfile tooling, no virtualenv inside the image).
- Must include `fastapi` and `uvicorn`, since the image's `CMD` invokes uvicorn.

### `backend/main.py`
- Must expose a module-level ASGI app named **`app`** — the container runs
  `uvicorn main:app --host 0.0.0.0 --port 8000`. Renaming the module or the
  variable requires editing `backend/Dockerfile`.
- Must bind to `0.0.0.0`, never `127.0.0.1`, or the port publish will not reach
  the service from the host.
- Must serve **`GET /api/health`** — the README documents this as the backend
  health check endpoint.

### `frontend/package.json`
- Must define both a **`build`** and a **`start`** script. The image runs
  `npm run build` at build time and `npm start` as its `CMD`; a missing `build`
  script fails the image build outright.
- `npm install` is used (not `npm ci`), so a lockfile is optional but
  recommended — commit `package-lock.json` if one is generated.

### `frontend/server.js`
- Must listen on port **3000** and bind to `0.0.0.0` for the same reason as the
  backend.
- Must serve **`GET /api/health`**.
- Reaches the backend at **`http://backend:8000`** — `backend` is the Compose
  service name and therefore its DNS hostname on the Compose network. Never
  hardcode `localhost:8000` for server-to-server calls; inside the frontend
  container `localhost` is the frontend itself. Browser-side code, which runs on
  the user's machine, does use `http://localhost:8000`.

## Development workflow

Docker Compose is the only supported workflow — there are no Makefiles, task
runners, or setup scripts.

```bash
docker-compose up --build      # build images and start both services
docker-compose up --build backend   # rebuild/start a single service
docker-compose logs -f backend      # follow one service's logs
docker-compose down            # stop and remove containers
```

Endpoints once running:

| URL | Purpose |
| --- | --- |
| http://localhost:3000 | Frontend |
| http://localhost:8000 | Backend API |
| http://localhost:3000/api/health | Frontend health check |
| http://localhost:8000/api/health | Backend health check |

There is **no hot reload configured**. `docker-compose.yml` defines no bind
mounts, so source edits require a rebuild (`docker-compose up --build`) to take
effect. If you add a mount or a `--reload` flag for local iteration, say so
explicitly rather than assuming it is expected.

### Running outside Docker

Nothing prevents running the services directly, but no tooling is checked in for
it. `backend/venv/` and `frontend/node_modules/` are already gitignored, so the
conventional local setup is a venv under `backend/` and a plain `npm install`
under `frontend/`.

## Testing, linting, CI

**None exist.** There is no test suite, no linter or formatter config, no
pre-commit hooks, and no `.github/` directory or CI workflows. Do not claim
tests pass — there are none to run. When adding code, verify it by building and
exercising the health endpoints:

```bash
docker-compose up --build -d
curl -f http://localhost:8000/api/health
curl -f http://localhost:3000/api/health
docker-compose down
```

If you introduce a test framework or CI, update this file and the README in the
same change.

## Conventions

- **Health endpoints** live under `/api/health` on both services. Keep new API
  routes under the `/api/` prefix for consistency.
- **Pinned base images**: `python:3.10` and `node:18`. Match these versions in
  any local tooling or CI you add so behaviour is consistent.
- **Ports are fixed** in three places each — the Dockerfile `CMD`, the app's own
  bind, and the `docker-compose.yml` publish. Changing a port means changing all
  three plus the README table.
- **Secrets** go in `backend/.env` and `frontend/.env`, both gitignored. Never
  commit either, and never hardcode credentials. Note that `docker-compose.yml`
  currently declares no `env_file` or `environment` keys — wire those up if you
  add env-based config, or the containers will not see the values.
- **Keep the three docs in sync**: when you add, remove, or move a file at the
  top of a service directory, update the README's Project Structure block and
  the relevant section here.

## Known rough edges

Flag or fix these when touching the surrounding files; they are not accidental
omissions you should assume someone is handling.

- **No `.dockerignore` in either service.** `frontend/Dockerfile` runs
  `COPY . .` *before* `npm install`, so a local `node_modules/` and any `.env`
  file get copied into the image. Adding `.dockerignore` files is worthwhile.
- **Frontend layer caching is ineffective.** Copying all sources before
  `npm install` invalidates the dependency layer on every source edit. The
  backend Dockerfile already does this correctly (`COPY requirements.txt .`
  first) — mirror that pattern in the frontend if you touch it.
- **`docker-compose.yml` uses `version: "3"`**, an obsolete key that current
  Docker Compose warns about. Harmless, removable.
- **`depends_on` without a healthcheck** only orders container *start*, not
  readiness. The frontend may come up before the backend can serve requests, so
  frontend code calling the backend at boot needs to tolerate that.

## Git workflow

- Default branch is `main`. Feature work happens on branches; the current branch
  is `claude/claude-md-docs-6tu4ms`.
- Commit messages in history are short and imperative ("Add Dockerfile for
  backend setup", "Create README.md file"). Match that style.
- No branch protection or required checks are configured in the repo.
