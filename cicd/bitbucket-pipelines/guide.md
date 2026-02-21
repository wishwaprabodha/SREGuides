<div style="font-size: 1.4em;">

# Guide to Bitbucket Pipelines

This guide teaches you Bitbucket Pipelines from the ground up. Each concept is explained independently with its own example, so you can understand the pieces before seeing them combined.

---

## 1. What Is a Pipeline?

A pipeline is an automated workflow that runs every time you push code to Bitbucket. It executes inside ephemeral Docker containers in the cloud — fresh environments spun up for every run. You define what happens in a single YAML file called `bitbucket-pipelines.yml` at the root of your repository.

The core idea:
1. You push code.
2. Bitbucket reads `bitbucket-pipelines.yml`.
3. It spins up Docker containers and runs your commands.
4. The containers are destroyed when done.

---

## 2. The Minimal Pipeline (Hello World)

The simplest possible pipeline:

```yaml
pipelines:
  default:
    - step:
        script:
          - echo "Hello, Pipelines!"
```

- `pipelines` — The root key. Everything lives under it.
- `default` — A catch-all pipeline that runs on **every** push to **any** branch (unless a more specific rule matches).
- `step` — A single unit of work. One Docker container = one step.
- `script` — The shell commands to execute, in order.

This is the skeleton. Everything else is built on top of this.

---

## 3. `image` — Choosing the Docker Container

Every step runs inside a Docker image. You can set a **global** image or override it **per step**.

```yaml
image: node:20                         # Global — used by all steps unless overridden

pipelines:
  default:
    - step:
        name: Use global image
        script:
          - node --version             # Uses node:20
    - step:
        name: Use custom image
        image: python:3.12             # Override — this step uses Python instead
        script:
          - python --version
```

**Key points:**
- If no `image` is specified at all, Bitbucket uses `atlassian/default-image:latest`.
- Per-step `image` overrides the global one for that step only.
- You can use public images (Docker Hub, ECR Public, etc.) or private registries (with credentials).

**Private registry example:**
```yaml
image:
  name: public.ecr.aws/f8b1a4l1/labs/build-node-go:1.1.0
```

---

## 4. `options` — Global Pipeline Settings

The `options` block sits at the top level and configures behavior for the **entire** pipeline.

```yaml
options:
  docker: true     # Enables the Docker daemon inside steps (needed for docker build/push)
  size: 2x         # Doubles the resources: 8GB RAM / 4 vCPUs (default is 4GB / 2 vCPUs)
```

- `docker: true` — Required if any step needs to build or push Docker images. Without this, the `docker` command is not available.
- `size: 2x` — Useful when builds are memory-hungry (e.g., compiling Go, large `npm install`). You can also set `size` per step.

> **Cost note:** `2x` steps consume double build minutes.

---

## 5. `pipelines` — Trigger Conditions

The `pipelines` block defines **when** your steps run. There are several trigger types:

### 5a. `default` — The Catch-All
Runs on any push that doesn't match a more specific rule.

```yaml
pipelines:
  default:
    - step:
        script:
          - echo "This runs on every branch without a specific rule"
```

### 5b. `branches` — Match Specific Branches
Runs only when pushing to branches that match the pattern.

```yaml
pipelines:
  branches:
    master:              # Exact branch name
      - step:
          script:
            - echo "Pushed to master"
    release/*:           # Glob pattern — matches release/anything
      - step:
          script:
            - echo "Pushed to a release branch"
    feature/**:          # Double glob — matches feature/foo, feature/foo/bar, etc.
      - step:
          script:
            - echo "Pushed to a feature branch"
```

**Branch matching rules:**
- Exact names: `master`, `develop`
- Single glob `*`: matches one level (e.g., `release/*` matches `release/1.0` but not `release/api/1.0`)
- Double glob `**`: matches any depth

### 5c. `tags` — Trigger on Git Tags
```yaml
pipelines:
  tags:
    v*:
      - step:
          script:
            - echo "Tag pushed: $BITBUCKET_TAG"
```

### 5d. `pull-requests` — Trigger on PRs
```yaml
pipelines:
  pull-requests:
    '**':                # All PRs
      - step:
          script:
            - npm test
```

### 5e. `custom` — Manual Triggers
Pipelines that only run when you manually trigger them from the Bitbucket UI or API.

```yaml
pipelines:
  custom:
    deploy-hotfix:
      - step:
          script:
            - echo "Manually triggered"
```

---

## 6. `step` — The Unit of Work

A step is one isolated Docker container that runs a sequence of commands. Steps within a branch pipeline run **sequentially** by default (Step 1 finishes, then Step 2 starts).

