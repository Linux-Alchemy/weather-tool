# Weather Tool — Build Plan

> **Companion to:** `OUTLINE.md` (read that first for the what/why)
> **This file is:** the how — every task, in order, with enough detail to execute or delegate.

**How to use this file:**
- Work top to bottom. Don't skip phases.
- Each task block is self-contained — you or an agent can read one block in isolation.
- Check the boxes as you go. Hit the phase checkpoint before moving on.
- If you're handing a task to an agent, point it at the specific task block.
  It has everything it needs: what to build, what signature to use, and what NOT to touch.

---

## Phase 0: Project Setup + Decisions

**Phase goal:** Scaffold the project and lock in every design choice before writing application code.
**Time estimate:** 30–60 minutes
**Files created:** `.env.example`, `DECISIONS.md`
**Files modified:** `pyproject.toml`, `.gitignore`

---

### Task 0.1: Create project scaffold

**Do:** Run `pyinit` in a new `weather-tool/` directory to generate the standard project structure.

**Done when:**
- `src/weather_tool/__init__.py` exists
- `tests/__init__.py` exists
- `pyproject.toml` exists
- `.venv/` exists
- `uv.lock` exists

**Don't touch:** Nothing else yet — just the scaffold.

- [ ] `mkdir weather-tool && cd weather-tool`
- [ ] `pyinit`
- [ ] Verify the directory tree matches what `pyinit` normally produces

---

### Task 0.2: Add runtime dependency

**Do:** Add `requests` as a project dependency via `uv`.

```bash
uv add requests
```

**Done when:** `requests` appears in `pyproject.toml` under `[project.dependencies]` and `uv.lock` is updated.

**Don't touch:** Don't manually edit `pyproject.toml` for this — let `uv add` handle it.

- [ ] `uv add requests`
- [ ] Verify: `grep requests pyproject.toml`

---

### Task 0.3: Configure CLI entry point

**Do:** Add a console script entry to `pyproject.toml` so `uv run weather` invokes the CLI.

**Add this block to `pyproject.toml`:**
```toml
[project.scripts]
weather = "weather_tool.cli:main"
```

**Done when:** The `[project.scripts]` section exists in `pyproject.toml` with the correct entry.

**Don't touch:** Don't create `cli.py` yet — it doesn't exist and that's fine.

- [ ] Add `[project.scripts]` to `pyproject.toml`

---

### Task 0.4: Set up secrets handling

**Do:** Create `.env.example` and ensure `.env` is gitignored.

**Create `.env.example`:**
```
WEATHER_API_KEY=your_openweathermap_api_key_here
```

**Verify `.gitignore` contains:**
```
.env
```

**Done when:**
- `.env.example` exists with placeholder value only
- `.env` is listed in `.gitignore`
- No actual API key exists anywhere in the repo

**Don't touch:** Don't create `.env` with a real key yet — that happens when you're ready to test live.

