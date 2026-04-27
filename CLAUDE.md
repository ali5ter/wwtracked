# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`wwtracked.py` is a single-file Python CLI that calls the undocumented Weight Watchers API to generate a
Markdown-formatted report of tracked foods over a date range. Optionally outputs nutritional data as a CSV.

## Running the Script

```bash
# Authenticate with email (prompts for password interactively)
python wwtracked.py -s 2024-01-01 -e 2024-01-07 -E youremail@example.com

# Authenticate with email and password inline (smbclient convention: email%password)
python wwtracked.py -s 2024-01-01 -e 2024-01-07 -E youremail@example.com%yourpassword

# Authenticate with a JWT from the browser
python wwtracked.py -s 2024-01-01 -e 2024-01-07 -J "Bearer eyJ0eX...zqdVwoQ"

# Include nutritional CSV output
python wwtracked.py -s 2024-01-01 -e 2024-01-07 -E email@example.com --nutrition

# Use a non-.com TLD (e.g., UK)
python wwtracked.py -s 2024-01-01 -e 2024-01-07 -E email@example.com -l co.uk
```

## Linting

```bash
flake8 wwtracked.py   # max line length is 120 (configured in .flake8)
```

## Architecture

The entire implementation lives in `wwtracked.py` with no external project dependencies beyond `requests`.

**Authentication flow** (`login` → `login1` → `login2`):

1. POST email+password to `auth.weightwatchers.{tld}/login-apis/v1/authenticate` → get `tokenId`
2. GET `auth.weightwatchers.{tld}/openam/oauth2/authorize` with `wwAuth2` cookie → 302 redirect contains
   the `id_token` JWT in the fragment

**Main report loop**: iterates each date in the range, calls
`cmx.weightwatchers.{tld}/api/v3/cmx/operations/composed/members/~/my-day/{date}`, and prints Markdown
sections for morning/midday/evening/anytime meals.

**Nutrition path** (`--nutrition` flag): for each food entry, `getfoodentrynutrition()` hits a separate food
or recipe endpoint to retrieve per-portion nutrition data, scales it by the tracked portion size, and
accumulates results in `nutritionarr`. After the report loop, `writenutritiondata()` writes the CSV.

**Global state**: `requestnutrition`, `nutritionarr`, `authheader`, `startdate`, `enddate`, and `tld` are
module-level globals set in `__main__` and referenced inside helper functions — keep this in mind when
refactoring.

**`sourceType` routing**: `getfoodentrynutrition()` selects the API endpoint based on `sourceType`
(WWFOOD, MEMBERFOOD, WWVENDORFOOD, MEMBERRECIPE, WWRECIPE). Recipes require a different response-parsing
path than regular foods. `MEMBERFOODQUICK` (quick-add) entries are skipped entirely.
