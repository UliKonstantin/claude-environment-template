# Feature: AWS Lambda + Trigger-Specific Registration

Validate documentation and codebase patterns before implementing.

## Feature Description

1. **AWS Lambda wrapper** — Run the Arbox registration bot on Lambda, invoked by EventBridge cron at precise times (17:00 UTC = 19:00 Israel).
2. **Single config** — `config.yaml` remains the single point of configuration; no secrets in config.
3. **Trigger-specific preferences** — Each EventBridge rule passes a `trigger` ID. Only preferences whose `triggers` include that ID (or have no triggers) are used. Example: Saturday run → only וויטליפטינג; Tuesday run → only ג'ימנסטיקס.

## User Story

As a gym member, I want the bot to run on AWS Lambda with precise scheduling, and I want different runs to register only for the relevant classes (e.g. Saturday only weightlifting, Tuesday only gymnastics), so signup windows are hit exactly and unnecessary 425 retries are avoided.

## Problem Statement

- GitHub Actions cron has 0–15 min delay; no guarantee of 19:00.
- Every run currently tries all preferences; Saturday run hits 425 for ג'ימנסטיקס, Tuesday for וויטליפטינג.

## Solution Statement

- Add Lambda handler that loads config, filters preferences by `trigger`, runs existing core logic.
- Extend `config.yaml` with `schedules` (trigger IDs + cron) and optional `triggers` on each preference.
- Add deploy script to build zip, create Lambda, create EventBridge rules with Constant JSON `{"trigger": "saturday_19"}` etc.
- Credentials via Lambda env vars (ARBOX_EMAIL, ARBOX_PASSWORD).

## Feature Metadata

**Feature Type**: New Capability + Enhancement
**Complexity**: Medium
**Primary Systems Affected**: config.py, run.py, scheduler, new lambda_handler.py, deploy/
**Dependencies**: AWS CLI, boto3 (deploy script only; not in runtime)

---

## CONTEXT REFERENCES

### Relevant Codebase Files — READ THESE FIRST

- `src/config.py` — Settings, ClassPreference; add `triggers: list[str] | None` and `schedules` parsing
- `src/run.py` — main() flow; needs to accept optional `preferences_filter` or run() accepts trigger
- `src/scheduler.py` — filter_classes(classes, preferences); no change; we filter preferences before calling
- `config.yaml` — Add schedules, triggers on preferences
- `src/logger.py` — setup_logging(); Lambda should set LOG_FORMAT=json via env

### New Files to Create

- `src/lambda_handler.py` — Lambda entrypoint
- `deploy/build.sh` — Build zip, output path
- `deploy/terraform/` or `deploy/README.md` — AWS setup instructions (EventBridge + Lambda)
- `tests/test_lambda_handler.py` — Unit tests for handler (mock main)
- `tests/test_config_triggers.py` — Tests for triggers in config

### Relevant Documentation

