# OneCP – Design & Overview

## 1. Problem OneCP is Trying to Solve

Modern teams that use Ansible usually end up in one of two worlds:

- Pure CLI / scripts – a handful of people know the playbooks and run `ansible-playbook` manually, often via shell scripts or CI jobs.
- Ansible AWX / Tower – a full automation platform with a web UI, REST API, inventories, credentials, job templates, RBAC and scheduling.

At the same time, there is a separate ecosystem of self-hosted PaaS panels (Coolify, CapRover, Dokploy) that aim to be “Heroku/Vercel on your own VPS”, with dashboards, Git integration and one-click services, but they are primarily Docker-centric.

There is a gap between these two:

- Teams that already codify their infrastructure with Ansible, but don’t want to run a full AWX instance or expose Ansible internals to every developer.
- Agencies and small product teams that prefer “deploy my app to these servers via playbooks” over “manage a Docker PaaS”, because they already have playbooks and existing server layouts.

OneCP exists exactly in that gap:

> A small, self-hosted control panel that wraps Ansible playbooks in an app-centric deployment UX, without trying to be a full PaaS or a full automation platform.

The key constraint is: Ansible stays the source of truth for deployment operations, but OneCP acts as the editor and runner. OneCP is a thin layer on top that makes everyday deployments easier.

---

## 2. Core Concepts

OneCP deliberately has a small set of core concepts. If you understand these, you understand the product.

### 2.1 Servers and Environments

- Server – a machine reachable over SSH (VPS or bare-metal).  
  Minimal fields: name, host/IP, SSH port, user, auth method (SSH key / password), tags.

- Environment – a logical grouping of servers, e.g. `production`, `staging`, `dev`.  
  Environments are where apps are deployed. They can be mapped to Ansible inventories or groups (e.g. `web_servers`, `db_servers`).

This keeps the mental model close to what teams already use: “this app goes to these hosts in this environment”.

### 2.2 Apps

An App represents a deployable unit (API service, worker, frontend, etc.):

- Metadata: name, description, repository URL.
- Relationships:
  - Has one or more environments.
  - Has one or more deployment recipes.

There is intentionally no deep magic here. An app in OneCP is mostly a way to:

- Group deployments and their history.
- Group environment-specific configuration (env vars).
- Attach recipes that know how to deploy that app.

### 2.3 Playbooks and Recipes

Playbooks are stored directly in OneCP's database, allowing you to manage and edit your automation logic right from the UI without requiring an external Git repository for your infra.

A Recipe is an application-facing wrapper around a playbook:

- References:
  - A specific playbook (managed internally within OneCP).
  - A target inventory or group (where to run it).
- Configuration:
  - Default variables.
  - Which environment it is meant for (production, staging, etc.).

In practice, a recipe corresponds to things like:

- “Deploy the API to production”
- “Deploy the worker to staging”
- “Roll back the last release”

Developers see recipes; ops maintain the underlying playbooks directly within OneCP.

### 2.4 Deployments

A Deployment is a single execution of a recipe with a concrete set of variables and a commit/branch reference:

- Inputs:
  - App, environment, recipe.
  - Git commit or branch/tag.
  - Variable values (from env vars + form input).
- Outputs:
  - Status (success/fail/cancel).
  - Start/end time.
  - Ansible output log (per task, per host).
  - Host/task summary (changed, skipped, failed).

Deployments have two entry points:

- Manual: a human clicks “Deploy” in the UI and fills a small form.
- Automatic: a Git webhook fires when code is pushed, triggering a recipe with preconfigured behaviour, similar to what Coolify and Dokploy do for container-based deploy flows.

---

## 3. High-Level Architecture

This section describes how OneCP is intended to be built. Exact implementation details can change, but the structure should remain stable.

### 3.1 Backend

The backend has three responsibilities:

1. API and storage layer
   - Expose API endpoints for:
     - Servers, environments, apps, playbooks, recipes, deployments, env vars.
   - Persist data in PostgreSQL (including playbook definitions).
   - Enforce basic RBAC (e.g. who can deploy to production vs staging).

2. Ansible execution layer
   - Accept “run this recipe with these parameters” from the API.
   - Use `ansible-runner` or similar to execute the underlying playbook:
     - Dump the playbook from the DB to a temporary file.
     - Read inventories and vars.
     - Stream output lines/tasks back to the frontend.
     - Capture final status and statistics.

3. Webhook and scheduling
   - Receive Git webhooks.
   - Translate “push to branch X” into “trigger recipe Y for app Z”.
   - Potentially run minimal schedules (e.g. nightly maintenance jobs) in later versions.

### 3.2 Frontend

The frontend is a browser-based UI that does not hide the realities of deployment, but makes them easier to manage:

- Dashboard
  - List of apps and their last deployment state.
  - Quick view of environments and server health.

- Servers page
  - Add/update servers.
  - Test SSH connectivity.
  - View basic tags and environment assignments.

- Playbooks page
  - Create and edit Ansible playbooks directly via a code editor in the browser.

- App page
  - Shows branches, environments, and recipes.
  - Shows deployments history with commit hashes and statuses.

- Deployment view
  - Streams Ansible output live in a terminal-like view.
  - Highlights failures and lets users filter by host or task.