- [ ] Create `.env.example`
- [ ] Confirm `.env` is in `.gitignore` (add it if `pyinit` didn't)

---

### Task 0.5: Write DECISIONS.md

**Do:** Document the key architecture decisions for this project. This is a short file — not a novel. One section per decision, each with a "Decision" and a "Why."

**Decisions to document:**
- API: OpenWeatherMap Current Weather Data (why: requires key → secrets practice)
- Auth: `WEATHER_API_KEY` env var (why: keeps secrets out of code and call signatures)
- Errors: Custom exception hierarchy (why: callers can distinguish failure types)
- Layout: `src/weather_tool/` package (why: matches `pyinit`, keeps imports honest)
- CLI: Console script via `pyproject.toml` (why: `uv run weather` instead of `python cli.py`)
- Returns: Frozen dataclass (why: structured, inspectable, serialisable)
- Units: Metric only for v1 (why: simplicity, add unit selection in v2 if needed)
- HTTP: Always `params=`, always `timeout=10` (why: no string concat, no hanging requests)

**Done when:** You can read `DECISIONS.md` and explain every choice without hesitation.

**Don't touch:** Don't write code. This is a documentation task only.

- [ ] Create `DECISIONS.md`
- [ ] One section per decision with rationale

---

### Task 0.6: Get OpenWeatherMap API key

**Do:** Sign up at openweathermap.org and obtain a free API key.

**Done when:**
- You have an API key
- It's stored in your local `.env` file (which is gitignored)
- You've verified it works: `curl "https://api.openweathermap.org/data/2.5/weather?q=Ottawa&units=metric&appid=YOUR_KEY"`

**Gotcha:** New keys can take a few minutes to activate. If you get a 401, wait 10 minutes.

**Don't touch:** Don't put the key anywhere except `.env`. Not in a test file, not in a script, not in your shell history if you can help it.

- [ ] Sign up at openweathermap.org
- [ ] Generate API key
- [ ] Create local `.env` with real key
- [ ] Test with `curl` to confirm key works

---

### Phase 0 Checkpoint

- [ ] Project scaffold exists and `uv sync` succeeds
- [ ] `requests` is in `pyproject.toml` dependencies
- [ ] `[project.scripts]` is configured
- [ ] `.env.example` exists with placeholder
- [ ] `.env` is gitignored
- [ ] `DECISIONS.md` is written and makes sense
- [ ] API key obtained and tested via `curl`
- [ ] `git status` is clean — nothing sensitive staged

---

## Phase 1: Walking Skeleton

**Phase goal:** CLI runs end-to-end returning stubbed (fake) data. The entire pipeline is wired — types, core logic, CLI — but no network calls happen. This proves the structure works before adding complexity.
**Time estimate:** 2–3 hours across tasks
**Files created:** `types.py`, `core.py`, `cli.py`
**Files modified:** `__init__.py`
**Phase constraint:** No real API calls. No `requests` usage. Hardcoded data only.

---

### Task 1.1: Create WeatherReport dataclass

**File:** `src/weather_tool/types.py`

**Skeleton:**
```python
def to_dict(self) -> dict[str, Any]:
    """Return a dict representation, useful for JSON serialisation."""
```

**What it does:**
- Defines a frozen dataclass called `WeatherReport`
- Fields (all required, no defaults):
  - `location: str` — resolved city name from OWM (e.g. "Ottawa, CA")
  - `temperature_c: float` — current temp in Celsius
  - `feels_like_c: float` — apparent temp in Celsius
  - `humidity_pct: int` — relative humidity, 0–100
  - `wind_kph: float` — wind speed in km/h
  - `description: str` — short text e.g. "light rain"
  - `weather_id: int` — OWM numeric weather condition code
- Includes a `to_dict()` method that returns `dataclasses.asdict(self)`
- Class docstring explains that all values use metric units

**Imports needed:** `dataclasses.dataclass`, `dataclasses.asdict`, `typing.Any`

**Done when:**
```python
from weather_tool.types import WeatherReport
r = WeatherReport(location="Ottawa", temperature_c=4.2, feels_like_c=1.0,
                  humidity_pct=78, wind_kph=12.5, description="light rain",
                  weather_id=500)
assert r.temperature_c == 4.2
assert isinstance(r.to_dict(), dict)
r.temperature_c = 99  # Should raise FrozenInstanceError
```

**Don't touch:** No other files. `types.py` depends only on stdlib.

- [ ] Create `types.py` with frozen `WeatherReport` dataclass
- [ ] Add `to_dict()` method using `asdict`
- [ ] Add class docstring explaining metric units
- [ ] Verify in REPL: construction, attribute access, `to_dict()`, frozen enforcement

---

### Task 1.2: Create exception classes and stubbed core function

**File:** `src/weather_tool/core.py`

**Skeleton:**
```python
class WeatherToolError(Exception):
    """Base exception for weather tool failures."""

class ConfigError(WeatherToolError):
    """Raised when required configuration is missing."""

class LocationNotFoundError(WeatherToolError):
    """Raised when the API cannot resolve the requested location."""

class WeatherAPIError(WeatherToolError):
    """Raised when the weather API returns an error or is unreachable."""

def get_current_weather(location: str) -> WeatherReport:
    """
    Fetch current weather conditions for a location.

    Args:
        location: A city name or "city, country" string.
            Examples: "Ottawa", "Tokyo, JP", "London, UK".

    Returns:
        A WeatherReport with current conditions in metric units.

    Raises:
        ConfigError: If WEATHER_API_KEY is not set in the environment.
        LocationNotFoundError: If the location cannot be resolved.
        WeatherAPIError: If the API returns an error or is unreachable.
    """
```

**What it does:**
- Defines four exception classes in a hierarchy (see skeleton above)
- Implements `get_current_weather` as a STUB:
  1. Reads `WEATHER_API_KEY` from `os.environ`
  2. If missing → raise `ConfigError("WEATHER_API_KEY is not set")`
  3. If `location.lower() == "nowhere"` → raise `LocationNotFoundError`
  4. Otherwise → return a hardcoded `WeatherReport` with plausible fake data
- The stub exists so the CLI can be wired and tested before any real API work

**Imports needed:** `os`, `WeatherReport` from `weather_tool.types`

**Rules:**
- Zero `print()` calls in this file — now or ever
- Do not import anything from `cli.py`
- Do not import `requests` yet — that's Phase 3

**Done when:**
```python
import os
os.environ["WEATHER_API_KEY"] = "dummy"
from weather_tool.core import get_current_weather, ConfigError
report = get_current_weather("Ottawa")  # Returns hardcoded WeatherReport
```

**Don't touch:** `types.py` (done), `cli.py` (not yet created).

- [ ] Define exception hierarchy: `WeatherToolError` → `ConfigError`, `LocationNotFoundError`, `WeatherAPIError`
- [ ] Add docstrings to each exception class
- [ ] Implement stubbed `get_current_weather` with env var check
- [ ] Stub returns hardcoded `WeatherReport` for any valid location
- [ ] Stub raises `LocationNotFoundError` for `"nowhere"`
- [ ] Verify: no `print()` anywhere in the file
- [ ] Verify in REPL: happy path returns `WeatherReport`, missing key raises `ConfigError`

---

### Task 1.3: Wire CLI

**File:** `src/weather_tool/cli.py`

**Skeleton:**
```python
def _format_report(report: WeatherReport) -> str:
    """Format a WeatherReport for human-readable terminal output."""

def main() -> int:
    """
    CLI entry point for the weather tool.

    Parses a location argument, fetches weather, prints a formatted report.
    Returns 0 on success, 1 on WeatherToolError, 2 on bad arguments (argparse default).
    """
```

**What it does:**
- Uses `argparse` to accept a single positional argument: `location`
  - Help text: `"City name or 'city, country' (e.g. Ottawa, 'Tokyo, JP')"`
- `_format_report` takes a `WeatherReport` and returns a readable multi-line string
  - Include: location, temp, feels like, humidity, wind, description
  - Format example: `"Weather for Ottawa: 4.2°C (feels like 1.0°C), light rain..."`
  - This is YOUR design choice — make it readable, no strict template required
- `main()` function:
  1. Parse args
  2. Call `get_current_weather(args.location)`
  3. Print `_format_report(report)`
  4. Return 0
  5. Catch `WeatherToolError` → print clean error to stderr, return 1
- Includes `if __name__ == "__main__": sys.exit(main())` at bottom

**Imports needed:** `argparse`, `sys`, `get_current_weather` and `WeatherToolError` from `weather_tool.core`, `WeatherReport` from `weather_tool.types`

**Rules:**
- Catch `WeatherToolError` ONLY — never bare `Exception`
- All `print()` calls in the entire project live here and only here
- Error messages go to `stderr`: `print(f"Error: {e}", file=sys.stderr)`

**Done when:**
```bash
export WEATHER_API_KEY=dummy
uv run weather Ottawa          # Prints formatted stubbed report, exit 0
uv run weather "Tokyo, JP"     # Same
uv run weather nowhere         # Prints clean error, exit 1
unset WEATHER_API_KEY
uv run weather Ottawa          # Prints ConfigError message, exit 1
uv run weather                 # Argparse usage message, exit 2
```

**Don't touch:** `types.py`, `core.py` — both are done for this phase.

- [ ] Set up `argparse` with `location` positional arg
- [ ] Implement `_format_report(report: WeatherReport) -> str`
- [ ] Implement `main() -> int` with error handling
- [ ] Catch `WeatherToolError` only — print to stderr, return 1
- [ ] Add `if __name__ == "__main__": sys.exit(main())`
- [ ] Run all 5 smoke test commands above

---

### Task 1.4: Wire package exports

**File:** `src/weather_tool/__init__.py`

**What it does:**
- Re-exports the public API so users can do:
  ```python
  from weather_tool import get_current_weather, WeatherReport
  from weather_tool import WeatherToolError, ConfigError, LocationNotFoundError, WeatherAPIError
  ```
- Defines `__all__` listing every public export

**Done when:**
```python
from weather_tool import get_current_weather, WeatherReport
from weather_tool import WeatherToolError, ConfigError
report = get_current_weather("Ottawa")  # Works (with WEATHER_API_KEY set)
```

**Don't touch:** No other files.

- [ ] Import and re-export: `get_current_weather`, `WeatherReport`, all 4 exception classes
- [ ] Define `__all__` explicitly
- [ ] Verify imports work in REPL

---

### Phase 1 Checkpoint

- [ ] `uv run weather Ottawa` → formatted stubbed weather, exit 0
- [ ] `uv run weather nowhere` → clean error message, exit 1
- [ ] `uv run weather` (no args) → argparse usage, exit 2
- [ ] Unset `WEATHER_API_KEY` → `uv run weather Ottawa` → ConfigError message, exit 1
- [ ] `from weather_tool import get_current_weather, WeatherReport` works in REPL
- [ ] `grep -rn "print" src/weather_tool/core.py` → zero results
- [ ] `grep -rn "print" src/weather_tool/types.py` → zero results
- [ ] No tracebacks on any expected failure
- [ ] `git add` + `git commit` — nothing sensitive in the diff

---

## Phase 2: Parser + Tests

**Phase goal:** Implement the pure parsing function that converts raw OWM JSON into a `WeatherReport`, and prove it works with tests — all before touching the network.
**Time estimate:** 1.5–2 hours across tasks
**Files created:** `tests/test_core.py`
**Files modified:** `core.py`
**Phase constraint:** No network calls. No `requests`. Tests must pass offline.

---

### Task 2.1: Implement `_parse_weather_response`

**File:** `src/weather_tool/core.py`

**Skeleton:**
```python
def _parse_weather_response(raw: dict) -> WeatherReport:
    """
    Parse a raw OpenWeatherMap API response into a WeatherReport.

    Args:
        raw: The JSON response dict from OWM's /data/2.5/weather endpoint
             (called with units=metric).

    Returns:
        A WeatherReport populated from the response data.

    Raises:
        WeatherAPIError: If expected keys are missing or data is malformed.
    """
```

**What it does:**
- Takes the raw `dict` from OWM's JSON response
- Extracts these fields from the OWM response shape:

| WeatherReport field | OWM JSON path | Notes |
|---|---|---|
| `location` | `raw["name"]` | City name as resolved by OWM |
| `temperature_c` | `raw["main"]["temp"]` | Already Celsius when `units=metric` |
| `feels_like_c` | `raw["main"]["feels_like"]` | Already Celsius when `units=metric` |
| `humidity_pct` | `raw["main"]["humidity"]` | Integer 0–100 |
| `wind_kph` | `raw["wind"]["speed"]` × 3.6 | OWM gives m/s even with metric; multiply by 3.6 for km/h |
| `description` | `raw["weather"][0]["description"]` | Short text like "light rain" |
| `weather_id` | `raw["weather"][0]["id"]` | Numeric condition code |

- Wraps the entire extraction in a try/except that catches `KeyError`, `IndexError`, `TypeError`
  and re-raises as `WeatherAPIError("Malformed API response")`
- This function is PURE — no side effects, no network, no env var access

**Reference: OWM response shape** (what `raw` looks like with `units=metric`):
```json
{
  "name": "Ottawa",
  "main": {
    "temp": 4.2,
    "feels_like": 1.0,
    "humidity": 78,
    "pressure": 1013
  },
  "wind": {
    "speed": 3.47,
    "deg": 210
  },
  "weather": [
    {
      "id": 500,
      "main": "Rain",
      "description": "light rain",
      "icon": "10d"
    }
  ],
  "cod": 200
}
```

**Done when:** Function exists, handles happy path and raises `WeatherAPIError` on bad input.
Actual testing happens in Task 2.2 — this task is implementation only.

**Don't touch:** Don't modify `get_current_weather` yet — it still uses the stub. Don't create test files yet.

- [ ] Implement `_parse_weather_response` with field mapping table above
- [ ] Wind speed conversion: `raw["wind"]["speed"] * 3.6`
- [ ] Wrap extraction in try/except → `WeatherAPIError` on `KeyError`/`IndexError`/`TypeError`
- [ ] Verify function is pure: no print, no network, no env var access

---

### Task 2.2: Write parser tests

**File:** `tests/test_core.py`

**What it does:**
- Three test functions, each testing `_parse_weather_response` against canned data:

**Test 1: `test_parse_valid_response`**
- Input: A dict matching the OWM response shape (use the reference JSON from Task 2.1)
- Asserts: Every field on the returned `WeatherReport` matches expected values
- Key check: `wind_kph` should be `speed * 3.6`, not raw m/s

**Test 2: `test_parse_edge_values`**
- Input: A valid OWM dict but with edge-case values:
  - `temp: -40.0` (extreme cold)
  - `feels_like: -55.0`
  - `humidity: 0` (bone dry)
  - `wind.speed: 50.0` (180 km/h — hurricane)
  - `weather[0].id: 781` (tornado)
- Asserts: All values parse without error, wind converts correctly

**Test 3: `test_parse_malformed_response`**
- Input: Multiple bad inputs, each tested with `pytest.raises(WeatherAPIError)`:
  - Empty dict: `{}`
  - Missing `"main"` key: `{"name": "Test", "weather": [{"id": 1, "description": "x"}], "wind": {"speed": 1}}`
  - Missing `"weather"` key
  - `"weather"` is empty list: `{"name": "Test", "main": {...}, "wind": {...}, "weather": []}`
- Asserts: Each raises `WeatherAPIError`, NOT `KeyError` or `IndexError`

**Imports needed:** `pytest`, `_parse_weather_response` from `weather_tool.core`, `WeatherAPIError` from `weather_tool.core`, `WeatherReport` from `weather_tool.types`

**Note on importing private functions:** Yes, `_parse_weather_response` has a leading underscore.
Testing it directly is fine — it's the pure logic seam and the most valuable thing to test.

**Done when:**
```bash
uv run python -m pytest tests/test_core.py -v
# All 3 tests pass
```

**Don't touch:** `core.py` implementation (done in 2.1). `cli.py`. `types.py`.

- [ ] Create `tests/test_core.py`
- [ ] Write `test_parse_valid_response` with realistic OWM data
- [ ] Write `test_parse_edge_values` with extreme but valid values
- [ ] Write `test_parse_malformed_response` with multiple bad inputs
- [ ] Run `uv run python -m pytest -v` — all green
- [ ] Run with no internet to confirm: no network calls

---

### Phase 2 Checkpoint

- [ ] All 3 tests pass: `uv run python -m pytest -v`
- [ ] Tests run successfully with no internet connection
- [ ] Malformed input raises `WeatherAPIError` (not `KeyError`, not `IndexError`)
- [ ] `_parse_weather_response` has no side effects (no print, no network, no env vars)
- [ ] Phase 1 smoke tests still work (stub CLI still functional)

**⚠️ All tests MUST be green before starting Phase 3. Do not build live API logic on unproven parsing.**

---

## Phase 3: Live API Integration

**Phase goal:** Replace the stub in `get_current_weather` with real HTTP calls to OpenWeatherMap. After this phase, `uv run weather Ottawa` returns actual weather data.
**Time estimate:** 1.5–2 hours across tasks
**Files modified:** `core.py`
**Phase constraint:** Never expose the API key in exceptions, logs, or stdout.

---

### Task 3.1: Implement `_fetch_weather_raw`

**File:** `src/weather_tool/core.py`

**Skeleton:**
```python
def _fetch_weather_raw(location: str, api_key: str) -> dict:
    """
    Call OpenWeatherMap Current Weather API and return the raw JSON response.

    Args:
        location: City name or "city, country" string.
        api_key: OpenWeatherMap API key (never log or include in errors).

    Returns:
        The parsed JSON response as a dict.

    Raises:
        LocationNotFoundError: If OWM returns 404 (city not found).
        WeatherAPIError: If OWM returns 401, 429, any other error,
            or if the network request fails.
    """
```

**What it does:**
- Makes an HTTP GET request to `https://api.openweathermap.org/data/2.5/weather`
- Uses `requests.get()` with keyword arguments:
  - `params={"q": location, "appid": api_key, "units": "metric"}`
  - `timeout=10`
- **NEVER** use string concatenation/f-strings to build the URL with params
- Maps HTTP response codes to exceptions:
  - `404` → `LocationNotFoundError(f"Location not found: {location}")`
  - `401` → `WeatherAPIError("Invalid API key")` (do NOT include the key value)
  - `429` → `WeatherAPIError("API rate limit exceeded")`
  - Any other non-200 → `WeatherAPIError(f"API error: HTTP {response.status_code}")`
- Wraps the entire call in try/except `requests.RequestException` → `WeatherAPIError`
- On 200: returns `response.json()`

**Imports needed:** `requests` (add at top of `core.py`)

**Rules:**
- The API key must NEVER appear in any exception message
- Always use `params=` kwarg — never build query strings manually
- Always use `timeout=10`
- This function is private (`_` prefix) — only called by `get_current_weather`

**Done when:** Function exists and handles all error paths. Integration tested in Task 3.2.

**Don't touch:** `_parse_weather_response` (done). `cli.py`. `types.py`.

- [ ] Add `import requests` to `core.py`
- [ ] Implement `_fetch_weather_raw` with `requests.get(url, params=..., timeout=10)`
- [ ] Map 404 → `LocationNotFoundError`
- [ ] Map 401 → `WeatherAPIError` (no key in message!)
- [ ] Map 429 → `WeatherAPIError`
- [ ] Catch `requests.RequestException` → wrap as `WeatherAPIError`
- [ ] Return `response.json()` on 200

---

### Task 3.2: Wire real `get_current_weather`

**File:** `src/weather_tool/core.py`

**What it does:**
- Replace the stub body of `get_current_weather` with real logic:
  1. Read `WEATHER_API_KEY` from `os.environ` → `ConfigError` if missing (keep this from stub)
  2. Call `_fetch_weather_raw(location, api_key)` → returns raw dict
  3. Call `_parse_weather_response(raw)` → returns `WeatherReport`
  4. Return the `WeatherReport`

**The complete function body should be roughly 5–8 lines.** If it's longer, something's wrong — the work should be delegated to `_fetch` and `_parse`.

**Signature stays exactly the same:**
```python
def get_current_weather(location: str) -> WeatherReport:
```

**Done when:**
```bash
export WEATHER_API_KEY=<your_real_key>
uv run weather Ottawa              # Real weather data
uv run weather "London, UK"        # Real weather data
uv run weather "Tokyo, JP"         # Real weather data
uv run weather "asdfghjkl"         # LocationNotFoundError, exit 1
WEATHER_API_KEY=bogus uv run weather Ottawa  # WeatherAPIError, exit 1
unset WEATHER_API_KEY
uv run weather Ottawa              # ConfigError, exit 1
```

**Don't touch:** `_parse_weather_response`, `_fetch_weather_raw` (both done). `cli.py` (shouldn't need changes). `types.py`.

