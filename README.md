# CLI Orchestrator

A local web app that runs three AI agents (Claude, GPT, Groq) as a team in a shared workspace under a coordinator agent. The user sends one prompt; the coordinator breaks it into sub-tasks and assigns each to the best agent. Every agent reads the same conversation log and can read/write files and run shell commands in the shared workspace.

## Architecture

- **Coordinator** (default: Claude Opus 4.7) — decides which agent acts next and what they should do. No file tools itself; only `assign_task` and `complete`.
- **Workers** — `claude`, `gpt`, `groq`. Each gets the same tool set: `read_file`, `write_file`, `edit_file`, `list_files`, `run_bash`, `post_message`, `finish_subtask`.
- **Shared message log** — every action, tool call, tool result, and message is appended to one log all agents (and the user) see in real time via WebSocket.
- **Workspace** — a single directory (default `./workspace`) where all file ops are sandboxed. Path traversal is blocked.

## Setup

```bash
pip install -r requirements.txt
cp .env.example .env
# edit .env and put in your real API keys
```

You need at least one API key. Agents whose key is missing are skipped. The coordinator model must match a configured key (defaults to Anthropic).

## Run

```bash
python run.py
```

Open <http://127.0.0.1:8000>.

## How it flows

1. You type a prompt in the bottom box and press Send.
2. The coordinator sees the prompt, calls `assign_task` to hand a sub-task to one of the workers.
3. That worker loops: reads files, writes/edits files, runs bash, posts messages — every action streams to the UI live.
4. The worker calls `finish_subtask`. Control returns to the coordinator.
5. The coordinator decides: assign another sub-task (same or different agent) or `complete`.
6. When `complete` is called or `MAX_TURNS` is hit, the run ends. Stop and Reset buttons are in the UI.

## Config (.env)

| Key | Default | Notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | — | enables `claude` worker |
| `OPENAI_API_KEY` | — | enables `gpt` worker |
| `GROQ_API_KEY` | — | enables `groq` worker |
| `CLAUDE_MODEL` | `claude-sonnet-4-6` | |
| `OPENAI_MODEL` | `gpt-4o` | |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | |
| `COORDINATOR_MODEL` | `claude-opus-4-7` | must be a Claude or GPT model |
| `WORKSPACE_DIR` | `./workspace` | created if missing |
| `MAX_TURNS` | `30` | coordinator turn limit per run |
| `BASH_TIMEOUT_SECONDS` | `60` | per-command timeout |

## Notes

- All three agents can edit the same files. Coordination is turn-based so there are no write races, but a later agent can still overwrite an earlier one's work. The coordinator should keep this in mind when assigning tasks.
- `run_bash` runs inside the workspace directory with the timeout above. There is no further sandbox — treat the workspace as you would any directory where you'd run untrusted scripts.
- The Groq client uses the OpenAI-compatible chat completions API under the hood, so it shares the worker implementation with GPT.
