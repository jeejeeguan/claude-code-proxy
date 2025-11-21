# Repository Guidelines

## Project Structure & Module Organization
The proxy lives in `server.py`, a FastAPI app translating Anthropic payloads to LiteLLM targets (OpenAI, Gemini, or native Anthropic). `pyproject.toml` plus `uv.lock` pin runtime dependencies; manage them with `uv`. Integration helpers, streaming fixtures, and tool definitions sit in `tests.py`. Containerized deployments rely on `Dockerfile`, while `README.md` and `.env.example` document environment variables such as `PREFERRED_PROVIDER`, `BIG_MODEL`, and Vertex AI settings. Keep auxiliary assets (e.g., `pic.png`, sample configs) in the repo root unless they grow large enough for a dedicated `assets/` directory.

## Build, Test, and Development Commands
`uv sync` installs the pinned toolchain (Python ≥3.10). Launch the dev server with `uv run uvicorn server:app --host 0.0.0.0 --port 8082 --reload` to enable hot reloads for Claude clients. To verify Docker parity, run `docker build -t claude-code-proxy .` followed by `docker run --env-file .env -p 8082:8082 claude-code-proxy`. Quick smoke tests can target specific providers by exporting `OPENAI_API_KEY`, `GEMINI_API_KEY`, or enabling `USE_VERTEX_AUTH=true` before running the server.

## Coding Style & Naming Conventions
Use 4-space indentation, type hints, and `pydantic` models for every request/response shape; extend existing `ContentBlock*` classes instead of ad-hoc dicts. Keep module-level constants in ALL_CAPS, environment keys in `snake_case` variables, and HTTP handler functions named `verb_noun` (e.g., `post_messages`). Favor FastAPI dependency injection rather than global state where possible, but centralized logging stays in `server.py` for visibility. Logging should go through the configured `logger` with concise, structured messages (e.g., `logger.info("MODEL MAPPING %s -> %s", requested, mapped)`).

## Testing Guidelines
`tests.py` provides the canonical harness; run `uv run python tests.py` for the full suite or narrow scopes like `uv run python tests.py --simple` (non-streaming) and `--no-streaming` when CI lacks SSE support. Each scenario name doubles as documentation—mirror that naming style when adding new coverage. Since tests hit live providers, ensure environment variables reference mock keys or sandbox projects before committing. Aim to exercise new routes via both streaming and non-streaming flows; when vertex-specific logic changes, add a short `TEST_SCENARIOS` entry covering ADC paths.

## Commit & Pull Request Guidelines
Follow the existing Chinese commit style with descriptive prefixes and multi-line bodies, for example:
`git commit -m "feat: 扩展Gemini模型映射" -m "- 支持gemini-2.5-pro-preview"`  
Reference related issues in the second message when applicable. PRs should include: summary of model/provider impact, testing evidence (`python tests.py --simple`, Docker run logs), configuration notes, and screenshots for UX-facing tweaks (e.g., new logs). Use `gh pr create --title "feat: ..." --base main --head <branch>` and attach `.env` diffs only when necessary.

## Security & Configuration Tips
Never commit `.env` or API keys; rely on `.env.example` placeholders. When running locally, prefer `USE_VERTEX_AUTH=true` with ADC so credentials stay in gcloud profiles. Validate new environment flags with graceful fallbacks—if a provider key is missing, raise `HTTPException(status_code=400, detail="...")` instead of exiting. Rotate logged UUIDs regularly and avoid logging raw user prompts outside debug builds.