- [ ] Replace stub body with: read key → fetch → parse → return
- [ ] Keep `ConfigError` check from the stub
- [ ] Run all 6 smoke test commands above
- [ ] Run `uv run python -m pytest -v` — parser tests still green
- [ ] `grep -rn "WEATHER_API_KEY" src/` — only `os.environ` reference, no hardcoded values
- [ ] Check stderr output on errors — no API key visible anywhere

---

### Task 3.3: Secrets audit

**Do:** Verify no secrets leak anywhere in the project.

**Run these checks:**
```bash
# Search for hardcoded keys in source
grep -rn "appid" src/
grep -rn "api_key" src/ --include="*.py"

# Check that api_key only appears as a parameter name, never a value
# Check exception messages don't contain the key
WEATHER_API_KEY=test_secret_key_12345 uv run weather "asdfghjkl" 2>&1 | grep -i "test_secret"
# ^^^ This MUST return nothing

# Verify .env is ignored
git status  # .env should NOT appear
```

**Done when:** All checks pass. No secrets anywhere they shouldn't be.

- [ ] No hardcoded API key values in source
- [ ] Exception messages don't contain key values
- [ ] `.env` doesn't appear in `git status`
- [ ] `git diff --cached | grep -i key` shows nothing sensitive

---

### Phase 3 Checkpoint

