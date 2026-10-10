# rkd-dockestra-core

<p align="center">
  <img src="https://img.shields.io/badge/platform-Django-092E20?style=for-the-badge&logo=django&logoColor=white" alt="Django">
  <img src="https://img.shields.io/badge/language-Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/language-Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white" alt="Bash">
  <img src="https://img.shields.io/badge/library-cryptography-3776AB?style=for-the-badge" alt="cryptography">
  <img src="https://img.shields.io/badge/library-WhiteNoise-555555?style=for-the-badge" alt="WhiteNoise">
  <img src="https://img.shields.io/badge/server-Gunicorn-499848?style=for-the-badge&logo=gunicorn&logoColor=white" alt="Gunicorn">
  <img src="https://img.shields.io/badge/database-SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white" alt="SQLite">
  <img src="https://img.shields.io/badge/tool-Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker">
  <img src="https://img.shields.io/badge/tool-Docker_Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker Compose">
  <img src="https://img.shields.io/badge/tool-Git-F05032?style=for-the-badge&logo=git&logoColor=white" alt="Git">
  <img src="https://img.shields.io/badge/tool-OpenSSL-721412?style=for-the-badge&logo=openssl&logoColor=white" alt="OpenSSL">
  <img src="https://img.shields.io/badge/service-GitHub_API-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub API">
  <img src="https://img.shields.io/badge/service-Cloudflare_Turnstile-F38020?style=for-the-badge&logo=cloudflare&logoColor=white" alt="Cloudflare Turnstile">
  <img src="https://img.shields.io/badge/service-Gmail_SMTP-EA4335?style=for-the-badge&logo=gmail&logoColor=white" alt="Gmail SMTP">
</p>

## 📖 About the project

**Dockestra Core** is the Django backend of the Dockestra platform (Django project `rkd_dockestra_core`, application `core`). It exposes a JSON API, restricted to staff operators, to manage a hierarchy of **Project → Environment → Image → Setup → Instance** records and to turn them into real Docker containers on the host's Docker Engine.

An *image* stores Dockerfile text, optionally linked to a GitHub repository and branch (public, or private with an encrypted token). A *setup* defines CPU, memory, an optional published port and an optional named volume. Creating an *instance* of a setup builds the image, starts a limited container named `<setup-code>-replica-<number>` and records it in SQLite. The project also ships the operator onboarding flow (the `RKD` superuser is created or recovered after a code sent to their email), Cloudflare Turnstile protection for the login, Docker/Compose files and deploy scripts.

The backend only serves JSON and the Django Admin. The web frontend and the Nginx reverse proxy live in a separate repository (`rkd-dockestra-web`) and are not part of this one.

<p align="center">
  <img src="docs/assets/dockestra-emblem.png" alt="Dockestra emblem" width="320">
</p>

## 🧰 Technologies used

| Technology | What it is used for |
| --- | --- |
| Django (6.0.x) | Web framework of the backend: models and migrations, session authentication, CSRF protection, Django Admin, the `createsuperuser` management command override and the JSON views under `/api/`. |
| Python (3.12+) | Language of the whole backend; the Docker image is based on `python:3.12-slim-bookworm`. |
| Bash | Deploy and maintenance scripts in `scripts/` and the container `docker/entrypoint.sh`. |
| cryptography | Fernet encryption of the GitHub tokens stored in image records, of the operator emails and of the SMTP sender email and app password, with keys derived from `DJANGO_SECRET_KEY`. |
| WhiteNoise | Serves the collected static files (for example the Django Admin assets) directly from the Gunicorn process, with compressed manifest storage. |
| Gunicorn | Production WSGI server started by `docker/entrypoint.sh` (`rkd_dockestra_core.wsgi:application`, 2 workers); not installed on Windows. |
| SQLite | Only database: stores the hierarchy, instances, operator accounts and encrypted credentials (`DJANGO_DB_PATH`, or `rkd_dockestra_core.sqlite3` in the project root by default). |
| Docker | Builds the backend image and, through the Docker CLI and the host socket, builds images and runs/removes the containers requested through the API; also queries the daemon for CPU and memory capacity. |
| Docker Compose | `docker-compose.yml` (production) and `docker-compose.local.yml` (local) run the `backend` service, its volumes, healthcheck and Docker networks. |
| Git | Shallow-clones the selected GitHub branch (`git clone --depth 1`) to use as the Docker build context for images linked to a repository. |
| OpenSSL | `scripts/configure-secret-key.sh` uses `openssl rand` to generate `DJANGO_SECRET_KEY`. |
| GitHub API | Reads repository branches and the About description (`api.github.com`) for the image form, using the stored token for private repositories. |
| Cloudflare Turnstile | Server-side validation (`siteverify`) of the CAPTCHA token required by the operator login. |
| Gmail SMTP | Sends the 16-character verification code used when creating or recovering operators (`EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_USE_TLS`, `EMAIL_USE_SSL`; defaults point to `smtp.gmail.com:587`). |

