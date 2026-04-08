# Time Report Generator

Generate time entry reports from your GitHub commits and Slack huddles.

## Overview

This tool consists of 4 Python scripts:

1. **`generate-github-report.py`** - Extracts time from GitHub commits
2. **`generate-slack-report.py`** - Extracts time from Slack huddles
3. **`generate-calendar-report.py`** - Extracts time from calendar events (via Nylas API)
4. **`format-time-report.py`** - Formats JSON data into readable time entries

## Features

### GitHub Report

- Fetches all your commits from the last week (Monday-Sunday) by default
- Groups commits by Pull Request
- Assumes 30 minutes of work per commit
- **Merges overlapping time blocks** - multiple commits close together become one longer session
- Extracts eng tags (e.g., "eng707") from PR titles and formats them as hashtags
- Works across all branches (not just default branch)
- Handles squash-merged PRs correctly

### Slack Report

- Loads huddles from `slack_huddles.json` file (via bookmarklet)
- Filters by your user ID and date range
- Labels all huddles as "Slack huddle #meetings"
- Preserves actual huddle start/end times

### Calendar Report

- Fetches calendar events from Nylas API (Google Calendar, Outlook, Exchange, etc.)
- Only includes busy time (excludes transparent/free events)
- Excludes cancelled events
- Uses actual meeting titles with #meetings hashtag
- Preserves actual event start/end times
- Optional: only runs if Nylas credentials are set

### Formatter

- Converts JSON to YAML-like time entry format
- Merges overlapping sessions automatically
- Sorts by timestamp

## Requirements

- Python 3
- GitHub CLI (`gh`) installed and authenticated (for GitHub data)
- Access to the vatfree GitHub organization
- Slack bookmarklet for downloading huddles data (for Slack data)

## Setup

### Environment Variables

```bash
# Required for Slack data
export SLACK_USER_ID='U03H3A69E2D'           # Your Slack user ID

# Optional for calendar data (via Nylas API)
export NYLAS_API_KEY='nylas_api_key_...'     # Your Nylas API key
export NYLAS_GRANT_ID='grant_id_...'         # Your Nylas grant ID (connected calendar)

# Optional
export SLACK_HUDDLES_PATH='~/Downloads'       # Where slack_huddles.json is located (default: ~/Downloads)
export GITHUB_ORG='vatfree'                   # GitHub organization (default: vatfree)
```

### Finding Your Slack User ID

In Slack: Profile → More → Copy member ID

### Getting Slack Huddles Data

1. Open Slack in your browser
2. Run the bookmarklet to download huddles data
3. Save the file as `slack_huddles.json` in your Downloads folder (or set `SLACK_HUDDLES_PATH`)

### Getting Your Nylas Credentials

1. Sign up at https://dashboard.nylas.com (free for up to 5 accounts)
2. Create a new application
3. Connect your Google Calendar (or other calendar provider)
4. Get your API key from Settings
5. Get your grant ID after connecting a calendar
6. Set the environment variables:
   ```bash
   export NYLAS_API_KEY='nylas_api_key_...'
   export NYLAS_GRANT_ID='grant_id_...'
   ```

## Usage

### Option 1: Individual Scripts (Separate Reports)

#### GitHub Only

```bash
./generate-github-report.py 2026-02-08 2026-02-09 | ./format-time-report.py
```

#### Slack Only

```bash
./generate-slack-report.py 2026-02-08 2026-02-09 | ./format-time-report.py
```

#### Calendar Only

```bash
./generate-calendar-report.py 2026-02-08 2026-02-09 | ./format-time-report.py
```

### Option 2: Combined Report (Recommended)

Combine GitHub, Slack, and Calendar data into one report:

```bash
# Using process substitution and jq to merge JSON
jq -s 'add' <(./generate-github-report.py) \
            <(./generate-slack-report.py) \
            <(./generate-calendar-report.py) \
    | ./format-time-report.py
```

#### Save to File

```bash
jq -s 'add' <(./generate-github-report.py 2026-02-08 2026-02-09) \
            <(./generate-slack-report.py 2026-02-08 2026-02-09) \
            <(./generate-calendar-report.py 2026-02-08 2026-02-09) \
    | ./format-time-report.py -o time-report.txt
```

#### Using Default Date Range (Current Week)