- [ ] `uv run weather Ottawa` → real weather data, exit 0
- [ ] `uv run weather "asdfghjkl"` → clean `LocationNotFoundError`, exit 1
- [ ] `WEATHER_API_KEY=bogus uv run weather Ottawa` → clean `WeatherAPIError`, exit 1
- [ ] Unset key → `ConfigError`, exit 1
- [ ] `uv run python -m pytest -v` → all tests still green
- [ ] Secrets audit (Task 3.3) passes completely
- [ ] No tracebacks on expected failures
- [ ] `git add` + `git commit` — clean diff, no secrets

---

## Phase 4: Documentation + SKILL.md + Polish

**Phase goal:** Make the project portfolio-ready for humans and agents.
**Time estimate:** 1–1.5 hours across tasks
**Files created:** `README.md`, `SKILL.md`
**Files modified:** Docstrings across all modules (if needed)

---

### Task 4.1: Write README.md

**File:** `README.md` (project root)

**Include these sections:**
1. **What this is** — one paragraph
2. **Setup** — `uv sync`, `uv add requests` if needed
3. **Configuration** — how to set `WEATHER_API_KEY` (env var, `.env` file)
4. **CLI usage** — `uv run weather Ottawa`, `uv run weather "Tokyo, JP"`
5. **Library usage** — import example with `get_current_weather`
6. **Testing** — `uv run python -m pytest -v`
7. **Agent integration** — link to `SKILL.md`

