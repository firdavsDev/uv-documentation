# UV migration cheat-sheet

Clean, step-by-step guide you can use. Shows how to add **uv** to an existing project (replace `python -m venv venv` + `requirements.txt`), daily uv commands, lock behavior, **and** a Dockerfile example.

---

## 1) Quick goal

Migrate old workflow:

```
python -m venv venv
pip install -r requirements.txt
```

to uv workflow:

```
uv venv
pyproject.toml + uv.lock
uv sync / uv run ...
```

---

## 2) Prep: remove old venv (optional but clear)

```bash
# from project root
rm -rf venv
```

---

## 3) Install uv (if not installed)

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
irm https://astral.sh/uv/install.ps1 | iex
```

---

## 4) Create uv virtual env and activate

```bash
# create a .venv managed by uv
uv venv

# activate (if you want to use shell activate)
source .venv/bin/activate    # macOS / Linux
# or on Windows PowerShell:
# .venv\Scripts\Activate.ps1
```

You can still use `uv run <command>` without activating.

---

## 5) Init pyproject.toml (if none)

```bash
uv init
```

This creates a minimal `pyproject.toml` with `dependencies = []`.

---

## 6) Import existing requirements.txt into uv

Preferred:

```bash
uv add --requirements requirements.txt
```

If your uv version doesn't support that, alternative:

```bash
# install into uv-managed venv
uv pip install -r requirements.txt
# then create lock
uv lock
```

Result: `pyproject.toml` gets dependencies, and `uv.lock` is created/updated. Packages are installed into `.venv`.

---

## 7) Ensure environment is in sync

```bash
# make .venv match pyproject.toml + uv.lock exactly
uv sync
```

---

## 8) Daily commands — most useful (copy this part)

```bash
# add package (prod)
uv add <package>
# add package as dev
uv add --dev <package>

# remove package
uv remove <package>

# update / create lock file
uv lock
uv lock --upgrade   # attempt upgrades

# install packages exactly from lock (ensures deterministic env)
uv sync

# run any command inside uv venv
uv run <command>
# ex:
uv run python manage.py migrate
uv run pytest

# if you need to use pip directly inside uv venv
uv pip install <pkg>
uv pip install -r requirements.txt
```

---

## 9) Lock file behavior — TL;DR (yes, you must update)

* `uv.lock` is the deterministic lock (exact versions & hashes).
* If you **use** `uv add` or `uv remove`, `uv` will update `uv.lock` automatically.
* If you **manually** edit `pyproject.toml`, run:

  ```bash
  uv lock
  ```

  then:

  ```bash
  uv sync
  ```
* Best practice: always commit both `pyproject.toml` and `uv.lock` to git. Team members run `uv sync` to reproduce exact env.

**Short table**

* add/remove via `uv` → lock auto-updated ✅
* manual edits to pyproject → run `uv lock` 🔁
* after lock change → run `uv sync` to apply to `.venv` ✅

---

## 10) Example full migration (pasteable)

```bash
# remove old venv (optional)
rm -rf venv

# install uv (if needed), create venv
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv
source .venv/bin/activate

# init project metadata (if not exist)
uv init

# import old requirements (preferred)
uv add --requirements requirements.txt

# OR fallback
# uv pip install -r requirements.txt
# uv lock

# sync environment and run
uv sync
uv run python manage.py runserver
```

---

## 11) How to build Docker image reliably (recommended pattern)

In Docker you usually install dependencies globally (no venv inside container). To ensure the container uses the same frozen versions, export a frozen `requirements.txt` from your uv venv and use that in the Docker build.

### a) Export frozen requirements locally (from your uv env)

```bash
# this produces a fully pinned requirements.txt that Docker can install
uv run pip freeze > docker-requirements.txt
```

### b) Dockerfile (simple, reproducible)

```dockerfile
# Dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# system deps (if needed)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc \
  && rm -rf /var/lib/apt/lists/*

# copy only pip freeze file to leverage cache
COPY docker-requirements.txt .

# install pinned dependencies
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r docker-requirements.txt

# copy project
COPY . .

# expose port (if Django)
EXPOSE 8000

# example run (Django)
CMD ["gunicorn", "my_project.wsgi:application", "--bind", "0.0.0.0:8000"]
```

**Notes**

* Generating `docker-requirements.txt` from your uv venv (`uv run pip freeze`) pins exact packages and makes Docker builds reproducible.
* Alternatively, you can `COPY pyproject.toml uv.lock` into the image and run a build step that installs from them, but the most portable & simple is `pip install -r docker-requirements.txt` created from your pinned env.

---

## 12) Extra tips & gotchas

* Always commit `pyproject.toml` + `uv.lock`. Don’t commit `.venv/`.
* Use `uv run` in CI to be explicit: `uv run pytest` is clearer than relying on an activate step.
* If something breaks after adding/removing packages: `uv lock` then `uv sync` usually fixes it.
* For platform-specific wheels (mac vs linux), `uv.lock` may have markers — CI/docker must match platform for perfect reproducibility. If building in Docker, freeze on the same platform where you build the image (usually Linux).

---

## Minimal snapshot you can use

```
# create uv venv
uv venv
source .venv/bin/activate

# migrate deps
uv init
uv add --requirements requirements.txt
uv sync

# daily
uv add <pkg>
uv remove <pkg>
uv lock
uv sync
uv run <command>

# docker workflow (freeze then build)
uv run pip freeze > docker-requirements.txt
docker build -t myapp:latest .
```

