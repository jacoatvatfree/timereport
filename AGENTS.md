# Agent Guidelines for Time Report Project

## Project Overview
Python scripts for generating time entry reports from GitHub commits and Slack huddles.

## Running Scripts
- Run GitHub report: `python3 generate-github-report.py [start_date] [end_date]`
- Run Slack report: `python3 generate-slack-report.py [start_date] [end_date]`
- Format report: `./generate-github-report.py | ./format-time-report.py`
- Combine sources: `cat github.json slack.json | jq -s 'add' | ./format-time-report.py`
- No test suite exists; scripts are standalone utilities

## Code Style
- Python 3 with standard library (subprocess, json, datetime, argparse)
- Use `#!/usr/bin/env python3` shebang for executable scripts
- Functions: lowercase with underscores (e.g., `run_command()`, `get_date_range()`)
- Constants: uppercase (e.g., default org='vatfree')
- Comprehensive docstrings at module and function level
- Type hints: Not used in this codebase
- Imports: Group standard library imports together at top
- Error handling: Use try/except with specific exceptions, print errors to stderr
- Output: Progress/errors to stderr, data to stdout (for piping)
- Date format: YYYY-MM-DD for inputs, '%d %b %Y, %H:%M' for display
