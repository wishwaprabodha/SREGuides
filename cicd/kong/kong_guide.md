# Kong DB-less API Gateway: Comprehensive Guide & Tutorial

This document walks through how Kong is configured in a *DB-less* mode using declarative YAML files, illustrated with examples from the `staging/staging-sg-api-gateway.yml` and `production/production-sg-api-gateway.yml` files in this workspace. It also includes a step‑by‑step tutorial for applying and managing these configurations.

---

## 🧠 What Is Kong DB-less?

Kong is a high‑performance API gateway built on NGINX. In **DB-less** mode, the gateway stores its configuration entirely in memory and is bootstrapped from a declarative file (YAML or JSON). There is no database; you manage your services, routes, and plugins by editing config files and pushing them to the gateway.

Advantages:

- Simplified deployment (no database to manage)
- Version‑controlled configuration
- Immutable, reproducible gateway state

This repository contains two environment-specific files for Singapore (SG):

- `staging/staging-sg-api-gateway.yml`
- `production/production-sg-api-gateway.yml`

Each file defines a list of services, with associated routes and plugins.

---

## 📁 Declarative Configuration Structure

The top‑level YAML consists of a single document with these key sections:

```yaml
_format_version: "1.1"  # schema version for Kong
services:                # array of service objects
  - name: ...
    url: ...
    plugins: [...]       # optional, service‑level plugins
    routes: [...]        # one or more routes bound to the service
```

### Services

A **service** represents an upstream API. Each service entry must have:

