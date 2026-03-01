# PRD: arbox_auto_register

## 1. Executive Summary

`arbox_auto_register` is a lightweight Python automation bot that automatically registers the user for fitness classes on the [Arbox](https://www.arboxapp.com/) platform. Arbox is a gym/CrossFit box management system used by thousands of fitness clubs worldwide. Popular class slots fill up within minutes of registration opening — this bot solves that problem by acting the instant a registration window opens.

The bot runs as a scheduled GitHub Actions workflow (free tier), authenticates with the user's Arbox account via email + password, fetches the upcoming class schedule, filters classes by user-defined day/time windows and class type, and registers automatically. Results are surfaced through GitHub Actions logs and optional email notifications.

**MVP goal**: A zero-cost, zero-UI Python script that reliably registers for preferred Arbox classes without any manual intervention, running entirely on GitHub Actions free compute.

---

## 2. Mission

> Eliminate the manual effort and timing stress of booking Arbox fitness classes by automating registration the moment slots open.

**Core Principles:**
1. **Zero cost** — runs entirely on GitHub Actions free tier; no servers, no databases, no hosting fees
2. **Zero UI** — configuration is a YAML file; no frontend, no dashboard
3. **Reliable over clever** — conservative retry logic, clear error messages, fail loudly via logs
4. **Credentials-safe** — all secrets stored in GitHub Secrets, never in code or config files
5. **Idempotent** — running the bot twice for the same class is safe; it detects existing registrations

---

## 3. Target Users

**Primary persona: Personal fitness enthusiast**
- Attends a CrossFit box or gym that uses Arbox
- Struggles to manually book popular classes before they fill up
- Comfortable with GitHub (can add secrets, edit YAML)
- Non-developer or light developer — doesn't need to touch Python code
- Runs no servers at home; wants a "set and forget" solution

**Pain points:**
- Classes fill up within 2–5 minutes of registration opening
- Must remember to log in and book at exactly the right time
- Missing a class means attending a less preferred time slot
- No built-in Arbox feature for automatic or recurring registration

---

## 4. MVP Scope

### Core Functionality

| Feature | Scope |
|---------|-------|
| ✅ Email + password authentication to Arbox | In scope |
| ✅ Session token management (reuse/refresh) | In scope |
| ✅ Fetch upcoming class schedule from Arbox API | In scope |
| ✅ Filter classes by day-of-week + time window | In scope |
| ✅ Filter classes by class type/name keywords | In scope |
| ✅ Register for matching classes | In scope |
| ✅ Detect and skip already-registered classes (idempotent) | In scope |
| ✅ Log results to stdout (visible in GitHub Actions) | In scope |
| ✅ GitHub Actions workflow with cron schedule | In scope |
| ✅ YAML config file for user preferences | In scope |
| ✅ GitHub Secrets for credentials | In scope |
| ✅ Dry-run mode (plan without registering) | In scope |
| ❌ Multi-user support | Out of scope |
| ❌ Web UI / dashboard | Out of scope |
| ❌ Push notifications (Telegram, Slack) | Out of scope (future) |
| ❌ Waitlist handling | Out of scope (future) |
| ❌ Calendar sync (Google Calendar, iCal) | Out of scope (future) |

### Technical

| Feature | Scope |
|---------|-------|
| ✅ Python 3.11+ with uv package manager | In scope |
| ✅ httpx for HTTP requests | In scope |
| ✅ Playwright fallback for JavaScript-rendered pages | In scope (if API insufficient) |
| ✅ pytest unit tests for filtering logic | In scope |
| ✅ GitHub Actions CI workflow | In scope |
| ❌ Docker containerization | Out of scope |
| ❌ SQLite or any persistent database | Out of scope |
| ❌ REST API wrapper around the bot | Out of scope |

---

## 5. User Stories

**US-1**: As a gym member, I want the bot to automatically book my Monday 07:00 CrossFit class every week, so I don't have to set an alarm to compete with other members for spots.
> Config: `day: Monday, time: "07:00", class_types: ["CrossFit"]`

**US-2**: As a user, I want to specify multiple preferred class slots in one config file, so the bot handles my entire weekly schedule in one run.
> Config lists multiple `preferences` entries; bot registers for all matching classes in the lookahead window.

**US-3**: As a user, I want the bot to skip a class if I'm already registered, so I don't see errors or double-bookings when the workflow runs multiple times.
> Bot checks existing registrations before attempting to register.

**US-4**: As a user, I want to run a dry-run to see which classes the bot would register for, without actually registering, so I can validate my config before going live.
> `--dry-run` flag prints matched classes and exits.

**US-5**: As a user, I want clear logs in GitHub Actions showing what was found, what was registered, and what was skipped, so I can verify the bot is working correctly.
> Structured output: `[FOUND] Monday 07:00 CrossFit`, `[REGISTERED]`, `[SKIPPED] already registered`.

**US-6**: As a user, I want my Arbox credentials stored securely as GitHub Secrets, so they are never exposed in the repository or logs.
> Script reads `ARBOX_EMAIL` and `ARBOX_PASSWORD` from environment variables only.

**US-7**: As a user, I want the GitHub Actions cron to trigger at the exact time registration opens (configurable), so the bot competes optimally for limited spots.
> Cron expression configurable in the workflow YAML.

**US-8** (technical): As a developer, I want the Arbox API interaction isolated in a thin client module, so it's easy to update if the API changes.
> `src/arbox_client.py` wraps all HTTP calls; business logic is separate.

---

## 6. Core Architecture & Patterns

### High-Level Architecture

```
GitHub Actions Cron
        │
        ▼
  run.py (entrypoint)
        │
        ├── config.py          ← loads config.yaml + env vars
        ├── arbox_client.py    ← Arbox API HTTP client (auth, schedule, register)
        ├── scheduler.py       ← filters classes against user preferences
        └── logger.py          ← structured stdout logging
```

### Directory Structure

```
arbox_auto_register/
├── src/
│   ├── __init__.py
│   ├── run.py              # CLI entrypoint
│   ├── arbox_client.py     # Arbox API client (auth, schedule, register)
│   ├── scheduler.py        # Class filtering and matching logic
│   ├── config.py           # Config loading and validation
│   └── logger.py           # Structured logging helpers
├── tests/
│   ├── conftest.py
│   ├── test_scheduler.py   # Unit tests for filtering logic
│   └── test_config.py      # Unit tests for config validation
├── config.yaml             # User preferences (committed, no secrets)
├── .github/
│   └── workflows/
│       └── register.yml    # GitHub Actions cron workflow
├── pyproject.toml          # uv-managed dependencies
└── README.md
```

### Key Design Patterns

- **Config-driven**: All user preferences in `config.yaml`; credentials only via env vars
- **Thin API client**: `arbox_client.py` wraps raw HTTP; everything else works with plain Python dicts/dataclasses
- **Pure filtering logic**: `scheduler.py` functions are pure (no side effects), making them easily unit-testable
- **Fail loudly**: Non-zero exit code on any registration failure; GitHub Actions marks run as failed
- **Idempotency**: Check registration status before attempting to register

---

## 7. Features

### 7.1 Authentication (`arbox_client.py`)

- POST to Arbox login endpoint with email + password
- Store session token in memory for the duration of the run
- Re-authenticate if token expires mid-run
- Mask credentials in all log output

### 7.2 Schedule Fetching

- Fetch classes for the next N days (configurable, default: 7)
- Parse class list: id, name, date, time, coach, capacity, registered_count, is_registered
- Return as a list of dataclass objects

### 7.3 Class Filtering (`scheduler.py`)

Match classes against user preferences. A class matches if ALL conditions pass:

| Condition | Config field | Example |
|-----------|-------------|---------|
| Day of week | `day` | `"Monday"` |
| Time window | `time`, `time_range` | `"07:00"` or `"07:00-08:00"` |
| Class type keyword | `class_types` | `["CrossFit", "Open Gym"]` |

Multiple preference entries are OR'd (register for any match).

### 7.4 Registration

- For each matched, unregistered class: POST registration request
- On success: log `[REGISTERED]` with class details
- On "already registered": log `[SKIPPED]`
- On "class full": log `[FULL]` and continue (no error)
- On network/API error: log `[ERROR]` and exit with non-zero status

### 7.5 Dry-Run Mode

- `--dry-run` flag: fetch + filter, but skip registration POST
- Print matched classes with `[DRY-RUN]` prefix
- Exit 0 regardless of matches

### 7.6 GitHub Actions Workflow

- Cron trigger: configurable schedule (e.g., daily at registration-open time)
- Manual trigger: `workflow_dispatch` for on-demand runs
- Matrix: single job, Python 3.11
- Secrets: `ARBOX_EMAIL`, `ARBOX_PASSWORD` injected as env vars
- Artifact: none (logs are sufficient)

---

## 8. Technology Stack

### Core

| Component | Technology | Reason |
|-----------|-----------|--------|
| Language | Python 3.11+ | Mature ecosystem, Arbox API reverse-engineering ease |
| Package manager | uv | Fast, modern, lockfile support |
| HTTP client | httpx | Async-capable, clean API, better than requests |
| Config | PyYAML | Human-friendly config format |
| Config validation | Pydantic v2 | Type-safe config parsing with clear errors |
| Logging | structlog | Structured output; easy to parse in CI logs |
| Testing | pytest | Standard Python testing |
| CI/CD | GitHub Actions | Free compute, cron support, Secrets integration |

### Optional (if Arbox uses JS rendering)

| Component | Technology | Reason |
|-----------|-----------|--------|
| Browser automation | Playwright (Python) | Fallback if Arbox API requires browser session |

### Dev Tools

| Tool | Purpose |
|------|---------|
| Ruff | Linting + formatting |
| pytest-cov | Test coverage |
| python-dotenv | Load `.env` for local development |

---

## 9. Security & Configuration

### Credentials

- `ARBOX_EMAIL` and `ARBOX_PASSWORD` stored **only** as GitHub Secrets
- Locally: stored in `.env` file (git-ignored)
- Never logged, never written to files
- Script exits with error if env vars are missing

### `config.yaml` (committed to repo — no secrets)

```yaml
# How many days ahead to look for classes
lookahead_days: 7

# Arbox club/location ID (find in your Arbox URL)
club_id: "12345"

# Classes to register for
preferences:
  - day: "Monday"
    time: "07:00"
    class_types: ["CrossFit"]
  - day: "Wednesday"
    time: "07:00"
    class_types: ["CrossFit"]
  - day: "Friday"
    time: "06:00"
    class_types: ["CrossFit", "Olympic Lifting"]
  - day: "Saturday"
    time_range: "09:00-10:00"
    class_types: ["Open Gym"]
```

### `.env` (local dev, git-ignored)

```
ARBOX_EMAIL=user@example.com
ARBOX_PASSWORD=yourpassword
```

### GitHub Actions Secrets

```
ARBOX_EMAIL    → Settings → Secrets → Actions
ARBOX_PASSWORD → Settings → Secrets → Actions
```

### Security Scope

| Item | Status |
|------|--------|
| ✅ Credentials via env vars only | In scope |
| ✅ .env git-ignored | In scope |
| ✅ Credential masking in logs | In scope |
| ❌ OAuth / token refresh flows | Out of scope (MVP uses session cookie) |
| ❌ Encrypted config | Out of scope |

---

## 10. API Specification

### Arbox API (Reverse-Engineered)

The Arbox platform exposes internal REST endpoints used by its mobile/web app. These will be discovered by intercepting network traffic using proxy software during implementation.

**API discovery approach (in order of preference):**

1. **Proxy software (recommended)** — Route the Arbox mobile app through a MITM HTTP proxy to capture all requests and responses in full detail. Recommended tools:
   - [Proxyman](https://proxyman.io/) (macOS, free tier sufficient) — GUI, easy SSL certificate setup
   - [mitmproxy](https://mitmproxy.org/) (cross-platform, open source) — CLI + web UI
   - [Charles Proxy](https://www.charlesproxy.com/) (macOS/Windows) — mature, paid
   - Setup: install proxy cert on iOS/Android device or simulator, route traffic through the proxy, then perform actions in the Arbox app to capture the exact requests

2. **Browser DevTools** — Fallback for endpoints accessible via the Arbox web app; less complete than mobile app traffic but easier to set up

**Expected endpoints (to be confirmed):**

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/api/v1/auth/login` | Authenticate, receive token |
| `GET` | `/api/v1/schedule` | Fetch class schedule |
| `POST` | `/api/v1/schedule/{class_id}/register` | Register for a class |
| `GET` | `/api/v1/schedule/{class_id}/registrations` | Check registration status |

**Auth header (expected):**
```
Authorization: Bearer <token>
```

**Login request:**
```json
{
  "email": "user@example.com",
  "password": "password"
}
```

**Login response:**
```json
{
  "token": "eyJ...",
  "user_id": 12345
}
```

**Class object (expected):**
```json
{
  "id": "abc123",
  "name": "CrossFit",
  "date": "2026-02-23",
  "start_time": "07:00",
  "end_time": "08:00",
  "capacity": 15,
  "registered_count": 12,
  "is_registered": false,
  "coach": "John D."
}
```

> **Note**: Actual endpoints and payloads must be confirmed by inspecting Arbox network traffic. The `arbox_client.py` module will encapsulate all API details.

---

## 11. Success Criteria

### MVP Success Definition

The MVP succeeds when the bot reliably registers for at least one configured class on GitHub Actions without any manual intervention.

### Functional Requirements

- ✅ Bot authenticates with Arbox using env var credentials
- ✅ Bot fetches class schedule for the next 7 days
- ✅ Bot correctly filters classes by day, time, and class type
- ✅ Bot registers for all matching unregistered classes
- ✅ Bot skips already-registered classes without error
- ✅ Dry-run mode works and never mutates state
- ✅ GitHub Actions cron workflow triggers and completes successfully
- ✅ Zero credentials visible in logs or code
- ✅ Bot exits non-zero on registration failure (alerts GitHub Actions)

### Quality Indicators

- ✅ `pytest` passes with >80% coverage on filtering/scheduling logic
- ✅ `ruff` passes with no linting errors
- ✅ Workflow runs complete in under 60 seconds
- ✅ Single `config.yaml` change is sufficient to update preferences

### User Experience Goals

- A non-developer can configure and activate the bot by following the README in under 15 minutes
- Logs are human-readable and clearly indicate what happened

---

## 12. Implementation Phases

### Phase 1: API Discovery & Authentication (Day 1)

**Goal**: Understand the Arbox API and authenticate successfully.

**Deliverables:**
- ✅ Intercept Arbox mobile app traffic via proxy software (Proxyman/mitmproxy) to map real endpoints, auth flow, and request/response shapes
- ✅ Cross-reference with browser DevTools on the Arbox web app for completeness
- ✅ Document actual API endpoints, headers, and payloads in `docs/api-notes.md`
- ✅ Implement `arbox_client.py` with `login()` and `get_schedule()` methods
- ✅ Manual test: print raw schedule JSON to stdout

**Validation**: Running `python -m src.run --dry-run` prints today's class list.

---

### Phase 2: Filtering Logic & Config (Day 2)

**Goal**: Config-driven class matching works correctly.

**Deliverables:**
- ✅ Implement `config.py` with Pydantic validation of `config.yaml`
- ✅ Implement `scheduler.py` with `filter_classes()` function
- ✅ Unit tests for all filter combinations in `tests/test_scheduler.py`
- ✅ Dry-run mode shows matched classes from real schedule

**Validation**: `pytest tests/` passes; dry-run shows expected classes.

---

### Phase 3: Registration & Error Handling (Day 3)

**Goal**: Bot registers for classes and handles all error cases gracefully.

**Deliverables:**
- ✅ Implement `arbox_client.py` `register_class()` method
- ✅ Handle: success, already-registered, class full, API error
- ✅ Implement structured logging via `structlog`
- ✅ Integration test against real Arbox account (manual verification)
- ✅ Full run: bot registers for a real class end-to-end

**Validation**: Bot registers for one class successfully; logs are clear.

---

### Phase 4: GitHub Actions & Hardening (Day 4)

**Goal**: Zero-touch automation running on schedule in GitHub Actions.

**Deliverables:**
- ✅ GitHub Actions workflow `register.yml` with cron + manual trigger
- ✅ Secrets configured in GitHub repository settings
- ✅ `.env` added to `.gitignore`
- ✅ README with setup instructions (fork → set secrets → edit config)
- ✅ Ruff linting passes in CI
- ✅ pytest runs in CI

**Validation**: Workflow runs on schedule and registers for a class without any local intervention.

---

## 13. Future Considerations

- **Waitlist support**: Detect full classes and join waitlist; re-trigger if spot opens
- **Cancellation automation**: Cancel a registration if plans change (detect via calendar)
- **Notifications**: Telegram bot or email summary after each run
- **Google Calendar sync**: Add registered classes to personal calendar
- **Multi-club support**: Handle accounts that span multiple Arbox locations
- **Retry with backoff**: Retry registration if Arbox is slow at peak times (registration-open moment)
- **Class preference scoring**: Rank preferences so if multiple slots open, the most preferred wins
- **Dashboard**: Simple read-only web page showing upcoming registered classes

---

## 14. Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| Arbox changes its internal API | Medium | Isolate all API calls in `arbox_client.py`; update one file to adapt |
| Arbox adds bot detection / rate limiting | Medium | Add polite delays between requests; use realistic headers; avoid hammering |
| GitHub Actions cron delay (up to 15 min) | High | Set cron to run 5–10 min before registration opens; most gyms tolerate slight latency |
| Credentials leaked in logs | Low | structlog masks env vars; never interpolate credentials into log messages |
| Arbox requires JS rendering (no plain API) | Low-Medium | Playwright fallback already planned in tech stack |

---

## 15. Appendix

### Related Documents

- [CLAUDE.md](../CLAUDE.md) — Development conventions and commands
- [context.md](../context.md) — Session history and setup notes
- `.claude/reference/fastapi-best-practices.md` — API patterns (if backend is added later)
- `.claude/reference/testing-and-logging.md` — pytest and structlog patterns

### Key External Resources

- Arbox web app: https://app.arboxapp.com
- Arbox API base: to be discovered via DevTools
- GitHub Actions cron syntax: https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule
- uv documentation: https://docs.astral.sh/uv/

### Environment Variables

| Variable | Required | Description |
|----------|---------|-------------|
| `ARBOX_EMAIL` | Yes | Arbox account email |
| `ARBOX_PASSWORD` | Yes | Arbox account password |
| `DRY_RUN` | No | Set to `1` to enable dry-run mode via env var |

### GitHub Actions Workflow Sketch

```yaml
name: Auto-Register Arbox Classes

on:
  schedule:
    - cron: '0 4 * * 1,3,5'  # Mon/Wed/Fri at 04:00 UTC — adjust to your registration-open time
  workflow_dispatch:           # Manual trigger

jobs:
  register:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
        with:
          python-version: '3.11'
      - run: uv sync
      - run: uv run python -m src.run
        env:
          ARBOX_EMAIL: ${{ secrets.ARBOX_EMAIL }}
          ARBOX_PASSWORD: ${{ secrets.ARBOX_PASSWORD }}
```
