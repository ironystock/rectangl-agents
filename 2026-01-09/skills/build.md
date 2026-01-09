---
description: Build the project using Docker (required - go-libsql doesn't compile on Windows)
allowed-tools: Bash
---

Build the project in the dev container. This is REQUIRED because go-libsql does not compile natively on Windows.

```bash
docker exec promptmark-dev-1 make build
```

If the container isn't running, start it first:
```bash
docker compose up -d dev
```

Run the build now and report the results.