## ✨ Mapped features

Statuses were decided by running the whole test suite (117 tests, all passing; see *How to run the project*) and checking which tests cover each feature.

| Feature | What it does | Status |
| --- | --- | --- |
| Health response | `GET /` returns `{"name": "Dockestra Core", "status": "ready"}`. | COMPLETE |
| Operator session and login | `GET /api/auth/session/`, `POST /api/auth/login/` and `POST /api/auth/logout/` handle staff-only session login with CSRF protection; every other `/api/` route returns 401 without a session and 403 for non-staff users. | COMPLETE |
| Cloudflare Turnstile on login | Validates the challenge token (action `login`, allowed hostnames); mandatory when `DJANGO_DEBUG=False` (fails closed without keys) and optional in debug mode when no key is set. | COMPLETE |
| Project / Environment / Image / Setup CRUD | JSON `GET/POST/PUT/DELETE` endpoints for the hierarchy, with audit fields (`created_by`, `last_modified_by`, timestamps), JSON/CSRF validation, `GET /api/projects/<id>/setups/` and HTTP 409 when deleting records that still have children. | COMPLETE |
| Setup resource limits and validation | Validates CPU and memory against the Docker daemon (or host) capacity, accepts named volumes only and a port as `host:container` or `IP:host:container` (loopback by default); `GET /api/system/cpu/` and `GET /api/system/memory/` report the capacity. | COMPLETE |
| GitHub branches and description lookup | `GET/POST /api/github/branches/` and `/api/github/description/` list branches (up to 2,000) and read the About text for public or private repositories, accepting only `https://github.com/owner/repo` URLs and caching results for five minutes. | COMPLETE |
| Encrypted private repository token | Stores the image token encrypted, never returns it (only `hasToken`), keeps it when an update leaves it empty, decrypts legacy-prefixed tokens and keeps it out of Docker and the clone URL. | COMPLETE |
| Container instances (replicas) | `GET/POST /api/setups/<id>/instances/` builds the image (from the repository checkout or the backend root), starts a CPU/memory-limited container, publishes the port offset by the replica number, reuses the lowest free replica number, and `GET/DELETE /api/instances/<id>/` inspects or force-removes it; setups with instances cannot be edited or deleted. Docker is mocked in the tests, so no real container is started by them. | COMPLETE |
| RKD operator creation and recovery by email | Overridden `createsuperuser` command: the first user must be `RKD`; a 16-character code (5 minutes, 3 attempts) is emailed before the account is created or its password is recovered; the emails and the Gmail app password are stored encrypted in SQLite. | COMPLETE |
| Encrypted credential storage and migrations | `SmtpCredential` and `OperatorEmail` models, encrypted-email migration `0010` and legacy-key compatibility. | COMPLETE |
| Django Admin | Registers Project, Environment, Image (token hidden) and Setup (blocked from editing while instances exist). Only the Setup behaviour is covered by tests; the other registrations are untested. | PARTIAL |
| Local deploy and secret key scripts | `scripts/configure-secret-key.sh` (generate/preserve `DJANGO_SECRET_KEY`, `--force` database reset) and `scripts/refresh-local.sh` (build, isolated tests, migration check, migrate, start) are exercised by the tests with a fake Docker. | COMPLETE |
| Operator creation script | `scripts/create-superuser.sh` runs `createsuperuser --username RKD` inside the local container; no test covers this wrapper. | PARTIAL |
| Container image and production Compose | `Dockerfile`, `docker/entrypoint.sh` (migrations, legacy database copy, Docker socket group, Gunicorn) and `docker-compose.yml`/`docker-compose.local.yml` with healthchecks; there are no automated tests for them. | PARTIAL |

## 🚀 How to run the project

### Requirements

- **Python 3.12 or later** (the Docker image uses 3.12; Django 6.0 requires it) with `venv` and `pip`, to run Django directly.
- **Git CLI**, to clone this repository and to create containers from images linked to a GitHub repository.
- **Docker** with a reachable daemon (**Docker Compose** plugin included, Linux containers), to create containers and to use the deploy scripts. The backend container talks to the host daemon through `/var/run/docker.sock`.
- **Bash** and **OpenSSL**, for the scripts in `scripts/` (on Windows use Git Bash with OpenSSL, or WSL).
- A **Gmail account with an app password**, to create the first operator: the verification code is sent through Gmail SMTP.
- **Cloudflare Turnstile** site key and secret key, required for the login when `DJANGO_DEBUG=False` (not needed in local debug mode).

