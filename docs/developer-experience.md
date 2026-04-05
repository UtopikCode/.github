# Developer Experience Guide

## Typical Workflow

### 1. Start Your Day

You open a repository you want to work on. It has a `.devcontainer` config:

```bash
# Clone or open repo
cd my-app
# (repo has .devcontainer/ already configured)
```

### 2. Create a Workspace

Using the VS Code Quickspaces extension:

1. Open command palette: `Ctrl+Shift+P` (or `Cmd+Shift+P`)
2. Select: "Quickspaces: Create Workspace"
3. Extension connects to Quickspaces control plane
4. You authenticate with GitHub (one-time)
5. Control plane checks: do you have access to this repo? ✓
6. Workspace ID assigned

### 3. Quickspaces Provisions

Behind the scenes:

1. Control plane selects a runner (on AWS, on-prem, etc.)
2. Runner clones your repo and branch
3. Runner reads `.devcontainer/devcontainer.json`
4. Docker pulls/builds the dev image
5. Docker Compose services start (database, cache, etc.)
6. VS Code Server boots inside the container
7. SSH port is assigned and reported back

All this takes **1–5 minutes**, depending on image size and network.

### 4. VS Code Connects

The extension receives connection details:

- SSH host: `runner-xyz.internal`
- Port: `2222`
- User: `quickspaces`

VS Code Remote - SSH extension connects automatically. You're now inside your dev container:

```bash
$ # You're inside the container now
$ node --version
v18.16.0
$ npm run dev
# Your full dev stack is running
```

Your local VS Code is just a window into the remote environment. All tools are on the runner.

### 5. Work Normally

Full Docker Compose support means you get everything you'd have locally:

- **Database**: PostgreSQL running on `localhost:5432`
- **Cache**: Redis on `localhost:6379`
- **File watching**: Changes sync from your local editor to the container
- **Port forwarding**: `npm run dev` exposes `localhost:3000` in your container
- **Extensions**: VS Code extensions work (some client-side, some server-side)
- **Debugging**: Full debugging support with VS Code debugger

Everything feels local, but it's running on the runner.

### 6. Take a Break

You step away for lunch. The workspace detects no activity (no file changes, no commands, no terminal input):

- After ~15–30 minutes (configurable), workspace transitions to `idle` state
- Containers are paused (or stopped)
- Compute resources are released (or right-sized)
- **Billing stops** (in Quickspaces Cloud)

Your SSH connection disconnects gracefully.

### 7. Come Back Later

You return and want to continue:

1. Click "Resume Workspace" in VS Code extension
2. Workspace transitions from `idle` to `started`
3. Containers resume (or restart)
4. VS Code reconnects
5. You're back where you left off

**No setup needed.** Everything is ready.

### 8. Clean Up

When you're done:

1. Click "Destroy Workspace" in extension
2. Workspace transitions to `destroyed`
3. Containers stop and are removed
4. Runner releases all resources
5. **Billing ends**

Your work is gone, but code is safely in GitHub. If you need the workspace again, just create a new one.

## Features

### Docker Compose

Full docker-compose support means your entire dev stack runs together:

```yaml
# .devcontainer/docker-compose.yml
version: '3'
services:
  app:
    image: node:18
    volumes:
      - .:/workspace
    command: npm run dev
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: dev
  redis:
    image: redis:7
```

All services are reachable from inside the container as if they were local.

### Port Forwarding

Ports configured in`.devcontainer.json` are automatically forwarded:

```json
{
  "forwardPorts": [3000, 8000],
  "portsAttributes": {
    "3000": {"label": "App"},
    "8000": {"label": "API"}
  }
}
```

In VS Code, you'll see a "Ports" panel showing which ports are open where.

### File Syncing

Your local files sync into the container automatically. Changes are reflected instantly. You can:
- Edit locally in VS Code (feels local)
- Run commands in the container (have access to dependencies)
- See output in real-time

### Deterministic Environments

Every time you create a workspace from the same branch/tag, you get the same environment:
- Same base image
- Same docker-compose versions
- Same installed dependencies

No "it works on my machine but not in CI" surprises.

### Workspaces Are Snapshots

Workspaces are tied to a specific repo commit:

```
Create workspace for owner/repo @ abc123def456
```

Even if the repo's `main` branch advances, your workspace stays at that commit. You can:
- Compare against newer branches
- Create workspaces from different branches simultaneously
- Test multiple versions without conflicts

## Limits and Quotas

Organizations can set policies:

| Policy | Default | Example |
|--------|---------|---------|
| Max concurrent workspaces per user | 5 | Can't create more than 5 at once |
| Max concurrent workspaces per org | 50 | Org-wide limit |
| Workspace TTL (time-to-live) | 24 hours | Auto-destroy after 24 hours |
| Idle timeout | 30 minutes | Auto-pause after 30 min inactivity |
| Max workspace size | 100 GB | Can't create workspaces larger than 100 GB |

The policy is declarative; operators can adjust based on their infrastructure.

## Troubleshooting

### Workspace won't connect

1. Check VS Code extens

ion logs: `Output` → `Quickspaces`
2. Verify you have network access to the runner
3. Check if SSH port is open (ask your network team if on-prem)

### Containers won't start

1. Check `.devcontainer/devcontainer.json` for errors
2. View container logs on the runner (if you have access)
3. Ensure base image is accessible (not on private registry without credentials)

### Workspace is slow

1. Check runner resource utilization
2. Ensure docker-compose services aren't resource-heavy
3. Consider creating workspace on a different runner pool

---

**Next:** [Getting Started](./getting-started.md) or back to [README](../README.md)
