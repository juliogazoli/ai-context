## Code Style

- Functions: 4–20 lines. Split if longer.
- Files: under 500 lines. Split by responsibility.
- One thing per function, one responsibility per module (SRP).
- Names: specific and unique. Avoid `data`, `handler`, `Manager`.
  Prefer names that return <5 grep hits in the codebase.
- Types: explicit. No `Any`, no bare `dict`, no untyped functions.
  Use `TypedDict` or dataclasses for structured payloads.
- No code duplication. Extract shared logic into a function or module.
- Early returns over nested ifs. Max 2 levels of indentation.
- Exception messages must include the offending value and expected shape.

## Django Conventions

- Follow the standard Django layout: `models/`, `views/`, `urls/`,
  `serializers/`, `services/`, `tests/`.
- `selectors.py` → all read queries (no writes, no side effects)
- `selectors.py` → all read queries (no writes, no side effects)
- `services.py` → writes, business logic, orchestration
- Views only call selectors and services — no ORM in views directly
- Fat models, thin views — business logic lives in `services/` or
  model methods, not in views or serializers.
- Use class-based views (CBV) for CRUD; function-based views (FBV)
  only for one-off endpoints that don't fit CBV patterns.
- URL names must be namespaced: `app_name` in every `urls.py`.
  Reference them with `reverse("app:name")`, never hardcode paths.
- Never commit secrets — use `django-environ` or `python-decouple`.
- Custom managers go in `managers.py`, not inside the model file,
  if they exceed ~20 lines or are reused across models.

## Models

- Every model field must have an explicit `verbose_name` and, where
  appropriate, `help_text`.
- Use `get_or_create` / `update_or_create` instead of manual
  check-then-save patterns.
- Never use `objects.all()` without a filter in production paths —
  always add `.select_related()` / `.prefetch_related()` where FKs
  are traversed.
- Avoid `null=True` on string fields (`CharField`, `TextField`);
  use `blank=True` and let the empty string be the sentinel.
- Add `__str__`, `Meta.ordering`, and `Meta.verbose_name_plural`
  to every model.

## Security

- Never trust user input. Validate at the form/serializer layer,
  not in the view or model.
- Use Django's built-in CSRF protection. Never exempt views without
  a documented reason in a comment.
- Avoid `raw()` and `extra()` SQL. If you must, parameterise every
  value — never interpolate strings into queries.
- Sensitive fields (`password`, `token`, `secret`) must never appear
  in logs, `__str__`, or API responses.
- Set `SECRET_KEY`, `DATABASE_URL`, and any credentials via
  environment variables. Assert they are present at startup.
- Use `django-ratelimit` or DRF throttling on any unauthenticated
  endpoint.

## Performance

- Enable `django-debug-toolbar` in development; treat any view with
  >10 queries as a bug to fix before merging.
- Use `select_related` for forward FK/OneToOne, `prefetch_related`
  for reverse/M2M. Document why in a comment if the queryset is
  non-obvious.
- Defer heavy computation to Celery tasks. Views must respond in <200 ms
  under normal load.
- Cache with `django.core.cache` behind a named alias
  (`"default"`, `"sessions"`). Cache keys must include a version
  prefix to allow easy invalidation.
- Backend: Redis via `django-redis`. Configure in `settings.py`
  with `CACHE_TTL` as a named constant. Never hardcode TTL values
  inline.
- Use database indexes on every field that appears in a `filter()`,
  `order_by()`, or `get()` in a hot path. Add via `Meta.indexes`.
- No queries inside loops.
- No queries in templates.
- Prefer `bulk_create` and `update()` over per-row saves.
- Prefer annotations over post-processing in Python.
- Use `values()` or `only()` when the full model object isn't needed.

## Comments

- Keep your own comments. Don't strip them on refactor — they carry
  intent and provenance.
- Write WHY, not WHAT. Skip `# increment counter` above `i += 1`.
- Docstrings on public functions and service methods: intent + one
  usage example.
- Reference issue numbers or commit SHAs when a line exists because
  of a specific bug or upstream constraint.

## Tests

- Run all tests with: `venv/bin/python manage.py test` or `pytest --ds=config.settings.test`.
- Every new function and service method gets a unit test.
  Bug fixes get a regression test.
- Use `TestCase` for DB-touching tests, `SimpleTestCase` for pure logic.
- Mock external I/O (APIs, email, S3, Celery) with named fake classes
  (`FakeEmailBackend`, `FakeS3Client`), not inline `MagicMock` stubs.
- Use `factory_boy` for fixtures. No raw `Model.objects.create()`
  scattered across test files.
- Tests must be F.I.R.S.T: fast, independent, repeatable,
  self-validating, timely.

## Dependencies

- Inject dependencies through function parameters or service
  constructors, not via global imports or module-level singletons.
- Wrap third-party libs (`boto3`, `stripe`, `requests`) behind a thin
  interface owned by this project (e.g. `services/storage.py`).
  This makes mocking and swapping trivial.
- Pin all dependencies in `requirements.txt`

## Structure

- Prefer small, focused apps over a monolithic `core/`.
- Each app owns its `urls.py` and is included via `include()` in
  `config/urls.py`.

## Formatting

- Formatter and linter: `ruff format .` and `ruff check .`
  with at minimum `E`, `F`, `I` rule sets. No style discussions beyond this.
- Run both in CI. PRs with formatting failures are not merged.

## Logging

- Use Django's `LOGGING` dict config. Never call `print()` in
  application code.
- Structured JSON in production (via `python-json-logger`).
- Plain text in development for readability.
- Log at `WARNING` or above in views. Use `DEBUG` only for
  development-only diagnostics gated behind `settings.DEBUG`.
- Never log sensitive fields: passwords, tokens, CPF, emails.

## AI Rules

- Do not invent abstractions not present in the codebase.
- Do not add dependencies without an explicit reason.
- Follow existing patterns before introducing new ones.
- If the intent is ambiguous — ask, don't guess.

## Hard Constraints

- No business logic in views, models, or templates.
- No untyped functions.
- No duplicated logic.
- No hidden queries (ORM calls outside selectors/services).
- No silent error swallowing.

## Priority (when in conflict)

1. Security
2. Correctness
3. Readability
4. Performance
5. Conciseness