```bash
# Omit dates to use current week (Monday to today)
jq -s 'add' <(./generate-github-report.py) \
            <(./generate-slack-report.py) \
            <(./generate-calendar-report.py) \
    | ./format-time-report.py
```

### Option 3: Manual JSON Combination

If you prefer to save intermediate files:

```bash
# Generate individual JSON files
./generate-github-report.py 2026-02-08 2026-02-09 -o github.json
./generate-slack-report.py 2026-02-08 2026-02-09 -o slack.json
./generate-calendar-report.py 2026-02-08 2026-02-09 -o calendar.json

# Combine them
jq -s 'add' github.json slack.json calendar.json | ./format-time-report.py

# Or save combined JSON
jq -s 'add' github.json slack.json calendar.json > combined.json
./format-time-report.py combined.json
```

## Output Format

The formatter generates a YAML-like format:

```yaml
# Time Entry Submission
# Review and confirm the sessions below:
# Save and close this file to submit, or delete all content to cancel

tasks:
  - taskName: "Make all receipts post free #eng707"
    focus:
      - 6 Feb 2026, 08:09, 30 min
      - 6 Feb 2026, 09:00, 45 min
      - 6 Feb 2026, 14:15, 30 min
  - taskName: "Slack huddle #meetings"
    focus:
      - 6 Feb 2026, 10:00, 60 min
      - 6 Feb 2026, 15:30, 45 min
  - taskName: "Team Standup #meetings"
    focus:
      - 6 Feb 2026, 11:00, 30 min
  - taskName: "Client Review Meeting #meetings"
    focus:
      - 7 Feb 2026, 14:00, 60 min
  - taskName: "Update translations #eng456"
    focus:
      - 6 Feb 2026, 08:58, 30 min
```

### Session Merging

If you make multiple commits within overlapping 30-minute windows, they're merged into a single longer session. For example, commits at 09:15 and 09:30 become a single 45-minute session (08:45-09:30).

### Eng Tag Extraction

The script automatically:

- Detects eng tags in various formats: `eng707`, `eng-707`, `ENG 707`, `eng#707`
- Removes the eng tag from the PR title
- Adds it as a hashtag at the end: `#eng707`

Example:

- PR Title: "Fix receipt issue eng-707"
- Output: `"Fix receipt issue #eng707"`

## Script Details

### `generate-github-report.py`

**Outputs:** JSON array of tasks

**JSON Format:**

```json
[
  {
    "name": "Fix receipt issue #eng707",
    "sessions": [{ "start": 1707209340, "end": 1707211140 }],
    "sort_timestamp": 1707209340
  }
]
```

**Command Line Options:**

```bash
./generate-github-report.py [start_date] [end_date] [-o output.json] [--org organization]
```

### `generate-slack-report.py`

**Outputs:** JSON array of tasks

**JSON Format:**

```json
[
  {
    "name": "Slack huddle #meetings",
    "sessions": [{ "start": 1707212400, "end": 1707216000 }],
    "sort_timestamp": 1707212400
  }
]
```

**Command Line Options:**

```bash
./generate-slack-report.py [start_date] [end_date] [-o output.json] [--slack-user-id UID] [--slack-huddles-path PATH]
```

**Requirements:**

- `SLACK_USER_ID` environment variable or `--slack-user-id` argument
- `slack_huddles.json` file in `~/Downloads` or path specified by `SLACK_HUDDLES_PATH`

### `generate-calendar-report.py`

**Outputs:** JSON array of tasks

**JSON Format:**

```json
[
  {
    "name": "Calendar Appointment #meetings",
    "sessions": [{ "start": 1707214800, "end": 1707216600 }],
    "sort_timestamp": 1707214800
  }
]
```

**Command Line Options:**

```bash
./generate-calendar-report.py [start_date] [end_date] [-o output.json] [--nylas-api-key KEY] [--nylas-grant-id ID]
```

**Requirements:**

- `NYLAS_API_KEY` and `NYLAS_GRANT_ID` environment variables (or CLI arguments)
- Nylas account with connected calendar (free for up to 5 accounts)

### `format-time-report.py`

**Inputs:** JSON from stdin or file

**Outputs:** Formatted YAML-like text

**Command Line Options:**

```bash
./format-time-report.py [input.json] [-o output.txt]
```

- If no input file specified, reads from stdin
- If no output file specified, writes to stdout

## How It Works

### GitHub Pipeline