```yaml
pipelines:
  branches:
    master:
      - step:                          # Step 1
          name: Build
          script:
            - npm install
            - npm run build
      - step:                          # Step 2 (runs AFTER Step 1 succeeds)
          name: Test
          script:
            - npm test
```

**Important:** Each step runs in a **fresh container**. Files created in Step 1 do **not** exist in Step 2 unless you pass them via `artifacts` (covered later).

### Step Properties at a Glance

- `name` — Label for the step in the Bitbucket UI.
- `image` — Override the global Docker image.
- `script` — Shell commands to execute.
- `caches` — Directories to cache between runs.
- `artifacts` — Files to pass to subsequent steps.
- `condition` — Only run this step if certain files changed.
- `services` — Sidecar containers (e.g., databases, Docker daemon).
- `deployment` — Link this step to a deployment environment.
- `after-script` — Commands that always run after `script` (even on failure).

---

## 7. `script` — Shell Commands

The `script` block is a list of shell commands executed in order. If any command exits with a non-zero code, the step fails and the pipeline stops.

```yaml
script:
  - echo "Step 1: Install"
  - npm install
  - echo "Step 2: Build"
  - npm run build
```

### Bash Techniques Used in Pipelines

**Environment Variables:**
Bitbucket provides built-in variables (e.g., `$BITBUCKET_BRANCH`, `$BITBUCKET_COMMIT`) and you can define custom ones in the Bitbucket UI under Repository Settings > Pipelines > Repository variables.

```yaml
script:
  - echo "Branch is $BITBUCKET_BRANCH"
  - echo "Redis URI is $REDIS_URI"       # Custom variable from Bitbucket UI
```

**Command Substitution — `$(...)`:**
Captures the output of a command into a variable.

```yaml
script:
  - export VERSION=$(head -n 1 bind/version)   # Reads first line of the file
  - echo "Deploying version $VERSION"
```

**`sed` — Find and Replace in Files:**
Used to inject dynamic values (versions, secrets, config) into template files.

```yaml
script:
  - export VERSION=$(head -n 1 bind/version)
  - sed -i "s|{{VERSION}}|$VERSION|g" bind/deploy.yml
  - sed -i "s|{{REPLICAS}}|3|g" bind/deploy.yml
```

Breaking down `sed -i "s|{{VERSION}}|$VERSION|g" bind/deploy.yml`:
- `sed` — Stream editor.
- `-i` — Edit the file in-place (modify the file directly).
- `s` — Substitute command.
- `|` — Delimiter (using `|` instead of `/` avoids conflicts with URLs and paths).
- `{{VERSION}}` — The placeholder text to find.
- `$VERSION` — The replacement value.
- `g` — Global (replace all occurrences, not just the first).

---

## 8. `caches` — Speed Up Repeated Builds

Caches persist directories between pipeline runs, so you don't re-download dependencies every time.

### Built-in Caches
Bitbucket has pre-defined cache names for common package managers:

```yaml
- step:
    caches:
      - node          # Caches node_modules (built-in)
    script:
      - npm install   # First run: downloads everything. Subsequent runs: uses cache.
```

### Custom Caches
If the built-in caches don't match your directory, define custom ones in the `definitions` block:

```yaml
definitions:
  caches:
    api: api/node_modules        # Custom cache named "api"
    client: client/node_modules  # Custom cache named "client"
    godir: pkg                   # Custom cache for Go packages

pipelines:
  branches:
    master:
      - step:
          caches:
            - api              # Uses the custom cache defined above
            - client
          script:
            - npm install
```

---

## 9. `artifacts` — Pass Files Between Steps

Since each step runs in a fresh container, artifacts are the only way to share files from one step to the next.

```yaml
pipelines:
  default:
    - step:
        name: Build
        script:
          - npm run build
        artifacts:
          - dist/**          # Everything in dist/ is saved
    - step:
        name: Deploy
        script:
          - ls dist/         # dist/ is available here because it was an artifact
          - ./deploy.sh
```

**Rules:**
- Artifacts use glob patterns (`**` matches recursively).
- They are only available to **subsequent steps in the same pipeline**.
- Without `artifacts`, files from Step 1 simply don't exist in Step 2.

---

## 10. `condition` — Conditional Step Execution (Changesets)

You don't want every step to run on every push. Conditions let you skip steps when irrelevant files changed.

```yaml
- step:
    name: Deploy API
    condition:
      changesets:
        includePaths:
          - api/**              # Only run if files under api/ changed
          - api/package.json    # Or you can target specific files
    script:
      - echo "API files changed, deploying..."
```

**How it works:**
- Bitbucket compares the diff of the push.
- If **none** of the files in the diff match the `includePaths`, the step is **skipped**.
- If **at least one** file matches, the step **runs**.