- [AWS Lambda Python handler](https://docs.aws.amazon.com/lambda/latest/dg/python-handler.html)
  - Handler: `def lambda_handler(event, context):`; return dict
- [EventBridge scheduled rule](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html)
  - Cron in UTC; use Constant (JSON text) for target input to pass `{"trigger": "saturday_19"}`

### Patterns to Follow

**Logging**: `structlog.get_logger()`; `logger.info("event", key=value)` — CLAUDE.md
**Config**: Pydantic BaseModel, `field_validator` — src/config.py
**Exit codes**: 0 success, 1 failure — run.py returns int

---

## IMPLEMENTATION PLAN

### Phase 1: Config Extension (triggers + schedules)

- Add `triggers: list[str] | None = None` to ClassPreference
- Add `schedules: dict[str, dict] | None = None` to Settings (optional; used by deploy/docs)
- When `triggers` is None or [], preference runs on all invocations (backward compat)
- When `triggers` is non-empty, preference runs only when event.trigger in triggers

### Phase 2: Core Logic Update

- Add `run(settings, config_path, dry_run=False, list_classes=False, trigger: str | None = None)` in run.py (or extend main)
- Filter settings.preferences: if trigger is set, keep only prefs with no triggers or trigger in prefs.triggers
- Lambda handler calls this with trigger from event

### Phase 3: Lambda Handler

- `lambda_handler(event, context)` → get trigger from event.get("trigger"), call run(..., trigger=trigger)
- Set LOG_FORMAT=json via Lambda env
- Return {"statusCode": 200 if exit_code==0 else 500}

### Phase 4: Deploy

- Build script: pip install -t package/; cp src config.yaml; zip
- Docs: EventBridge rules with cron `0 17 * * 6`, `0 17 * * 2`; Constant input `{"trigger":"saturday_19"}` and `{"trigger":"tuesday_19"}`

---

## STEP-BY-STEP TASKS

### 1. UPDATE src/config.py — add triggers to ClassPreference

- ADD `triggers: list[str] | None = None` to ClassPreference
- VALIDATE: preferences with triggers must have non-empty list
- PATTERN: MIRROR field_validator from day
- VALIDATE: `uv run ruff check src/config.py`

### 2. UPDATE config.yaml — add schedules and triggers

- ADD top-level `schedules:` (optional, for documentation):
  ```yaml
  schedules:
    saturday_19:
      cron_utc: "0 17 * * 6"
      description: "Saturday 19:00 Israel - וויטליפטינג"
    tuesday_19:
      cron_utc: "0 17 * * 2"
      description: "Tuesday 19:00 Israel - ג'ימנסטיקס"
  ```
- ADD `triggers: ["saturday_19"]` to וויטליפטינג preference
- ADD `triggers: ["tuesday_19"]` to ג'ימנסטיקס preference
- LEAVE WOD preferences without triggers (run on all)
- VALIDATE: `uv run python -c "from src.config import Settings; Settings.load(); print('ok')"` (with .env)

### 3. UPDATE src/config.py — parse schedules

- ADD `schedules: dict[str, dict] | None = None` to Settings (optional field)
- No validator needed; raw dict from YAML

### 4. REFACTOR src/run.py — extract run logic, add trigger filter

- EXTRACT core logic into `def run(config_path, dry_run, list_classes, trigger=None) -> int`
- At start, if trigger: filter `settings.preferences` to those with `triggers is None or triggers==[] or trigger in triggers`
- main() parses args and calls run()
- PATTERN: Keep main() as CLI entrypoint
- VALIDATE: `uv run python -m src.run --dry-run` and `uv run pytest tests/ -v`

### 5. CREATE src/lambda_handler.py

- IMPLEMENT:
  ```python
  def lambda_handler(event, context):
      from src.logger import setup_logging
      from src.run import run
      setup_logging()  # LOG_FORMAT=json from env
      trigger = event.get("trigger")
      exit_code = run(Path("config.yaml"), dry_run=False, list_classes=False, trigger=trigger)
      return {"statusCode": 200 if exit_code == 0 else 500}
  ```
- IMPORTS: Path from pathlib
- GOTCHA: Lambda bundles config.yaml at package root; use Path("config.yaml") or env CONFIG_PATH
- VALIDATE: `uv run ruff check src/`

### 6. UPDATE tests/test_config.py — triggers

- ADD test for preference with triggers
- ADD VALID_CONFIG variant with triggers
- VALIDATE: `uv run pytest tests/test_config.py -v`

### 7. CREATE tests/test_lambda_handler.py

- MOCK run(); assert lambda_handler({"trigger":"saturday_19"}, None) returns 200
- MOCK run() to return 1; assert statusCode 500
- VALIDATE: `uv run pytest tests/test_lambda_handler.py -v`

### 8. CREATE deploy/build.sh

- `pip install -t package/ -r requirements.txt` (or uv pip install)
- `cp -r src config.yaml package/`
- `cd package && zip -r ../lambda.zip .`
- Output: lambda.zip
- VALIDATE: unzip -l lambda.zip | head -20

### 9. CREATE deploy/README.md

- Steps: Create Lambda (Python 3.11), set handler `lambda_handler.lambda_handler`, env ARBOX_EMAIL, ARBOX_PASSWORD, LOG_FORMAT=json
- EventBridge: 2 rules, cron 0 17 * * 6 and 0 17 * * 2, target Lambda, Constant JSON {"trigger":"saturday_19"} and {"trigger":"tuesday_19"}
- IAM: Lambda basic execution role

---

## TESTING STRATEGY

- Unit: config triggers parsing, run() preference filtering
- Unit: lambda_handler with mocked run()
- Integration: `uv run python -m src.run --dry-run` with trigger=saturday_19 (add --trigger flag for local test)

---

## VALIDATION COMMANDS

```bash
uv run ruff check src/ tests/
uv run ruff format src/ tests/
uv run pytest tests/ -v
uv run python -m src.run --dry-run
```

---

## ACCEPTANCE CRITERIA

- [ ] ClassPreference has optional triggers field
- [ ] config.yaml has triggers on weightlifting and gymnastics
- [ ] run() accepts trigger, filters preferences
- [ ] lambda_handler exists, returns statusCode
- [ ] deploy/build.sh produces lambda.zip
- [ ] All tests pass
- [ ] --dry-run works locally with trigger filter (add --trigger for testing)

---

## NOTES

- Lambda timeout: 60s recommended (registration + retries)
- Config path in Lambda: bundle config.yaml in zip root; handler uses Path("config.yaml"). Lambda execution dir is package root.
- For `--trigger` CLI flag: add to argparse for local testing; optional.
