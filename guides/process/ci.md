# CI/CD Guide

> [Doctrine](../../README.md) > [Process](../README.md) > CI/CD

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

Recommended GitHub Actions[^1] configurations per language/framework.

## General Principles

1. **Fast feedback**: **MUST** lint before test, fail fast
2. **Cache dependencies**: **SHOULD** use language-specific caching
3. **Matrix testing**: **SHOULD** test across OS/versions where relevant
4. **Security scanning**: **MUST** run on every PR
5. **Artifact preservation**: **SHOULD** save coverage reports, binaries

## Python

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: astral-sh/setup-uv@v4[^3]
      - run: uv sync
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy src/

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: astral-sh/setup-uv@v4[^3]
      - run: uv sync
      - run: uv run pytest -n auto --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v4[^4]
        with:
          files: coverage.xml

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: astral-sh/setup-uv@v4[^3]
      - run: uv sync
      - run: uv run pip-audit[^5]
      - run: uv run semgrep --config=p/python --config=p/security-audit src/[^6]
```

### Django

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16[^7]
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4[^2]
      - uses: astral-sh/setup-uv@v4[^3]
      - run: uv sync
      - run: uv run ruff check .
      - run: uv run mypy .
      - run: uv run pytest --cov --reuse-db
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
```

## Go

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-go@v5[^8]
        with:
          go-version: '1.23'
      - uses: golangci/golangci-lint-action@v6[^9]
        with:
          version: latest

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-go@v5[^8]
        with:
          go-version: '1.23'
      - run: go test -race -coverprofile=coverage.out ./...
      - uses: codecov/codecov-action@v4[^4]
        with:
          files: coverage.out

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-go@v5[^8]
        with:
          go-version: '1.23'
      - run: go install golang.org/x/vuln/cmd/govulncheck@latest
      - run: govulncheck ./...[^10]
```

## Rust

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  CARGO_TERM_COLOR: always

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: dtolnay/rust-toolchain@stable[^11]
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2[^12]
      - run: cargo fmt -- --check
      - run: cargo clippy --all-targets --all-features -- -D warnings

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: dtolnay/rust-toolchain@stable[^11]
      - uses: Swatinem/rust-cache@v2[^12]
      - run: cargo test --all-features
      - run: cargo install cargo-tarpaulin[^13]
      - run: cargo tarpaulin --out xml
      - uses: codecov/codecov-action@v4[^4]

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: rustsec/audit-check@v2[^14]
```

## Ruby

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: ruby/setup-ruby@v1[^15]
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - run: bundle exec standardrb

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: ruby/setup-ruby@v1[^15]
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - run: bundle exec rspec
      - uses: codecov/codecov-action@v4[^4]

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: ruby/setup-ruby@v1[^15]
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - run: bundle exec bundler-audit check --update[^16]
```

### Rails

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16[^7]
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4[^2]
      - uses: ruby/setup-ruby@v1[^15]
        with:
          ruby-version: '3.3'
          bundler-cache: true
      - run: bundle exec standardrb
      - run: bundle exec brakeman --exit-on-warn --no-pager[^17]
      - run: bundle exec rails db:setup
        env:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
      - run: bundle exec rspec
```

## TypeScript / Node.js

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-node@v4[^18]
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npx biome ci .[^19]
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-node@v4[^18]
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npx vitest run --coverage[^20]
      - uses: codecov/codecov-action@v4[^4]

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-node@v4[^18]
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm audit --audit-level=high
```

## C# / .NET

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-dotnet@v4[^21]
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore /p:TreatWarningsAsErrors=true
      - run: dotnet format --verify-no-changes

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-dotnet@v4[^21]
        with:
          dotnet-version: '8.0.x'
      - run: dotnet test --collect:"XPlat Code Coverage"
      - uses: codecov/codecov-action@v4[^4]

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4[^2]
      - uses: actions/setup-dotnet@v4[^21]
        with:
          dotnet-version: '8.0.x'
      - run: dotnet list package --vulnerable --include-transitive
```

## Multi-Platform Matrix

For libraries/tools requiring cross-platform support:

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        # Add language version matrix as needed
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4[^2]
      # ... rest of steps
```

## Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4[^22]
        with:
          release-type: python  # or node, rust, go, etc.
```

## Dependabot

```yaml
# .github/dependabot.yml
version: 2[^23]
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      python-deps:
        patterns:
          - "*"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Branch Protection

Configure in repository settings:

- **MUST** require status checks to pass before merging
- **SHOULD** require branches to be up to date
- **SHOULD** require review from code owners
- **MUST NOT** allow bypassing the above settings

## See Also

- [Testing Guide](testing.md) - Testing strategy and best practices
- [Versioning Guide](versioning.md) - Semantic versioning and release automation
- [GitHub Templates](github-templates.md) - Issue and PR templates

## References

[^1]: [GitHub Actions Documentation](https://docs.github.com/en/actions)
[^2]: [actions/checkout](https://github.com/actions/checkout)
[^3]: [astral-sh/setup-uv](https://github.com/astral-sh/setup-uv)
[^4]: [codecov/codecov-action](https://github.com/codecov/codecov-action)
[^5]: [pip-audit](https://github.com/pypa/pip-audit)
[^6]: [Semgrep](https://semgrep.dev/docs/)
[^7]: [PostgreSQL Docker Official Images](https://hub.docker.com/_/postgres)
[^8]: [actions/setup-go](https://github.com/actions/setup-go)
[^9]: [golangci/golangci-lint-action](https://github.com/golangci/golangci-lint-action)
[^10]: [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
[^11]: [dtolnay/rust-toolchain](https://github.com/dtolnay/rust-toolchain)
[^12]: [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache)
[^13]: [cargo-tarpaulin](https://github.com/xd009642/tarpaulin)
[^14]: [rustsec/audit-check](https://github.com/rustsec/rustsec/tree/main/audit-check)
[^15]: [ruby/setup-ruby](https://github.com/ruby/setup-ruby)
[^16]: [bundler-audit](https://github.com/rubysec/bundler-audit)
[^17]: [Brakeman](https://brakemanscanner.org/)
[^18]: [actions/setup-node](https://github.com/actions/setup-node)
[^19]: [Biome](https://biomejs.dev/)
[^20]: [Vitest](https://vitest.dev/)
[^21]: [actions/setup-dotnet](https://github.com/actions/setup-dotnet)
[^22]: [googleapis/release-please-action](https://github.com/googleapis/release-please-action)
[^23]: [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)
