# Docker Dev Setup — Additional References

## VS Code Integration

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

---

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

## Dynamic Port Allocation

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
