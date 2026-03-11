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

### VS Code Integration (Optional)

If user wants VS Code launch configurations, add to `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Dev Server (Docker)",
      "type": "node-terminal",
      "request": "launch",
      "command": "docker compose up",
      "cwd": "${workspaceFolder}"
    },
    {
      "name": "Dev Server (Docker) - Rebuild",
      "type": "node-terminal",
      "request": "launch",
      "command": "docker compose up --build",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

For Doppler projects, prefix the command with `doppler run --`.

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

## Non-Node Adaptations

### Python (FastAPI/Flask)
- Base image: `python:3.12-slim`
- Install: `pip install -r requirements.txt`
- Run: `uvicorn main:app --host 0.0.0.0 --reload`

### Go
- Base image: `golang:1.22-alpine`
- No hot reload by default (use `air` for live reload)
- Build: `go build -o app .`

### Ruby (Rails)
- Base image: `ruby:3.3-slim`
- Install: `bundle install`
- Run: `rails server -b 0.0.0.0`

---

## Reference: Dynamic Port Allocation

If the user needs dynamic port allocation (e.g., "port 3000 is busy", "I run multiple projects simultaneously"), work with them to create a startup script. This is an iterative process - don't try to generate a perfect solution upfront.

### When to use dynamic ports

- User runs multiple Docker projects that might conflict
- Port 3000 is commonly in use by other tools
- User wants Next.js-like behavior (auto-find available port)

### Logic for dynamic port script

The startup script should provide **sticky ports** - the same port is reused across restarts rather than incrementing each time. This avoids bookmark/muscle-memory issues.

**Three-tier port resolution:**
1. **Container running?** → Get its current port via `docker compose port dev 3000`, stop it, wait until fully gone, reuse that port
2. **Port file exists?** → Read saved port from `.docker-dev-port`, use it if available
3. **Otherwise** → Scan for first free port starting from BASE_PORT (default 3000)

After resolving, save the chosen port to `.docker-dev-port` for next time.

**Waiting for container shutdown:**
- Use `docker compose ps -q` to poll until empty (cross-platform, only requires docker)
- Avoids platform-specific port-checking tools for the wait logic

**Checking if a port is in use:**
- macOS/Linux: `lsof -i :PORT` or `ss -tuln | grep :PORT` or `netstat -tuln | grep :PORT`
- Windows PowerShell: `Get-NetTCPConnection -LocalPort PORT` or `Test-NetConnection -ComputerName localhost -Port PORT`

**Docker compose changes for dynamic ports:**
- Change port mapping to `"${HOST_PORT:-3000}:3000"`
- The script sets HOST_PORT before running docker compose

**Additional files needed when implementing dynamic ports:**
- Add `.docker-dev-port` to `.gitignore` (machine-specific port persistence)
- Add `.docker-dev-port` to `.dockerignore`

### Reference implementation (bash - macOS/Linux)

```bash
#!/bin/bash
# Start dev server in Docker with auto-incrementing port (like Next.js does)
# Ports are sticky across restarts via .docker-dev-port file

BASE_PORT=${PORT:-3000}
MAX_TRIES=20

# Cross-platform port check: prefer lsof, fall back to ss, then netstat
check_port_in_use() {
  local port=$1
  if command -v lsof >/dev/null 2>&1; then
    lsof -i :$port >/dev/null 2>&1
  elif command -v ss >/dev/null 2>&1; then
    ss -tuln | grep -q ":$port "
  elif command -v netstat >/dev/null 2>&1; then
    netstat -tuln | grep -q ":$port "
  else
    return 1  # No tool available, assume port is free
  fi
}

# Check if container is currently running
CURRENT_PORT=$(docker compose port dev 3000 2>/dev/null | cut -d: -f2)

if [ -n "$CURRENT_PORT" ]; then
  # Container running - stop it and reuse same port
  docker compose down 2>/dev/null
  while docker compose ps -q 2>/dev/null | grep -q .; do
    sleep 0.1
  done
  PORT=$CURRENT_PORT

elif [ -f ".docker-dev-port" ]; then
  # Container not running but we have a saved port - try to reuse it
  SAVED_PORT=$(cat .docker-dev-port 2>/dev/null)
  if [ -n "$SAVED_PORT" ] && ! check_port_in_use "$SAVED_PORT"; then
    PORT=$SAVED_PORT
  fi
fi

# If still no port, find an available one
if [ -z "$PORT" ]; then
  port=$BASE_PORT
  max_port=$((BASE_PORT + MAX_TRIES))
  while [ $port -lt $max_port ]; do
    if ! check_port_in_use $port; then
      PORT=$port
      break
    fi
    port=$((port + 1))
  done
  if [ -z "$PORT" ]; then
    echo "ERROR: No available port found between $BASE_PORT and $max_port" >&2
    exit 1
  fi
fi

# Save port for next time
echo "$PORT" > .docker-dev-port

echo ""
echo "========================================="
echo "  Starting dev server on port $PORT"
echo "  http://localhost:$PORT"
echo "========================================="
echo ""

export HOST_PORT=$PORT
docker compose up "$@"
```

**Adapt the last line for secret management:**
- Doppler: `doppler run -- docker compose up "$@"`
- .env files: `docker compose up "$@"`

**Windows (PowerShell)** - create `dev.ps1` instead:
- Use `Get-Content`/`Set-Content` for file operations
- Use `Get-NetTCPConnection -LocalPort $port -ErrorAction SilentlyContinue` for port checks
- Same docker commands work in PowerShell

**Make executable**: `chmod +x dev.sh` (Unix) or adjust execution policy (Windows)

**Platform-appropriate scripts**: When writing shell scripts, use the Bash tool with heredoc syntax (`cat << 'EOF' > dev.sh`) on Unix/macOS to ensure proper LF line endings. For Windows users, create PowerShell scripts (.ps1) instead of bash scripts.
