# Vulnerability Scanning with Grype

This guide covers scanning the  Maven project for known vulnerabilities using [Grype](https://github.com/anchore/grype), an open-source vulnerability scanner by Anchore.

## Prerequisites

- Docker installed and running (recommended — no local install needed), **or**
- Grype installed natively (optional alternative, see below)
- JDK 17 and Maven 3.9+ (to build the project before scanning)

## Option A: Run via Docker (recommended, no install)

### 1. Build the project

```bash
mvn clean package
```

This produces the jar under `target/` and resolves all Maven dependencies needed for an accurate scan.

### 2. Run the scan

From the project root:

```bash
docker run --rm -v "$(pwd)":/app anchore/grype dir:/app
```

- `-v "$(pwd)":/app` mounts the current project directory into the container
- `dir:/app` tells Grype to scan that directory (source, `pom.xml`, and built artifacts)
- `--rm` removes the container after the scan finishes

**First run note:** Grype downloads its vulnerability database (~a few hundred MB) on first use. This is a one-time cost — subsequent scans are much faster.

### Caching the vulnerability database (recommended)

By default, `--rm` removes the container after each run, so the database re-downloads every time. To persist the cache to a folder in your project and skip re-downloading, mount a host folder to Grype's cache path:

```bash
docker run --rm \
  -v "$(pwd)":/app \
  -v "$(pwd)/.grype-cache":/root/.cache/grype \
  anchore/grype dir:/app
```

This creates a `.grype-cache/` folder in your project on first run and reuses it on every scan after that. Add `.grype-cache/` to `.gitignore` so it isn't committed.

### 3. Review the output

By default, Grype prints a table sorted by severity (Critical → High → Medium → Low → Negligible), including the affected package, installed version, fixed version (if available), and CVE ID.

## Option B: Install Grype natively (no Docker)

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/anchore-engine/main/scripts/install_grype.sh | sh -s -- -b /usr/local/bin
```

Then scan with:

```bash
grype dir:.
```

## Common variations

**Save results as JSON (for reports or CI):**
```bash
docker run --rm -v "$(pwd)":/app anchore/grype dir:/app -o json > grype-report.json
```

**Save results as a table file:**
```bash
docker run --rm -v "$(pwd)":/app anchore/grype dir:/app -o table > grype-report.txt
```

**Fail the command on High/Critical findings (useful in CI pipelines):**
```bash
docker run --rm -v "$(pwd)":/app anchore/grype dir:/app --fail-on high
```
Exits non-zero if any vulnerability meets or exceeds the given severity — use this as a build gate.

**Scan a built Docker image instead of source:**
```bash
docker build -t java-application-template:latest -f docker/Dockerfile .
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock anchore/grype java-application-template:latest
```

## Typical workflow for this project

```bash
# 1. Build
mvn clean package

# 2. Scan source and dependencies (with cached DB)
docker run --rm \
  -v "$(pwd)":/app \
  -v "$(pwd)/.grype-cache":/root/.cache/grype \
  anchore/grype dir:/app

# 3. (Optional) Build and scan the Docker image
docker build -t java-application-template:latest -f docker/Dockerfile .
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock anchore/grype java-application-template:latest
```

## Interpreting results

- **Fixed version available** → upgrade the dependency in `pom.xml` and re-scan.
- **No fix available** → assess actual exposure (is the vulnerable code path reachable?), document as an accepted risk if not exploitable, and monitor for future patches.
- Re-run the scan after every dependency change or before each release to catch regressions.

## References

- Grype GitHub: https://github.com/anchore/grype