**Rules:**
- No real API keys in examples — use `<your_key>` or `your_key_here`
- Keep it concise — someone should be able to set up and use this in 5 minutes

**Done when:** A stranger could clone the repo, read the README, and have the tool working.

- [ ] Write README with all 7 sections
- [ ] Verify no real keys in any examples
- [ ] Read it fresh — does it make sense without the outline?

---

### Task 4.2: Write SKILL.md

**File:** `SKILL.md` (project root)

**Include:**
1. **Trigger phrases** — "what's the weather in...", "current temperature in...", "is it raining in..."
2. **Anti-triggers** — climate questions, forecasts, "under the weather", historical weather
3. **Function signature** — `get_current_weather(location: str) -> WeatherReport`
4. **Parameter details** — what `location` accepts: city name, "city, country" format
5. **Return shape** — list every `WeatherReport` field with type and description
6. **Error behaviour** — what each exception means and how the agent should respond to it
7. **Example interaction** — user asks → agent calls function → agent formats response

**Done when:** An LLM reading only this file could correctly decide when to use the tool and how to call it.

- [ ] Write SKILL.md with all 7 sections
- [ ] Verify: would trigger on "weather in Tokyo"?
- [ ] Verify: would NOT trigger on "I'm feeling under the weather"?
- [ ] Verify: return shape matches actual `WeatherReport` fields

