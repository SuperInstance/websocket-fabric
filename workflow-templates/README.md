# GitHub Actions CI/CD Workflow Templates

Collection of reusable GitHub Actions workflow templates for SuperInstance repositories.

## Templates

### Rust Projects

| File | Description |
|------|-------------|
| `ci-rust.yml` | CI workflow with formatting, linting, testing, and coverage |
| `publish-rust.yml` | Release workflow with multi-platform builds, crates.io publishing, and Docker |

### TypeScript/JavaScript Projects

| File | Description |
|------|-------------|
| `ci-typescript.yml` | CI workflow with type checking, linting, testing, and coverage |
| `publish-typescript.yml` | Release workflow with npm publishing and provenance |

### Python Projects

| File | Description |
|------|-------------|
| `ci-python.yml` | CI workflow with type checking (mypy), linting (ruff), testing (pytest) |
| `publish-python.yml` | Release workflow with PyPI publishing and wheel building |

### Next.js Projects

| File | Description |
|------|-------------|
| `ci-nextjs.yml` | CI workflow with build verification and Vercel preview deployments |

## Usage

### Quick Start

1. Copy the appropriate workflow files to your repository:
   ```bash
   mkdir -p .github/workflows
   cp workflow-templates/ci-rust.yml .github/workflows/ci.yml
   cp workflow-templates/publish-rust.yml .github/workflows/release.yml
   ```

2. Customize placeholders in the workflow files:
   - `PACKAGE_NAME`: Your package/binary name
   - `REPO_NAME`: Your repository name

3. Configure required secrets in your GitHub repository settings:
   - For Rust: `CRATES_IO_TOKEN`
   - For TypeScript: `NPM_TOKEN`
   - For Python: `PYPI_API_TOKEN`

### Placeholders to Replace

When copying templates to your repository, replace these placeholders:

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `PACKAGE_NAME` | The name of your package | `websocket-fabric` |
| `REPO_NAME` | Your GitHub repository name | `websocket-fabric` |

### Required Secrets

#### Rust (crates.io)
- `CRATES_IO_TOKEN`: API token from https://crates.io/me

#### TypeScript (npm)
- `NPM_TOKEN`: Automation token from https://www.npmjs.com/settings

#### Python (PyPI)
- `PYPI_API_TOKEN`: API token from https://pypi.org/manage/account/token/
- `TEST_PYPI_API_TOKEN`: (Optional) Token for TestPyPI

#### Vercel (Next.js)
- `VERCEL_TOKEN`: Personal access token from Vercel
- `VERCEL_ORG_ID`: Your Vercel organization ID
- `VERCEL_PROJECT_ID`: Your Vercel project ID

## Features by Template

### ci-rust.yml
- Multi-platform testing (Linux, macOS, Windows)
- Rust version testing (stable, nightly)
- Formatting checks (rustfmt)
- Linting (clippy)
- Code coverage (tarpaulin)
- MSRV (Minimum Supported Rust Version) check
- Security audit

### publish-rust.yml
- Multi-platform binary builds (x86_64, ARM64)
- Cross-compilation support
- GitHub Releases with assets
- crates.io publishing
- Docker image building

### ci-typescript.yml
- Multi-version Node.js testing (18, 20, 22)
- Multi-platform testing (Linux, macOS, Windows)
- Type checking (TypeScript)
- Linting (ESLint)
- Code coverage (Codecov)
- Security audit (pnpm audit)

### publish-typescript.yml
- npm publishing with provenance
- Semantic versioning support
- GitHub Releases
- Dry-run mode for testing

### ci-python.yml
- Multi-version Python testing (3.9, 3.10, 3.11, 3.12)
- Multi-platform testing (Linux, macOS, Windows)
- Type checking (mypy)
- Linting (ruff)
- Testing (pytest)
- Security scanning (bandit, safety)

### publish-python.yml
- Universal and platform-specific wheel building
- Source distribution building
- PyPI/TestPyPI publishing
- GitHub Releases

### ci-nextjs.yml
- Multi-version Node.js testing
- Type checking and linting
- Build verification
- Lighthouse CI (for PRs)
- Vercel preview deployments (for PRs)

## Repository Setup by Language

### Rust Repository

```bash
# Create workflows directory
mkdir -p .github/workflows

# Copy templates
cp workflow-templates/ci-rust.yml .github/workflows/ci.yml
cp workflow-templates/publish-rust.yml .github/workflows/release.yml

# Replace placeholders
sed -i 's/PACKAGE_NAME/your-package-name/g' .github/workflows/*.yml
sed -i 's/REPO_NAME/your-repo-name/g' .github/workflows/*.yml
```

### TypeScript/JavaScript Repository

```bash
# Create workflows directory
mkdir -p .github/workflows

# Copy templates
cp workflow-templates/ci-typescript.yml .github/workflows/ci.yml
cp workflow-templates/publish-typescript.yml .github/workflows/release.yml

# Replace placeholders
sed -i 's/PACKAGE_NAME/your-package-name/g' .github/workflows/*.yml
sed -i 's/REPO_NAME/your-repo-name/g' .github/workflows/*.yml
```

### Python Repository

```bash
# Create workflows directory
mkdir -p .github/workflows

# Copy templates
cp workflow-templates/ci-python.yml .github/workflows/ci.yml
cp workflow-templates/publish-python.yml .github/workflows/release.yml

# Replace placeholders
sed -i 's/PACKAGE_NAME/your-package-name/g' .github/workflows/*.yml
sed -i 's/REPO_NAME/your-repo-name/g' .github/workflows/*.yml
```

### Next.js Application

```bash
# Create workflows directory
mkdir -p .github/workflows

# Copy template
cp workflow-templates/ci-nextjs.yml .github/workflows/ci.yml
```

## Triggering Workflows

### CI Workflow
- Runs on every push to `main` or `develop` branches
- Runs on every pull request targeting `main` or `develop`

### Release Workflow
- Runs on version tags (e.g., `v1.0.0`)
- Can be manually triggered via GitHub Actions UI with custom version

## Badge Integration

Add a CI badge to your README.md:

```markdown
[![CI](https://github.com/SuperInstance/REPO_NAME/actions/workflows/ci.yml/badge.svg)](https://github.com/SuperInstance/REPO_NAME/actions/workflows/ci.yml)
```

## License

MIT
