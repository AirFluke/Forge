# Forge CI/CD Platform

Forge is a CI/CD platform with an integrated artifact registry, built for highly reliable and isolated builds.

## Public URL
The platform is accessible at: `http://<YOUR_VPS_IP>`

## Pipeline YAML Schema

```yaml
name: build-lib-http          # Required: Pipeline name
version: 1.0.0                # Required: Semver version of the pipeline run
dependencies:                 # Optional: Dependencies pulled before any job runs
  - name: lib-core
    version: "^1.0.0"         # Caret constraint resolves to highest >=1.0.0 <2.0.0
jobs:                         # Required: Map of jobs to execute
  build:                      # Job name
    runtime: alpine:3.18      # Optional: Docker image for execution (default: alpine:3.18)
    resources:                # Optional: Resource limits
      cpu: 1.0                # Fractional CPUs
      memory: 512Mi           # Memory limit (bytes or suffixes like Ki, Mi, Gi)
    needs: []                 # Optional: List of jobs that must complete successfully first
    steps:                    # Required: List of steps to execute sequentially
      - name: test            # Required: Step name
        run: "sh ./test.sh"   # Required: Shell command
      - name: package
        run: "tar czf out.tar.gz src/"
artifacts:                    # Optional: Artifacts to auto-publish if all jobs succeed
  - name: lib-http            # Required: Artifact name
    version: 1.0.0            # Required: Semver version
    path: ./out.tar.gz        # Required: Path to artifact within the shared workspace
```

## System Architecture

### DAG Scheduler
The CI engine uses Kahn's algorithm for topological sorting to execute jobs in the correct dependency order. A 3-color DFS algorithm detects circular dependencies before the pipeline runs, instantly rejecting invalid DAGs. Independent jobs in the same topological level are executed concurrently up to the global `max_concurrency` limit using Python's `asyncio.Semaphore`. If a job fails, all transitive dependents are marked as skipped rather than failed.

### Isolation Mechanism
Jobs run in strictly isolated Docker containers configured via `--security-opt no-new-privileges` and `--cap-drop ALL`. Resource exhaustion is contained by strict `--memory` limits (with swap disabled) and `--cpus` limits. Network isolation is enforced by attaching the container to a dedicated Docker network that only permits routing to the artifact registry. Host filesystem access is blocked; jobs execute in a temporary `tmpfs` volume that is deleted after the run. Fork bombs are mitigated via `--pids-limit 200`.

### Storage Layer
The artifact registry stores files using Content-Addressable Storage (CAS). Blobs are saved to disk under their SHA-256 hash (`blobs/{sha256[:2]}/{sha256}`). SQLite manages metadata in WAL mode to handle concurrent reads/writes safely. The registry recomputes the SHA-256 hash of all uploads server-side and rejects any request where the declared checksum does not match the actual bytes.

### Dependency Resolver
The custom semver resolver builds a dependency graph via Breadth-First Search (BFS) starting from the root dependencies. It parses complex constraints (`^`, `~`, exact, comparators), queries the registry, and finds the intersection of all constraints for each package. **Determinism is guaranteed** because available versions are semantically sorted in descending order, and the resolver systematically picks the highest version satisfying all constraints. For an unchanged registry state, this identical logic yields an identical lockfile byte-for-byte.

### Log Streaming
Live log streaming uses Server-Sent Events (SSE) with `text/event-stream`. Instead of buffering in memory, the engine streams JSON-lines log files incrementally from disk. Chunked reads ensure memory usage stays flat regardless of log size (easily handling 50MB logs). Clients can use `Last-Event-ID` to resume a broken connection without losing messages.

### Concurrency & Race Conditions
If two pipelines race to publish the same `(name, version)`, SQLite handles the conflict via a `UNIQUE(name, version)` constraint. The first transaction commits successfully, while the second transaction encounters a SQLite `IntegrityError` which translates into an HTTP `409 Conflict`. Immutability is strictly maintained.

## Slack Alerts
![Slack Alerts Screenshot](./slack_alerts.png)
*(Ensure your Slack webhook URL is set in `config.yml` or `compose.yml` to receive pipeline status, cycle, and integrity alerts).*

## Setup Instructions

1. **Clone the Repository**
   ```bash
   git clone <repo-url> forge-platform
   cd forge-platform
   ```

2. **Configure Environment**
   Set your Slack Webhook URL in `.env` (or directly in `compose.yml` / `config.yml`):
   ```bash
   export FORGE_SLACK_WEBHOOK="https://your-slack-workspace.slack.com/webhook/..."
   ```

3. **Deploy with Docker Compose**
   ```bash
   docker compose up -d --build
   ```

4. **Install CLI Tool**
   ```bash
   cd cli
   pip install .
   ```

5. **Create First Auth Token**
   Since the registry is fresh, execute the token creation command directly in the engine container:
   ```bash
   # Generate an admin token for yourself
   docker exec -it forge-engine python -c "import sys; sys.path.insert(0, '/app/registry'); from auth import create_token; print(create_token('/data/registry/auth.db', 'admin'))"
   ```
   *Copy the outputted raw token.*

6. **Login via CLI**
   ```bash
   # Assuming the platform is running on localhost or your VPS IP
   forge login http://localhost
   # Paste the token when prompted
   ```

You are now ready to run pipelines!