1. Fetches all repositories in the specified organization
2. Gets all merged PRs in the date range across all repositories
3. For each PR, checks if it contains commits by you (using your git email)
4. **Works across all branches** - finds commits even if PRs merge to `development`, `staging`, or other non-default branches
5. **Handles squash merges** - examines the original commits in the PR before they were squashed
6. Filters commits to only include those within the specified date range
7. Groups commits by PR and extracts eng tags from PR titles
8. Assumes 30 minutes of work per commit
9. Outputs JSON array of tasks with sessions

### Slack Pipeline

1. Loads huddles from `slack_huddles.json` file
2. Filters by your `SLACK_USER_ID` (only huddles you participated in)
3. Filters by date range
4. Labels all huddles as "Slack huddle #meetings"
5. Preserves actual huddle duration from start to end
6. Outputs JSON array of tasks with sessions

### Calendar Pipeline

1. Fetches events from Nylas API using `NYLAS_API_KEY` and `NYLAS_GRANT_ID`
2. Filters by date range and busy status (only busy events, excludes free time)
3. Excludes cancelled events
4. Labels all events as "Calendar Appointment #meetings"
5. Preserves actual event duration from start to end
6. Outputs JSON array of tasks with sessions
7. Gracefully skips if Nylas credentials are not set

### Formatting Pipeline

1. Reads JSON array from all sources (GitHub, Slack, Calendar, etc.)
2. Merges overlapping time sessions within each task
3. Sorts tasks by earliest timestamp
4. Formats as YAML-like time entry submission

## Data Flow Diagram

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│ GitHub Commits      │     │ Slack Huddles       │     │ Calendar Events     │
│ (via gh CLI)        │     │ (slack_huddles.json)│     │ (via Nylas API)     │
└──────────┬──────────┘     └──────────┬──────────┘     └──────────┬──────────┘
           │                           │                           │
           ▼                           ▼                           ▼
  ┌────────────────┐         ┌────────────────┐         ┌────────────────┐
  │ generate-      │         │ generate-      │         │ generate-      │
  │ github-report  │         │ slack-report   │         │ calendar-report│
  └────────┬───────┘         └────────┬───────┘         └────────┬───────┘
           │                           │                           │
           │    JSON Array             │    JSON Array             │    JSON Array
           │    ┌──────────────────────────────────────────┐       │
           └───►│                                          │◄──────┘
                │          jq -s 'add'                     │
                │          (merge arrays)                  │
                └────────────────┬─────────────────────────┘
                                 │
                                 ▼    Combined JSON
                         ┌───────────────┐
                         │ format-time-  │
                         │ report        │
                         └───────┬───────┘
                                 │
                                 ▼
                          YAML-like output
```

## Examples

### Example 1: GitHub Only

```bash
./generate-github-report.py 2026-02-03 2026-02-09 | ./format-time-report.py
```

### Example 2: Slack Only

```bash
export SLACK_USER_ID='U03H3A69E2D'
./generate-slack-report.py 2026-02-03 2026-02-09 | ./format-time-report.py
```

### Example 3: Combined Report (Full Week)

```bash
export SLACK_USER_ID='U03H3A69E2D'
export NYLAS_API_KEY='nylas_api_key_...'
export NYLAS_GRANT_ID='grant_id_...'

jq -s 'add' <(./generate-github-report.py 2026-02-03 2026-02-09) \
            <(./generate-slack-report.py 2026-02-03 2026-02-09) \
            <(./generate-calendar-report.py 2026-02-03 2026-02-09) \
    | ./format-time-report.py
```

### Example 4: This Week (Auto Date Range)

```bash
export SLACK_USER_ID='U03H3A69E2D'
export NYLAS_API_KEY='nylas_api_key_...'
export NYLAS_GRANT_ID='grant_id_...'

jq -s 'add' <(./generate-github-report.py) \
            <(./generate-slack-report.py) \
            <(./generate-calendar-report.py) \
    | ./format-time-report.py
```

### Example 5: Save to File

```bash
export SLACK_USER_ID='U03H3A69E2D'
export NYLAS_API_KEY='nylas_api_key_...'
export NYLAS_GRANT_ID='grant_id_...'

jq -s 'add' <(./generate-github-report.py) \
            <(./generate-slack-report.py) \
            <(./generate-calendar-report.py) \
    | ./format-time-report.py -o my-timesheet.txt
