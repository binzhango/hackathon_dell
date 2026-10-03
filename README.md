# ClawWatch Log Lab

ClawWatch Log Lab is a local Python demo for replaying synthetic SIEM events into SQLite
and monitoring them through a Gradio dashboard. Development follows the reviewed
[phase plan](docs/gradio-live-log-demo-plan.md).

## Current status

All six planned phases are complete: Python 3.12 packaging, configuration, dependency locks,
versioned SQLite storage, the pandas-backed importer, durable replay, the Gradio monitoring
dashboard, the persistent review board, and final performance and browser validation.

## Development setup

Run the idempotent setup script from the repository root:

```bash
./scripts/setup.sh
```

It creates the Python 3.12 environment, installs the development lockfile and editable
package, protects or creates `.env`, validates the configuration, and safely imports the
dataset. Use `--skip-import` for environment-only setup or `--runtime-only` to install the
runtime lockfile.

Equivalent manual commands:

```bash
rtk proxy uv venv --python 3.12 --seed .venv
rtk proxy .venv/bin/python -m pip install -r requirements-dev.lock
rtk proxy .venv/bin/python -m pip install --no-deps --no-build-isolation -e .
```

`uv venv` is used because this machine's managed Python 3.12 installation cannot run
`ensurepip` correctly through `python3.12 -m venv`. The resulting `.venv` is a normal,
isolated CPython 3.12 virtual environment.

Validate the checked-in configuration:

```bash
rtk proxy .venv/bin/clawwatch-demo config-check --config config/demo.toml
```

The configured database path resolves to
`/Users/binzhang/vibe_coding_repo/hackathon_dell/var/clawwatch_demo.sqlite3` and is excluded
from Git.

Import the configured JSONL dataset or safely re-run an existing completed import:

```bash
rtk proxy .venv/bin/clawwatch-demo import-data --config config/demo.toml
rtk proxy .venv/bin/clawwatch-demo db-info --config config/demo.toml
```

The importer reads bounded 10,000-row batches with pandas, validates and normalizes each
record, bulk-inserts accepted events, and falls back to isolated JSON parsing when a batch
contains malformed input. A clean 100,000-row import measured 4.04 seconds (about 24,778
rows/second) on the development machine. The complete original JSON line remains stored
alongside normalized query fields.

The replay engine in `clawwatch_demo.replay` provides fixed-rate source or original-time
ordering, event-type and severity selection, record limits, and acknowledged start, pause,
resume, rate-change, stop, and interruption controls. Each emitted event and its selection
cursor commit in one transaction. Restart recovery marks an unfinished producer as
interrupted and resumes from its last committed sequence only after an explicit command.

Launch the local dashboard:

```bash
rtk proxy .venv/bin/clawwatch-demo serve --config config/demo.toml
```

Open `http://127.0.0.1:7860`. The monitor provides replay controls, live counters, pandas-
backed charts and tables, literal search, event details, original payload inspection, run
history, and a four-stage review board. Review notes, moves, revision checks, and activity
history persist in the repository SQLite database.

The final stress checks sustained 99.9 events/second for a real-time 1,000-event run and
persisted a controllable-clock, full-corpus replay of 100,000 unique sequences and source
events. Five dashboard snapshots over that full run took 0.250–0.259 seconds each. The UI
was inspected at 1440×900 and 1280×800 without horizontal page overflow, and the browser
walkthrough covered event details, idempotent review creation, note/stage persistence, stale
revision protection, and activity history.

Run the automated checks:

```bash
rtk proxy .venv/bin/python -m pytest
rtk proxy .venv/bin/python -m ruff check .
rtk proxy .venv/bin/python -m ruff format --check .
```