---

### Task 4.3: Final docstring and polish pass

**Do:** Read through every public function and class. Verify docstrings match actual behaviour.

**Check each file:**
- `types.py` — `WeatherReport` class docstring, `to_dict` docstring
- `core.py` — all exception class docstrings, `get_current_weather` docstring (Args/Returns/Raises)
- `cli.py` — `main` docstring, `_format_report` docstring
- `__init__.py` — module docstring if desired

**Also verify:**
```bash
# No secrets anywhere
grep -rR "your_real" .
grep -rR "secret" src/

# Tests still pass
uv run python -m pytest -v

# CLI still works
uv run weather Ottawa

# Repo is clean
git status
```

**Done when:** Docstrings are accurate. Tests pass. CLI works. Repo is clean.

- [ ] Review and update all docstrings
- [ ] Final test run: `uv run python -m pytest -v`
- [ ] Final CLI run: `uv run weather Ottawa`
- [ ] Final secrets check
- [ ] `git status` clean

---

### Phase 4 Checkpoint — Ship It

- [ ] `README.md` exists and is clear
- [ ] `SKILL.md` exists and is accurate
- [ ] All docstrings match actual behaviour
- [ ] `uv run python -m pytest -v` → all green
- [ ] `uv run weather Ottawa` → real weather data
- [ ] No secrets in source, docs, or git history
- [ ] You'd be comfortable pushing this to a public GitHub repo right now

---

## Quick Reference: Don't Touch List

A reminder of what's off-limits in each phase:

| Phase | Don't touch |
|---|---|
| 0 | No application code at all |
| 1 | No `requests` usage, no real API calls |
| 2 | No changes to `get_current_weather` wiring, no `cli.py` changes |
| 3 | No `_parse_weather_response` changes, no `cli.py` changes, no `types.py` changes |
| 4 | No logic changes — documentation and polish only |
