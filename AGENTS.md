# Agent Guidelines for Time Report Project

## Project Overview
Python scripts for generating time entry reports from GitHub commits, Slack huddles, and calendar events (via Nylas API).

## Running Scripts
- Run GitHub report: `python3 generate-github-report.py [start_date] [end_date]`
- Run Slack report: `python3 generate-slack-report.py [start_date] [end_date]`
- Run Calendar report: `python3 generate-calendar-report.py [start_date] [end_date]`
- Format report: `./generate-github-report.py | ./format-time-report.py`
- Combine sources: `jq -s 'add' <(./generate-github-report.py) <(./generate-slack-report.py) <(./generate-calendar-report.py) | ./format-time-report.py`
- No test suite exists; scripts are standalone utilities

## Code Style
- Python 3 with standard library (subprocess, json, datetime, argparse, urllib)
- Use `#!/usr/bin/env python3` shebang for executable scripts
- Functions: lowercase with underscores (e.g., `run_command()`, `get_date_range()`)
- Constants: uppercase (e.g., default org='vatfree')
- Comprehensive docstrings at module and function level
- Type hints: Not used in this codebase
- Imports: Group standard library imports together at top
- Error handling: Use try/except with specific exceptions, print errors to stderr
- Output: Progress/errors to stderr, data to stdout (for piping)
- Date format: YYYY-MM-DD for inputs, '%d %b %Y, %H:%M' for display
- API calls: Use urllib.request for HTTP requests (no external dependencies)