This is essential in monorepos (like this one) where multiple services live in the same repository. Changing `bind/version` should only trigger Bind steps, not API steps.

---

## 11. `services` — Sidecar Containers

Services are additional Docker containers that run alongside your step. Common uses: databases for tests, or the Docker daemon for building images.

```yaml
pipelines:
  default:
    - step:
        services:
          - docker           # Runs the Docker daemon as a sidecar
        script:
          - docker build -t myimage .
          - docker push myimage

definitions:
  services:
    docker:
      memory: 4096           # Give the Docker daemon 4GB RAM
```

**The Docker Service:**
When `options: docker: true` is set globally, you still need `services: [docker]` in steps that perform `docker build` or `docker push`. The `definitions` block lets you configure the service (e.g., allocating more memory).

---

## 12. `pipes` — Pre-Built Integrations

Pipes are reusable, vendor-maintained integrations that simplify complex tasks. Instead of writing 20 lines of bash to authenticate with AWS, configure `kubectl`, and apply a manifest, you use a pipe.

### AWS ECR Push Pipe
Pushes a Docker image to Amazon Elastic Container Registry:

```yaml
- pipe: atlassian/aws-ecr-push-image:2.6.0
  variables:
    AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
    AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    AWS_DEFAULT_REGION: 'ap-southeast-1'
    IMAGE_NAME: wisdom/dnsapi
    TAGS: $VERSION
```

### AWS EKS Kubectl Pipe
Authenticates with an EKS cluster and runs a `kubectl` command:

```yaml
- pipe: atlassian/aws-eks-kubectl-run:3.2.0
  variables:
    AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
    AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    AWS_DEFAULT_REGION: "ap-southeast-1"
    CLUSTER_NAME: 'sg-brs-1'
    KUBECTL_COMMAND: 'apply'
    RESOURCE_PATH: bind/deploy.yml
```

What this pipe does under the hood:
1. Configures AWS credentials.
2. Runs `aws eks update-kubeconfig` to set the kubectl context.
3. Runs `kubectl apply -f bind/deploy.yml` against the specified cluster.

### GCP GKE Kubectl Pipe
Same concept, but for Google Kubernetes Engine:

```yaml
- pipe: atlassian/google-gke-kubectl-run:3.5.0
  variables:
    KEY_FILE: $GCP_KEY_BASE64
    PROJECT: 'kirk-1588691722895'
    COMPUTE_ZONE: 'asia-southeast1'
    CLUSTER_NAME: 'wisdom-k8s-cluster-1'
    KUBECTL_COMMAND: 'apply'
    RESOURCE_PATH: api/deploy-gcp.yml
```

**Pipe variables use `${VAR}` or `$VAR`** — both reference repository/deployment variables you set in the Bitbucket UI.

---

## 13. `deployment` — Deployment Environments

Deployment environments let you track which version of code is running in each environment (staging, production, etc.).

```yaml
- step:
    name: Release to Production
    deployment: production       # Links this step to the "production" environment
    script:
      - echo "Deploying to prod"
```

**Why this matters:**
- Bitbucket tracks deployment history per environment.
- You can set environment-specific variables (e.g., different `REDIS_URI` for staging vs. production).
- The Bitbucket UI shows which commit is currently deployed to each environment.
- You can require manual approval before production deployments.

---

## 14. `after-script` — Cleanup Commands

Commands in `after-script` always run after `script`, **even if the script failed**. Use this for cleanup, notifications, or tagging.

```yaml
- step:
    name: Build and Tag
    script:
      - export VERSION=$(head -n 1 bind/version)
      - docker build -t wisdom/bind:$VERSION .
    after-script:
      - export VERSION=$(head -n 1 bind/version)
      - git tag bind/$VERSION
      - git push origin bind/$VERSION
```

**Note:** Variables exported in `script` are **not** available in `after-script`. You must re-export them.

---

## 15. `definitions` — Reusable Configuration

The `definitions` block sits at the top level and defines reusable caches, services, and step templates.

```yaml
definitions:
  caches:
    api: api/node_modules
    client: client/node_modules
    dnsmw: dnsmw/node_modules
    godir: pkg
  services:
    docker:
      memory: 4096
```

Anything defined here can be referenced by name in any step.

### YAML Anchors (DRY Principle)
If you have repeated steps, use YAML anchors to avoid duplication:

```yaml
definitions:
  steps:
    - step: &run-tests
        name: Run Tests
        script:
          - npm test

pipelines:
  branches:
    develop:
      - step: *run-tests        # Reuse the anchored step
    master:
      - step: *run-tests        # Same step, no duplication
```

---

## 16. Environment Variables and Secrets

### Where Variables Come From

