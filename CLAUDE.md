# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the codebase for the **Application Modernization in GCP** workshop. It is an intentionally legacy PHP 5.6 image catalog application backed by MySQL, used as a starting point for cloud modernization exercises. **It is not production-safe** — it contains known security issues by design.

## Architecture

The project has two main components:

### 1. PHP Web Application

A flat-file PHP 5.6 app (no framework, no Composer) served via Apache inside Docker. All pages are top-level `.php` files:

- `config.php` — establishes the PDO connection and starts the session; included by every other page
- `index.php` — image catalog (role-aware: admins see all images including flagged ones)
- `upload.php` — handles file upload, writes to `uploads/` and inserts a row into `images`
- `inappropriate.php` — admin-only action to flag an image
- `login.php` / `logout.php` / `register.php` — session-based auth

Database credentials are read exclusively from environment variables (`DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASS`).

User roles: `admin` (sees flagged images, can mark images inappropriate) and `user` (sees only clean images).

### 2. Image Captioning Service (`gcf/`)

A Python Cloud Run service triggered by GCS `object.finalized` events via Eventarc. When an image lands in the GCS bucket (`gcp-app-mod-workshop-public-images`), it calls Vertex AI Gemini (`gemini-2.5-flash`) to generate a caption and writes it back to the `images.description` column in MySQL.

- Entry point: `gcf/main.py` → `generate_caption(cloud_event)`
- Deploy script: `gcf/deploy.sh` (uses `gcloud run deploy --source`)
- DB password pulled from Secret Manager secret `APP_MOD_WORKSHOP_DB_PASS`
- `PROJECT_ID` in `gcf/main.py` must be updated to your GCP project before deploying

## Database

Initialize with the scripts in `db/` in order:

```bash
mysql -u root -p < db/01_schema.sql   # creates image_catalog db, users and images tables
mysql -u root -p < db/02_seed.sql     # seeds 3 users (admin/admin123, user1/user123, user2/user123)
```

The `images` table has a `description` column (populated by the captioning service) and an `inappropriate` flag (set by admins via `inappropriate.php`).

## Running Locally with Docker

```bash
docker build -t php-amarcord .
docker run -p 8080:8080 \
  -e DB_HOST=<host> -e DB_NAME=image_catalog \
  -e DB_USER=<user> -e DB_PASS=<pass> \
  php-amarcord
```

The `/uploads` directory must be writable by the web server. Uncomment `chmod 777` in the Dockerfile only for development.

## Deployment

**CI/CD via Cloud Build:** `cloudbuild.yaml` builds the Docker image and deploys to two Cloud Run services (`${_SERVICE_NAME}-dev` and `${_SERVICE_NAME}-prod`) on every push. Default region: `asia-southeast3`.

**Manual Cloud Run deploy:**
```bash
bash scripts/gcloud-deploy.sh
```

**Deploy the captioning Cloud Run service:**
```bash
cd gcf
pip install -r requirements.txt   # local dev/testing
bash deploy.sh                    # deploys to Cloud Run + creates Eventarc trigger
```

## Key Conventions

- No PHP package manager (no Composer). No linting or automated test tooling is configured.
- The Dockerfile uses `php.ini-development`; switch to `php.ini-production` for hardened deployments.
- The captioning service assumes uploaded images are PNG (`mime_type="image/png"` in `gcf/main.py`) — adjust if other formats are needed.
- GCS bucket name and DB connection details are hardcoded in `gcf/deploy.sh`; update before deploying to a different environment.
