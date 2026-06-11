# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# JARVIS — Voice AI Assistant

JARVIS is a voice-first AI assistant for macOS: a Python/FastAPI backend, a Vite/TypeScript/Three.js frontend, and AppleScript bridges to Apple Calendar, Mail, and Notes. It can also spawn Claude Code subprocesses to perform real development work.

## First-Run Setup (walk a new user through this)
The README points new users here; help them through these steps in order:
1. `cp .env.example .env` — then add keys (see Environment Variables below). Note `.env.example` only lists `ANTHROPIC_API_KEY`; `FISH_API_KEY` is also required.
2. Get an Anthropic API key from console.anthropic.com and a Fish Audio key from fish.audio.
3. `pip install -r requirements.txt`
4. `cd frontend && npm install`
5. Generate SSL certs (needed for `wss://` + mic access): `openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes -subj '/CN=localhost'`
6. Install the Claude Code CLI (`claude`) — required for BUILD/RESEARCH/work-mode actions.
7. Run backend + frontend (see Commands), open Chrome (Web Speech API is Chrome-only), click once to enable audio, then speak.

## Commands
```bash
# Backend — defaults to port 8340, auto-enables HTTPS if cert.pem/key.pem exist
python server.py                    # 0.0.0.0:8340
python server.py --reload           # auto-reload on changes (dev)
python server.py --port 8340 --ssl  # force HTTPS
# Banner prints the actual WebSocket/REST URLs on startup.

# Frontend (separate terminal) — Vite dev server on :5173
cd frontend && npm run dev
cd frontend && npm run build        # tsc + vite build
cd frontend && npm run preview

# Tests — mixed: some are pytest, some are standalone asyncio scripts
pytest tests/                                   # runs pytest-style tests (needs pytest + pytest-asyncio)
pytest tests/test_feedback_loop.py              # single file
pytest tests/test_feedback_loop.py::test_name   # single test
python tests/test_classifier.py                 # standalone scripts have __main__ / asyncio.run
```
Note: `pytest` and `pytest-asyncio` are NOT in `requirements.txt` — install them separately to run the suite. Integration tests (`test_browser_integration`, `test_e2e_pipeline`) hit real services and skip without `ANTHROPIC_API_KEY`.

## Architecture

**Voice loop:** browser captures speech (Chrome Web Speech API in `frontend/src/voice.ts`) → transcript sent over WebSocket (`/ws/voice`) → `server.py` classifies intent → generates a reply (Claude Haiku) → Fish Audio TTS → audio streamed back as binary frames → `orb.ts` deforms the Three.js particle orb in time with the audio. JSON control messages and binary audio share the same socket.

**`server.py` (~2700 lines) is the monolith** — WebSocket handler, intent classification, LLM calls, the action system, TTS, and all REST endpoints (`/api/health`, `/api/tasks`, `/api/projects`, `/api/usage`, `/api/settings/*`, `/api/restart`, `/api/fix-self`). Most behavior changes start here. It composes the supporting modules below rather than the other way around.

**Models** (hardcoded in `server.py`): fast path is `claude-haiku-4-5-20251001` (voice replies, classification); deep path is Opus (research). Grep for the literals before changing them.

**Action system** — the LLM emits inline `[ACTION:NAME]` tags that `server.py` parses and dispatches. Tags include `BUILD`, `BROWSE`, `RESEARCH`, `PROMPT_PROJECT`, `ADD_TASK`, `COMPLETE_TASK`, `ADD_NOTE`/`CREATE_NOTE`/`READ_NOTE`, `REMEMBER`, `SCREEN`, `OPEN_TERMINAL`. `BUILD`/`RESEARCH`/`PROMPT_PROJECT` spawn `claude` subprocesses; the rest run AppleScript or local logic.

**macOS integration layer** — all native access is AppleScript (no OAuth): `calendar_access.py`, `mail_access.py` (READ-ONLY by design — do not add write operations), `notes_access.py`, `actions.py` (Terminal/Chrome/Claude Code), `screen.py` (active windows + screenshots). `helpers/` holds a compiled Swift `calendar_helper` binary and alternate calendar fetchers. Any string interpolated into AppleScript must go through `applescript_escape()` (`actions.py`) — there's a dedicated injection test for it.

**Persistence** — single SQLite DB at `data/jarvis.db` (WAL mode), shared by `memory.py` (facts/preferences, FTS5 full-text search), `dispatch_registry.py` (active/recent project builds so JARVIS knows what it's "working on"), and `tracking.py` (success metrics). LLM/cost usage is appended to `data/usage_log.jsonl`. There are no migrations — tables self-create via `CREATE TABLE IF NOT EXISTS` on first use.

**Self-improvement subsystem** — a feedback loop around the Claude Code dispatch path, mostly independent of the voice loop:
- `planner.py` — conversational planning before spawning a build (asks clarifying questions; `BYPASS_PHRASES` skips planning).
- `conversation.py` — multi-turn planning sessions, tracks decisions across exchanges.
- `templates.py` / `ab_testing.py` / `evolution.py` — prompt templates with A/B-tested versions; `evolution.py` analyzes failures and generates improved template versions.
- `qa.py` — spawns `claude -p` to verify completed task output and auto-retries on failure.
- `learning.py` / `suggestions.py` — learn request patterns to pre-load context and offer one heuristic follow-up suggestion per completed task.
- `tracking.py` records the success/failure data the above consume.
- `work_mode.py` — persistent `claude -p` sessions tied to a project dir (resumes via `--continue`).
- `monitor.py` — standalone log-watcher that critiques conversation quality; run alongside the server, not imported by it.

**Frontend** (`frontend/src/`) — `main.ts` is the state machine, `voice.ts` does speech-in/audio-out, `orb.ts` the Three.js visualization, `ws.ts` the socket, `settings.ts` the settings UI. Plain Vite + TS, only runtime dep is `three`.

## Conventions
- **Personality:** British butler — dry wit, economy of language. Voice responses are max 1–2 sentences.
- **Never modify/delete user data** in Mail, Calendar, or Notes beyond what already exists (Mail is strictly read-only).
- **No telemetry/analytics**, and no new external services beyond Anthropic and Fish Audio.
- Avoid adding dependencies unless necessary.
- `server.py` is a known large monolith; refactoring into modules is welcome but must not break the voice loop.

## Environment Variables
- `ANTHROPIC_API_KEY` (required), `FISH_API_KEY` (required), `FISH_VOICE_ID` (optional voice model), `USER_NAME` (optional), `CALENDAR_ACCOUNTS` (optional, comma-separated; empty = auto-discover).