- `name`: unique identifier within Kong
- `url`: the upstream target (scheme://host[:port]/[path])

Example from staging:

```yaml
- name: SSG-DISCOVER
  url: http://ssg-discover.circles.life
  plugins:
    - name: cors
    - name: circles-authorization-plugin
      config:
        allow_guest: true
    - name: circles-jwt-plugin
      config: ...
  routes: [...]
```

Production example:

```yaml
- name: PSG-DiscoverBackend
  url: http://saru.circles.life:80
  plugins:
    - name: newrelic-insights
      config: ...
    - name: circles-authorization-plugin
      config:
        allow_guest: true
    - name: cors
    - name: circles-jwt-plugin
      config: ...
  routes:
    - name: PSG-V3DiscoverMBE
      strip_path: false
      hosts: ["app.circles.asia", "appbeta.circles.asia"]
      paths:
        - /api/v3/discover
```

> 🔑 **Tip:** Service‑level plugins apply to every request that matches any of its routes. Route‑level plugins can override or supplement them.

### Routes

A **route** tells Kong how to match incoming requests and forward them to the associated service. Common route fields:

- `hosts`: host header(s) to match
- `paths`: path prefixes or regex (see `strip_path` and `regex_priority`)
- `methods`: HTTP verbs to match
- `headers`: header field matchers
- `strip_path`: whether the matched path portion is removed when proxying
- `regex_priority`: higher values evaluated first when using regex paths

Example:

```yaml
- name: SSG-DISCOVER-GENERAL
  strip_path: false
  hosts: ["ssg-app.circles.life", "ssg-appbeta.circles.life"]
  paths:
    - /api/v3/discover
    - /api/v4/discover
```

Routes may also have their own plugins; e.g. production restricted endpoint:

```yaml
- name: PSG-V3DiscoverRestricted
  regex_priority: 1
  plugins:
    - name: circles-authorization-plugin
      config:
        allow_guest: false
  methods:
    - POST
    - PUT
  paths:
    - /api/v3/discover/article/.+/like/.+
    - /api/v3/discover/article/.+/unlike/.+
```

### Plugins

Kong supports a rich plugin ecosystem. In these configs you’ll see:

- `cors` – cross‑origin resource sharing
- `rate-limiting` – basic throttling (limit_by: service/IP, seconds/minutes)
- `circles-jwt-plugin` – custom JWT authorizer with Redis key store
- `circles-authorization-plugin` – another custom plugin toggling guest access
- `newrelic-insights` – telemetry integration

Each plugin has a `config` section with type‑specific settings. Secrets such as Redis passwords and public keys are embedded directly.

---

## 🛠 Working with the Files

### Prerequisites

1. **Kong Gateway** (2.x or later) running in DB-less mode. Example `kong.conf`:

   ```ini
   database = off
   declarative_config = /path/to/your/config.yml
   ```

2. **decK** – the official CLI for Kong declarative configuration. Install via `brew install deck` or from the [GitHub releases](https://github.com/hbagdi/deck).

3. Access to the Kong Admin API (e.g. `http://localhost:8001`).

### Syncing a Configuration

```bash
# validate the YAML before applying
deck check --state staging/staging-sg-api-gateway.yml --insecure

# push to gateway (overwrites current state)
deck sync --state staging/staging-sg-api-gateway.yml --insecure
```

`--insecure` is needed if Kong’s admin API uses a self‑signed certificate. Omit in production with trusted TLS.

> 🔍 `deck diff` can show changes between local file and running state.

### Inspecting Resources

Once synced, you can use the Admin API or decK to query objects:

```bash
curl localhost:8001/services
curl localhost:8001/routes/SSG-DISCOVER-GENERAL
```

Or with decK:

```bash
deck dump --format yaml | grep -A5 'name: SSG-DISCOVER'
``` 

### Deploy Workflow

1. Edit the appropriate YAML (staging or production).
2. Commit changes to version control.
3. Run `deck check` to catch syntax errors.
4. Execute `deck sync` against the target environment.
5. Verify via Admin API or by sending test requests through Kong.
6. Roll back by applying the previous Git commit if needed.

---

## 📝 Tutorial: From Staging to Production

1. **Start a local Kong instance** (Docker example):

   ```bash
   docker run -d --name kong \
     -e "KONG_DATABASE=off" \
     -e "KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative.yml" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -p 8000:8000 -p 8443:8443 -p 8001:8001 -p 8444:8444 \
     kong:2.8
   ```

2. **Apply staging config to the running gateway**:

   ```bash
   deck sync -s staging/staging-sg-api-gateway.yml -k http://localhost:8001
   ```

3. **Test a route**:

   ```bash
   curl -H "Host: ssg-app.circles.life" http://localhost:8000/api/v3/discover
   ```

   Expect a proxied response from the upstream `http://ssg-discover.circles.life`.

4. **Modify a service** (e.g. add a rate limit):

   ```yaml
   - name: SSG-INVENTORY
     url: http://ssg-inventory.circles.life
     plugins:
       - name: rate-limiting
         config:
           second: 25
           policy: local
     routes:
       ...
   ```

   Then run `deck sync` again and retest.

5. **Promote to production**: once staging changes are validated, merge the YAML changes to the `main` branch (or your release branch) and run `deck sync` using the `production/production-sg-api-gateway.yml` file against the production Kong cluster.

---

## 🔁 Environment Differences

- **Hostnames**: staging uses `ssg-app.circles.life`/`qsg-app.circles.life`, production uses `app.circles.asia`.
- **Upstream URLs**: vary between `ssg-`/`qsg-` prefixes in staging and `saru`, `lovok` etc. in production.
- **Plugins**: production often includes `newrelic-insights` and higher rate limits.
- **Secrets**: Redis passwords, API keys, and public keys differ.

> Maintain separate files per environment to avoid accidental cross‑promotion of credentials.

---

## ⚠️ Best Practices & Troubleshooting

- **Regex priority**: assign `regex_priority` to routes using regex paths to control evaluation order.
- **Strip path**: set to `true` when the upstream service expects the matched portion removed.
- **Plugin ordering**: the order in the `plugins` array determines execution order.
- **Avoid exposing secrets**: consider using environment variables or a secrets manager with `deck` templates if the repo is public.
- **Validation**: always run `deck check` before syncing; it catches malformed YAML and missing fields.
- **Rollback**: keep previous state files accessible or rely on Git history.

---

## 📌 Summary

This guide provided an overview of how Kong operates in DB-less mode, explained the structure of the SG staging and production configuration files, and gave a hands-on tutorial for syncing and testing changes. With proper version control and the `decK` CLI, you can manage your gateway configuration reliably across environments.

Feel free to expand this document with additional environment notes, plugin descriptions, or advanced topics such as Kong clustering, health checks, or migrations to hybrid DB mode.

---

*Last updated: 26 February 2026*