```

## Important Notes

### GitHub Workflow

This script is designed to work with your workflow:

- You create a feature branch and make commits there
- GitHub Actions creates a PR automatically (via bot)
- The PR is squash merged, combining all commits into one
- **The script looks at the original commits in the PR** (before squashing) to capture all your work
- Works across all branches, not just the default branch (e.g., finds easyclaim PRs merged to `development`)

### Time Assumptions

- Each GitHub commit represents a 30-minute work session ending at the commit time
- **Overlapping sessions are merged automatically** - commits close together become one longer session
- Slack huddles use their actual duration (not 30-minute blocks)
- The minimum time per task is 30 minutes for GitHub commits

### Data Sources

- GitHub data is fetched live via `gh` CLI
- Slack data must be downloaded manually via bookmarklet first
- Calendar data is fetched live via Nylas API (requires API key and grant ID)
- Progress messages go to stderr, so stdout can be redirected to files

### Performance

- The GitHub script may take a few minutes if you have many repositories
- Slack processing is fast (just reads a local JSON file)
- Calendar processing depends on Nylas API response time (usually fast)

### Eng Tags

- Detected in formats like: `eng707`, `eng-707`, `ENG 707`, `eng#707`, `[eng789]`
- Automatically extracted from PR titles and formatted as hashtags

## Troubleshooting

### "SLACK_USER_ID not set"

```bash
export SLACK_USER_ID='U03H3A69E2D'
```

Find your ID in Slack: Profile → More → Copy member ID

### "No slack_huddles.json file found"

1. Make sure you ran the bookmarklet in Slack
2. Check the file is saved as `slack_huddles.json` (not `slack_huddles (1).json`)
3. Verify it's in `~/Downloads` or set `SLACK_HUDDLES_PATH`

### "Nylas credentials not set"

This is just informational - calendar integration is optional. If you want to include calendar events:

1. Sign up at https://dashboard.nylas.com (free for up to 5 accounts)
2. Connect your calendar and get your credentials
3. Set the environment variables:
   ```bash
   export NYLAS_API_KEY='nylas_api_key_...'
   export NYLAS_GRANT_ID='grant_id_...'
   ```

### "jq: command not found"

Install jq:

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq
```

### Empty Output

- Check you have commits/huddles in the specified date range
- Verify `gh` is authenticated: `gh auth status`
- Check stderr for error messages

## File Structure

```
timereport/
├── bookmarklet.js               # Run to listen to slack huddles.history call and save JSON
├── generate-github-report.py    # Extracts data from GitHub
├── generate-slack-report.py     # Extracts data from Slack huddles
├── generate-calendar-report.py  # Extracts data from calendar (via Nylas API)
├── format-time-report.py        # Formats JSON to YAML
└── README.md                    # This file
```

## Advanced Usage

### Custom Date Range for Each Source

```bash
# Get GitHub commits from one week, Slack huddles from another, calendar from yet another
jq -s 'add' <(./generate-github-report.py 2026-02-03 2026-02-09) \
            <(./generate-slack-report.py 2026-02-01 2026-02-28) \
            <(./generate-calendar-report.py 2026-02-01 2026-02-28) \
    | ./format-time-report.py
```

### Inspect Raw JSON

```bash
# See what GitHub returns
./generate-github-report.py 2026-02-03 2026-02-09 | jq .

# See what Slack returns
./generate-slack-report.py 2026-02-03 2026-02-09 | jq .

# See what Calendar returns
./generate-calendar-report.py 2026-02-03 2026-02-09 | jq .

# See combined data
jq -s 'add' <(./generate-github-report.py 2026-02-03 2026-02-09) \
            <(./generate-slack-report.py 2026-02-03 2026-02-09) \
            <(./generate-calendar-report.py 2026-02-03 2026-02-09) \
    | jq .
```

### Add More Data Sources

The JSON format is extensible. You can create your own script that outputs:

```json
[
  {
    "name": "Task name #tag",
    "sessions": [
      {"start": unix_timestamp, "end": unix_timestamp}
    ],
    "sort_timestamp": unix_timestamp
  }
]
```

Then combine it with the others:

```bash
jq -s 'add' <(./generate-github-report.py) \
            <(./generate-slack-report.py) \
            <(./generate-calendar-report.py) \
            <(./my-custom-report.py) \
    | ./format-time-report.py
```
