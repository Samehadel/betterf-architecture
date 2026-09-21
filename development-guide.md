# Development Guide

## Getting Started

### Prerequisites

- **[BACKEND_RUNTIME]** (required)
- **[BUILD_TOOL]** (via wrapper — no local install needed)
- **[DATABASE_SYSTEM]**
- **Git**
- **IDE**: [RECOMMENDED_IDE_LIST]

### Initial Setup

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd <project-name>
   ```

2. **Install dependencies**
   ```bash
   [DEPENDENCY_INSTALL_COMMAND]
   ```

3. **Set up [DATABASE_SYSTEM]**
   ```bash
   # [DATABASE_SETUP_COMMANDS]
   # Customize based on your database and local environment
   ```

4. **Configure environment**
   Create `[CONFIG_FILE_NAME]`:
   ```
   [CONFIG_TEMPLATE]
   # Customize with your local database credentials and settings
   ```

5. **Run the application**
   ```bash
   [RUN_APPLICATION_COMMAND]
   ```

6. **Verify setup**
   - API Docs: http://localhost:[PORT]/[API_DOCS_PATH]
   - Health Check: http://localhost:[PORT]/[HEALTH_CHECK_PATH]

---

## Development Commands

### Build & Run

```bash
# Build the project (compile, run tests, package)
[BUILD_COMMAND]

# Run the application
[RUN_COMMAND]

# Run with specific profile/environment
[RUN_WITH_ENV_COMMAND]

# Clean build artifacts
[CLEAN_COMMAND]

# Build without tests (local troubleshooting only — never for PR validation)
[BUILD_NO_TEST_COMMAND]
```

### Testing

```bash
# Run all tests
[TEST_ALL_COMMAND]

# Run a single test class
[TEST_SINGLE_CLASS_COMMAND]

# Run a single test method
[TEST_SINGLE_METHOD_COMMAND]

# Run integration tests only
[TEST_INTEGRATION_COMMAND]

# Run with coverage report
[TEST_WITH_COVERAGE_COMMAND]

# View coverage report
[VIEW_COVERAGE_COMMAND]
```

### Gradle Tasks

```bash
# List all available tasks
./gradlew tasks

# Check dependencies
./gradlew dependencies

# Refresh generated API docs
./gradlew openApiGenerate

# Format code (Spotless)
./gradlew spotlessApply
```

---

## Code Review Checklist

Before creating a pull request, ensure:

- [ ] Code follows naming conventions defined in [Backend Architecture](./backend-architecture.md)
- [ ] Unit tests written and passing (85%+ coverage)
- [ ] Integration tests for new API endpoints
- [ ] API documentation and shared conventions updated before any controller or transport-model change
- [ ] Liquibase changeset created for every schema change, with rollback instructions
- [ ] Exception handling uses the centralized global handler — no ad-hoc try/catch in controllers
- [ ] Response Views and request models use the documented field naming conventions
- [ ] No hardcoded values or credentials
- [ ] No direct SQL queries — use Spring Data JPA
- [ ] Git history is clean (no merge commits in PR)


---

## Git Workflow

See `CLAUDE.md` for branch naming conventions, commit message format, and development workflow.
