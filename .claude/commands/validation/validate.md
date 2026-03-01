Run comprehensive validation of the project.

Execute the following commands in sequence and report results:

## 1. Backend Linting

```bash
cd backend && uv run ruff check .
```

**Expected:** "All checks passed!" or no output (clean)

## 2. Backend Tests

```bash
cd backend && uv run pytest -v
```

**Expected:** All tests pass, execution time < 5 seconds

## 3. Backend Tests with Coverage

```bash
cd backend && uv run pytest --cov=app --cov-report=term-missing
```

**Expected:** Coverage >= 80% (configured in pyproject.toml)

## 4. Frontend Build

```bash
cd frontend && npm run build
```

**Expected:** Build completes successfully, outputs to `dist/` directory

## 5. Local Server Validation (Optional)

If backend is not already running, start it:

```bash
cd backend && uv run uvicorn app.main:app --port 8000 &
```

Wait 2 seconds for startup, then test:

```bash
# Check API docs
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://localhost:8000/docs
```

Stop the server if started:

```bash
# Linux/Mac
lsof -ti:8000 | xargs kill -9 2>/dev/null || true
```

## 6. E2E Tests (Optional)

```bash
npx playwright test
```

**Expected:** All Playwright tests pass

## 7. Summary Report

After all validations complete, provide a summary report with:

- Linting status
- Tests passed/failed
- Coverage percentage
- Frontend build status
- Any errors or warnings encountered
- Overall health assessment (PASS/FAIL)

**Format the report clearly with sections and status indicators**