- Configuration/Env vars
  - Simple key-value editor per app + environment.
  - Values are passed into Ansible via `extra_vars` or inventory vars.

The UI should be straightforward: no nested configuration wizards, no hidden magic states. If something fails, the log should show exactly what happened.

---

## 4. Typical Workflow

This section is meant to show how OneCP should feel in daily use.

### 4.1 First Setup

1. Deploy OneCP itself  
   - Run via Docker Compose on a small VPS (for early versions).
   - Connect it to a PostgreSQL instance.

2. Add servers
   - Enter host/IP, user, port and SSH credentials.
   - Tag servers `prod`, `staging`, etc.
   - Hit “Test Connection” to confirm SSH works.

3. Create Playbooks
   - Write your Ansible playbooks directly within the OneCP UI.
   - Playbooks are stored safely in the database and can be edited anytime.

4. Define environments
   - Create `production`, `staging`, `dev`.
   - Map them to inventories/groups where needed.

### 4.2 Adding an App

1. Create an App:
   - Name: `api-service`.
   - Connect it to your app's Git repository.
   - Attach `production` and `staging` environments.

2. Add Recipes:
   - `Deploy API to production` → maps to `deploy_api.yml` against `web_servers` group.
   - `Deploy API to staging` → same playbook, different inventory group.

3. Configure Env vars:
   - In `staging`, set `DEBUG=true`, `DOMAIN=staging.example.com`.
   - In `production`, set `DEBUG=false`, `DOMAIN=api.example.com`.

### 4.3 Running a Deployment

1. A developer pushes a commit to the main branch.
2. Either:
   - Manually: they open OneCP, choose the app, click “Deploy to production”, pick the commit/branch and confirm.
   - Automatically: a Git webhook triggers the recipe with preconfigured defaults.

3. OneCP:
   - Resolves server list from the `production` environment.
   - Prepares Ansible vars from env vars + form inputs.
   - Dumps the playbook from the DB.
   - Starts `ansible-runner` with the chosen playbook and inventory.

4. The deployment view:
   - Shows tasks and host output in real time.
   - Marks changed hosts and failed tasks.

5. When finished:
   - The deployment history entry is updated.
   - The commit, duration, and summary are recorded for future reference.

Over time, teams build a rhythm around this flow. The important thing is that the same playbooks can still be run manually if exported; OneCP doesn’t lock them in, though it acts as the primary source of truth.

---

## 5. Relationship to Existing Tools

OneCP is not trying to compete with everything. It has a reasonably narrow band of responsibility.

### 5.1 Compared to AWX

AWX is an upstream project for Red Hat Ansible Automation Platform. It provides:

- Web UI for projects, inventories, credentials, job templates.
- RBAC, schedules, notifications.
- REST API and a full task engine.

OneCP differs in intent:

- Smaller scope: mostly focused on deployments, not general automation.
- App-centric: emphasises “apps + environments + recipes” instead of “jobs + templates”.
- Simpler: one VPS install, fewer moving parts, suitable for small teams and agencies.
- Independent: OneCP runs playbooks natively and does not require an AWX instance or any connection to it.

### 5.2 Compared to Coolify / CapRover / Dokploy

Self-hosted PaaS tools like Coolify, CapRover, and Dokploy focus on:

- Managing Docker containers and services.
- One-click databases and external services.
- SSL, reverse proxy, backing up databases.
- Git-push or CI-based deployments.

They are great if your deployment model is “build an image, run it in a container”.

OneCP is for teams whose deployment model is “run a playbook against servers”:

- Ansible stays at the centre.
- No assumption that apps are containerised.
- No need to adopt a particular PaaS stack just to get a basic deployment dashboard.

---

## 6. Scope and Future Work

This section is intentionally explicit about what might be added later and what is out-of-scope.

### 6.1 Planned Extensions

- Git Integrations
  - First-class support for GitHub/GitLab/Bitbucket/Gitea webhooks.
  - Mapping branches to environments (e.g. `main` → production, `develop` → staging).

- Secrets Backend
  - Pluggable secret storage (Vault, cloud secret managers).
  - Stronger controls around who can view or edit secrets.

- RBAC and Audit
  - Per-app permissions.
  - Central audit log of deploy actions, configuration changes.

### 6.2 Explicit Non-Goals (for now)

- Becoming a full CI/CD system.
- Managing Kubernetes clusters or Docker Swarm directly (that’s a different product class).
- Becoming a general-purpose IT automation platform.

If the project ever grows into those areas, it should do so carefully, without losing the “small control panel on top of Ansible” identity.

---

## 7. Practical Notes

- Use it on non-critical infra first.  
  Early versions will change schemas and APIs. Treat this as an experimental path to deployment, not a source of truth until it stabilises.

- Playbooks live in OneCP.  
  You manage and version your playbooks directly in OneCP's database, simplifying the workflow and removing dependency on external repos for infrastructure logic.

- Design for failure visibility.  
  If a deploy fails, the Ansible log is the ground truth. OneCP’s job is to make it easy to get there quickly.

- Prefer small, incremental features.  
  The project should grow by solving real, observed pain points rather than chasing feature parity with large platforms.