### Configuration files

`.env.example` is the template for `.env`, which is read by both Compose files (`env_file: .env`); `.env` is ignored by Git and must never be committed. Direct runs with `manage.py` do **not** load `.env`.

| Variable | What to edit and why |
| --- | --- |
| `DJANGO_SECRET_KEY` | **Required** in every environment (the app refuses to start without it). Do not write it by hand: `scripts/configure-secret-key.sh` generates it. Keep it stable and backed up separately: it also derives the keys that encrypt the stored GitHub tokens, emails and SMTP password. |
| `TURNSTILE_SITE_KEY`, `TURNSTILE_SECRET_KEY` | Fill with the keys of your Cloudflare Turnstile widget for production. `docker-compose.local.yml` blanks them to disable Turnstile in local mode. |
| `EMAIL_HOST`, `EMAIL_PORT`, `EMAIL_USE_TLS`, `EMAIL_USE_SSL` | SMTP settings, already set for Gmail (`smtp.gmail.com`, `587`, TLS on, SSL off). Change only to use another SMTP provider. The sender email and app password are **not** stored in `.env`: they are asked by the terminal when the operator is created and saved encrypted in SQLite. |

Other variables read by `rkd_dockestra_core/settings.py` and set by the Compose files: `DJANGO_DEBUG`, `DJANGO_ALLOWED_HOSTS`, `DJANGO_DB_PATH`, `DJANGO_TRUST_PROXY_HEADERS` and `TURNSTILE_ALLOWED_HOSTNAMES`.

For **production**, edit `docker-compose.yml`: `DJANGO_ALLOWED_HOSTS` and `TURNSTILE_ALLOWED_HOSTNAMES` are fixed to `sinan-pro.com` (and so is the `Host` header of the healthcheck); replace it with your own domain.

### Option 1: run with Docker (local mode)

```bash
git clone https://github.com/madrijkaard/rkd-dockestra-core.git
cd rkd-dockestra-core

# 1. Create .env from .env.example and generate DJANGO_SECRET_KEY (never overwrites an existing key)
bash scripts/configure-secret-key.sh

# 2. Build the image, run the tests in a disposable container, check migrations, migrate and start
bash scripts/refresh-local.sh

# 3. Create the RKD operator (asks for the email, the Gmail app password, the emailed code and the password)
bash scripts/create-superuser.sh
```

`refresh-local.sh` starts the container `rkd-dockestra-core-local-1` on the network `rkd-local-network` (alias `rkd-backend`) and persists SQLite in `volumes/sqlite/`. Port 8000 is **not published to the host** by the Compose files: the frontend/Nginx from `rkd-dockestra-web` is expected to reach it through the Docker network, and how to publish it for direct access is Not defined yet. `scripts/configure-secret-key.sh --force` generates a new key **and deletes every SQLite database of this checkout**; use `--help` for details.

### Option 2: run with Docker Compose (production)

Prerequisites: Docker Engine with the Compose plugin, Git and OpenSSL on the server, and the domain configured in `docker-compose.yml` (see above).

```bash
bash scripts/configure-secret-key.sh
nano .env                       # fill TURNSTILE_SITE_KEY and TURNSTILE_SECRET_KEY
docker compose up -d --build
docker compose ps
docker compose exec backend python manage.py createsuperuser --username RKD
```

The service `backend` (container `rkd-dockestra-core-1`) creates the network `rkd-network` (alias `rkd-backend`), runs `migrate` on start, and stores SQLite in `volumes/sqlite/rkd_dockestra_core.sqlite3`. The reverse proxy, HTTPS certificates and frontend are Not defined yet in this repository. Optional corporate CA certificates can be placed in `docker/certificates/` before the build.

### Option 3: run Django directly (development)

```bash
git clone https://github.com/madrijkaard/rkd-dockestra-core.git
cd rkd-dockestra-core
python3 -m venv .venv
source .venv/bin/activate                  # Windows Git Bash: source .venv/Scripts/activate
pip install -r requirements.txt
export DJANGO_SECRET_KEY='<a-long-random-stable-value>'
python manage.py migrate
python manage.py createsuperuser --username RKD
python manage.py runserver
```

The API answers at `http://127.0.0.1:8000/` (`DJANGO_DEBUG` defaults to `True`, so Turnstile is not required). Creating containers requires the Docker CLI and daemon on the same machine.

### Run the tests

```bash
export DJANGO_SECRET_KEY='<any-non-empty-value>'
python manage.py test
python manage.py makemigrations --check --dry-run
```

The deploy-script tests need `bash`. Docker, GitHub, Cloudflare and SMTP calls are mocked in the tests. Linters, formatters and CI/CD configuration are Not defined yet.