1. **Built-in Variables** — Provided automatically by Bitbucket:
   - `$BITBUCKET_BRANCH` — Current branch name.
   - `$BITBUCKET_COMMIT` — Full commit hash.
   - `$BITBUCKET_TAG` — Tag name (for tag-triggered pipelines).
   - `$BITBUCKET_REPO_SLUG` — Repository name.

2. **Repository Variables** — Set in Bitbucket UI > Repository Settings > Pipelines > Repository variables.
   - Available to **all** pipelines in the repo.
   - Example: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

3. **Deployment Variables** — Set per deployment environment.
   - Only available to steps with a matching `deployment:` value.
   - Example: `REDIS_URI` might differ between staging and production.

### Security
- Mark variables as **Secured** in the UI. They will be masked (`*****`) in pipeline logs.
- Never hardcode secrets in `bitbucket-pipelines.yml`.
- Use `${VAR_NAME}` syntax in pipe variable blocks.

---

## 17. Putting It All Together

Here is a complete example showing every concept in action:

```yaml
# ── Global Settings ──────────────────────────────────────────────
options:
  docker: true                    # [§4]  Enable Docker daemon
  size: 2x                        # [§4]  Double resources

# ── Trigger Rules & Steps ────────────────────────────────────────
pipelines:
  branches:
    master:
      - step:                     # [§6]  Step 1: Build
          name: Build App
          image:                  # [§3]  Per-step image override
            name: public.ecr.aws/f8b1a4l1/labs/build-node-go:1.1.0
          caches:                 # [§8]  Custom caches
            - api
          condition:              # [§10] Only if these files changed
            changesets:
              includePaths:
                - api/package.json
          script:                 # [§7]  Shell commands
            - make all
            - export VERSION=$(cd api && npm run version --silent)
            - pipe: atlassian/aws-ecr-push-image:2.6.0   # [§12] Pipe
              variables:
                AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
                AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
                AWS_DEFAULT_REGION: 'ap-southeast-1'
                IMAGE_NAME: wisdom/dnsapi
                TAGS: $VERSION
          services:               # [§11] Docker sidecar
            - docker
          after-script:           # [§14] Always runs — tag the version
            - export VERSION=$(cd api && npm run version --silent)
            - git tag api/$VERSION
            - git push origin api/$VERSION

      - step:                     # [§6]  Step 2: Deploy
          name: Deploy App
          condition:
            changesets:
              includePaths:
                - api/package.json
          script:
            - export VERSION=$(cd api && npm run version --silent)
            - sed -i "s|{{VERSION}}|$VERSION|g" api/deploy.yml    # [§7] sed
            - pipe: atlassian/aws-eks-kubectl-run:3.2.0           # [§12] EKS pipe
              variables:
                CLUSTER_NAME: 'sg-staging-1'
                KUBECTL_COMMAND: 'apply'
                RESOURCE_PATH: api/deploy.yml

    release/api/*:               # [§5b] Glob branch pattern
      - step:
          name: Release to Production
          deployment: production  # [§13] Deployment environment
          script:
            - echo "Production deployment"

# ── Reusable Definitions ─────────────────────────────────────────
definitions:                      # [§15]
  caches:
    api: api/node_modules
  services:
    docker:
      memory: 4096
```

The `[§N]` annotations reference the sections in this guide where each concept is explained.

---

## 18. Quick Reference

**Top-Level Keys:**
- `options` — Global settings (docker, size)
- `image` — Default Docker image
- `pipelines` — Trigger rules and steps
- `definitions` — Reusable caches, services, step templates

**Step-Level Keys:**
- `name` — Display name
- `image` — Docker image override
- `script` — Commands to run
- `after-script` — Cleanup commands (always run)
- `caches` — Cached directories
- `artifacts` — Files to pass to next step
- `condition` — Changeset-based filtering
- `services` — Sidecar containers
- `deployment` — Link to environment

**Pipeline Trigger Types:**
- `default` — Any branch without a specific rule
- `branches` — Specific branch patterns
- `tags` — Git tag patterns
- `pull-requests` — Pull request events
- `custom` — Manual triggers

---

## 19. Troubleshooting Tips

- **Pipeline fails immediately** — Check YAML indentation. Use Bitbucket's YAML validator.
- **"docker: not found"** — Ensure `options: docker: true` is set and `services: [docker]` is on the step.
- **Step runs when it shouldn't** — Check `condition.changesets.includePaths`. The paths are relative to the repo root.
- **Step skipped unexpectedly** — The changeset condition found no matching file changes in the push.
- **Variables show as empty** — Verify the variable is defined in Bitbucket UI. Check spelling and case.
- **Files missing between steps** — You need `artifacts` to pass files between steps.
- **Out of memory** — Use `size: 2x` at the options or step level. Increase `services.docker.memory` for Docker builds.

</div>
