# Code Review

GitHub Action that auto-reviews Pull Requests using Google Gemini with DeepSeek fallback.

## How It Works

1. **Evaluate Diff & Check PR Size** — Fetches the PR diff and skips if it's empty or exceeds line/file limits.
2. **AI Review** — Sends the diff to the primary Gemini model with a structured prompt covering:
   - Critical issues (security, data loss, unhandled exceptions)
   - Performance & scalability (N+1 queries, blocking I/O, inefficient algorithms)
   - Maintainability (SRP violations, debug leftovers, cascading impact)
   
   If the primary model fails (rate limited, unavailable), it falls back through: **Primary Gemini → Fallback Gemini → DeepSeek**
3. **Post Comment** — Publishes the review as a formatted PR comment.

## Usage

```yaml
name: AI Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: javedquadri/code-review@main
        with:
          gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
```

### With DeepSeek Fallback

```yaml
- uses: javedquadri/code-review@main
  with:
    gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
    deepseek_api_key: ${{ secrets.DEEPSEEK_API_KEY }}
```

### Customizing Models

```yaml
- uses: javedquadri/code-review@main
  with:
    gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
    model: gemini-2.5-pro
    fallback_gemini_model: gemini-2.5-flash
    deepseek_model: deepseek-chat
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `gemini_api_key` | Yes | — | Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey) |
| `deepseek_api_key` | No | `''` | DeepSeek API key for fallback |
| `model` | No | `gemini-2.5-flash` | Primary Gemini model |
| `fallback_gemini_model` | No | `gemini-3.5-flash` | Fallback Gemini model if primary fails |
| `deepseek_model` | No | `deepseek-v4-flash` | DeepSeek model for final fallback |
| `tech_stack` | No | `''` | Tech stack description (e.g. `Python/Django`). Leave blank for auto-detection |
| `max_changed_lines` | No | `3000` | Skip review if total changed lines exceeds this |
| `max_changed_files` | No | `60` | Skip review if changed files exceeds this |

## Fallback Chain

1. **Primary Gemini** (`model`) — first attempt
2. **Fallback Gemini** (`fallback_gemini_model`) — tried if primary fails (rate limits with 5s pause, 500/503, or other errors)
3. **DeepSeek** (`deepseek_model`) — final attempt, only if `deepseek_api_key` is provided

If all providers fail, a message is posted asking reviewers to proceed manually.

## Permissions

The action needs these permissions in your workflow:

```yaml
permissions:
  contents: read      # To fetch the PR diff
  pull-requests: write # To post review comments
```

## Requirements

- `actions/checkout@v4` with `fetch-depth: 0` (needs full git history to diff)
- Triggered by `pull_request` events (`opened`, `synchronize`)
