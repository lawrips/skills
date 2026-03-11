---
name: docker-dev-setup
description: Set up isolated Docker dev environment for web projects. Use when asked to add Docker, containerize development, or improve dev security.
disable-model-invocation: true
compatibility: Requires Docker. Claude Code only.
metadata:
  author: lawrips
  version: 1.2.0
---

# Docker Dev Environment Setup

Containerized development environment for consistent, isolated local development.

**Scope**: This skill is tuned for Node.js/Next.js projects but the patterns apply to any stack Docker supports. Adapt the base image, install commands, and run commands for other frameworks.

## Before Starting

1. **Detect the tech stack** from existing files (package.json, requirements.txt, go.mod, etc.)
2. **Detect secret management** from existing files:
   - `.env` / `.env.local` → use `--env-file` or mount
   - `doppler.yaml` → wrap commands with `doppler run --`
   - `.envrc` / direnv → similar to .env approach
   - Nothing yet → ask user how they want to manage secrets
3. **Ask the user**:
   - What port to use (default: 3000)
   - If they want VS Code launch.json configurations added

## Benefits

- **Consistent environment**: No "works on my machine" - everyone runs identical setup
- **Easy onboarding**: Clone repo, run one command, done
- **Dependency isolation**: Node versions, native modules don't pollute host
- **Production parity**: If deploying containers, dev matches prod
- **Security isolation**: Limits blast radius from RCE vulnerabilities (see Security Hardening section)

## Files to Create

Create these files in the project root:

### 1. Dockerfile.dev

```dockerfile
FROM node:20-alpine

# Security: run as non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy package files first (for layer caching)
COPY --chown=appuser:appgroup package.json package-lock.json ./

# Install dependencies and create build dir with correct ownership
RUN npm ci && \
    mkdir -p /app/.next && \
    chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Disable telemetry
ENV NEXT_TELEMETRY_DISABLED=1

# Hot reload needs polling in Docker
ENV WATCHPACK_POLLING=true

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### 2. docker-compose.yml

Use the port the user specified (default 3000).

```yaml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"  # ADAPT: Use the port user specified

    environment:
      - NODE_ENV=development
      - NEXT_TELEMETRY_DISABLED=1
      - WATCHPACK_POLLING=true
      # ADAPT: Add project secrets here - search codebase for process.env usage
      # to discover what env vars are needed, then list them here.
      # These pass through from host env (for Doppler, direnv, etc.)
      # - DATABASE_URL
      # - API_KEY

    # ADAPT: Uncomment if using .env files
    # env_file:
    #   - .env.local

    volumes:
      # ADAPT: Mount source directories for hot reload
      - ./pages:/app/pages
      - ./components:/app/components
      - ./lib:/app/lib
      - ./styles:/app/styles
      - ./contexts:/app/contexts
      - ./public:/app/public:ro

      # ADAPT: Config files (read-only)
      - ./next.config.js:/app/next.config.js:ro

      # Keep node_modules inside container (not from host)
      - node_modules:/app/node_modules

      # Keep build cache in container
      - next_build:/app/.next

    # Security hardening
    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

volumes:
  node_modules:
  next_build:
```

### 3. .dockerignore

```
node_modules
.next
out
.git
.gitignore
.vscode
.idea
.DS_Store
*.log
npm-debug.log*
.env*
docs
*.md
tests
```

## Adaptation Checklist

When setting up for a new project:

1. **Port**: Update the port mapping in docker-compose.yml
2. **Source volumes**: Update mounts to match project's directory structure
3. **Config files**: Add any config files needed at runtime (read-only where possible)
4. **Environment vars**: Search codebase for `process.env` usage to discover required vars, then list them (this is an allowlist - unlisted vars don't enter container)
5. **Base image**: Change `node:20-alpine` if not a Node project
6. **Build command**: Update `npm ci` / `npm run dev` for your package manager/framework
7. **.dockerignore**: Add project-specific directories (mobile builds, scripts, etc.)

## After Setup

Let the user know: "If you need dynamic port allocation later (e.g., running multiple projects, port conflicts), let me know and we can set that up."

## Handling Port Conflicts

If the user reports port conflicts or says they run multiple projects:
1. Don't just change the static port number - that doesn't solve the underlying problem
2. Refer to the "Reference: Dynamic Port Allocation" section at the bottom of this skill
3. Work with the user to implement the dev.sh approach for sticky dynamic ports

## Usage

Requires: Docker installed

```bash
# First time / after package.json changes
docker compose up --build

# Normal startup
docker compose up

# Stop
docker compose down

# Background mode
docker compose up -d
docker compose logs -f

# Reset volumes (if permission issues)
docker compose down -v
docker compose up --build
```

### Secret Management Variations

Adapt the commands based on how the project manages secrets:

**Using .env files** (env_file in docker-compose.yml):
```bash
docker compose up
```

**Using Doppler**:
```bash
doppler run -- docker compose up
```

**Using direnv** (secrets loaded into shell):
```bash
docker compose up  # picks up env vars from shell
```

## Security Hardening

The templates above include security hardening that limits blast radius if an attacker gets RCE (e.g., via dependency supply chain attack or framework exploit).

### Config Choices and Why

| Config | Location | Why |
|--------|----------|-----|
| Non-root user (`appuser`) | Dockerfile | Limits what attacker can do inside container |
| `no-new-privileges: true` | docker-compose.yml | Prevents privilege escalation via setuid binaries |
| `cap_drop: ALL` | docker-compose.yml | Removes all Linux capabilities |
| Named volumes for node_modules | docker-compose.yml | Host node_modules not accessible to container |
| Read-only mounts (`:ro`) | docker-compose.yml | Limits write access to config files |
| Explicit env var list | docker-compose.yml | Only listed vars enter container (allowlist pattern) |

### What This Protects

If attacker gets RCE, they **cannot** access:
- Home directory (~/.ssh, ~/.aws, other credentials)
- Other projects on machine
- Host system files
- Persistence mechanisms (cron, launch agents)

### What This Doesn't Protect

If attacker gets RCE, they **can** access:
- All environment variables passed to container
- Mounted source code (read/write)
- Outbound network (exfiltration)

**Key insight**: If your app needs a secret to function, and an attacker has RCE in your app, they can get that secret. Docker isolation provides **blast radius reduction, not secret protection**. Your machine and other projects stay safe; this project's secrets do not.

## Additional References

For VS Code integration, non-Node adaptations (Python, Go, Ruby), and dynamic port allocation, see `references/extras.md`.
