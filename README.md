# clipboard-sharing-app

Send up to 300 characters of text from one device to another using a 6-character code. The text can be read once and expires automatically after 10 minutes.

**Status:** in development. A hello-world API runs locally; storage is not connected yet.

## How it works (planned)

1. The sender's page posts text to the API and gets a 6-character code.
2. The receiver types the code and gets the text once. After that read, the text is deleted.

## Architecture (planned)

| Piece | Tool |
|---|---|
| Frontend | Static page on GitHub Pages |
| API | FastAPI on Render (free web service) |
| Storage | Upstash Redis (free tier) with automatic expiry |

## Data model

| Item | Value |
|---|---|
| Redis key | `clip:<code>` |
| Value | the text, 300 characters max |
| Lifetime | 600 seconds, set once at creation, never extended |
| Code | 6 characters from a secure random source |

Nothing else is stored: no accounts, no IP addresses, no history.

## API contract (planned)

| Route | Success | Failure |
|---|---|---|
| `POST /clips` with `{"text": "..."}` | `201` with `{"code": "...", "expires_in": 600}` | `422` if empty or over 300 characters |
| `GET /clips/{code}` | `200` with `{"text": "..."}`, then the clip is deleted | `404` for unknown, expired, or already-read codes |

## Rules the server enforces

- The 300-character limit is checked on the server, not only in the form.
- Codes use 56 characters (letters and digits, minus `0 O o 1 l I`), about 30.8 billion combinations.
- Uniqueness comes from storing with `SET ... NX`, which refuses to overwrite an existing code.
- Reading uses `GETDEL`, so a clip can be read once.
- Unknown, expired, and already-read codes all return the same `404`, so guessers learn nothing.
- Lookups will be rate-limited, and only this site's address may call the API (CORS).

## Design decisions

- **Delete on first read** was chosen over "delete after N reads" or a 30-second grace window. It is the simplest rule and the strongest signal: if someone else reads first, the real receiver sees "not found." A grace window may be revisited if testing shows reads getting lost.
- Text is not encrypted at rest. Do not send passwords or other secrets.

## Run the API locally (Windows)

1. `cd api`
2. `py -m venv .venv`
3. Activate it: `.venv\Scripts\activate` (Command Prompt) or `.venv\Scripts\Activate.ps1` (PowerShell)
4. `pip install "fastapi[standard]"`
5. `fastapi dev`
6. Open http://127.0.0.1:8000/docs