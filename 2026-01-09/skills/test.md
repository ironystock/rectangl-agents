---
description: Run tests using Docker (required - go-libsql doesn't compile on Windows)
argument-hint: [package or test name]
allowed-tools: Bash
---

Run tests in the dev container. This is REQUIRED because go-libsql does not compile natively on Windows.

If no argument provided, run all tests:
```bash
docker exec promptmark-dev-1 make test
```

If a specific test or package is provided via $ARGUMENTS:
- For a test name: `docker exec promptmark-dev-1 go test -v -run $ARGUMENTS ./...`
- For a package: `docker exec promptmark-dev-1 go test -v $ARGUMENTS`

Examples:
- `/test` - Run all tests
- `/test TestCreatePrompt` - Run specific test
- `/test ./internal/server/...` - Run package tests

Run the appropriate test command now and report the results.
