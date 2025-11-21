# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Anthropic API proxy server that allows Anthropic clients (like Claude Code) to use Gemini, OpenAI, or direct Anthropic backends via LiteLLM. The proxy receives requests in Anthropic's API format, translates them using LiteLLM, and forwards them to the appropriate backend.

## Key Commands

### Running the Server

```bash
# Install dependencies (uses uv - the Python package installer)
uv pip install -e .

# Run the proxy server (development mode with auto-reload)
uv run uvicorn server:app --host 0.0.0.0 --port 8082 --reload

# Run without reload for production
uv run uvicorn server:app --host 0.0.0.0 --port 8082
```

### Running Tests

```bash
# Run all tests (requires ANTHROPIC_API_KEY in .env)
uv run python tests.py

# Run only simple tests (no tool usage)
uv run python tests.py --simple

# Run only streaming tests
uv run python tests.py --streaming-only

# Run only tool-related tests
uv run python tests.py --tools-only

# Skip streaming tests
uv run python tests.py --no-streaming
```

### Building and Publishing

```bash
# The project uses Docker for containerization
# Build image locally
docker build -t claude-code-proxy .

# Or use docker compose with the provided configuration
docker compose up
```

GitHub Actions automatically publishes to `ghcr.io/${{ github.repository_owner }}/${{ github.event.repository.name }}` on push to main branch.

## Architecture

### Core Components

**server.py** - Main FastAPI application with two endpoints:
- `POST /v1/messages` - Primary endpoint for chat completions (streaming and non-streaming)
- `POST /v1/messages/count_tokens` - Token counting endpoint

**tests.py** - Comprehensive test suite that:
- Compares proxy responses against direct Anthropic API
- Tests streaming and non-streaming scenarios
- Validates tool use, multi-turn conversations, and content blocks

### Request Flow

1. Client sends Anthropic-format request to proxy
2. Pydantic models validate and transform the request (MessagesRequest, TokenCountRequest)
3. Model mapping logic converts model names based on PREFERRED_PROVIDER:
   - `haiku` → `openai/gpt-4.1-mini` or `gemini/gemini-2.5-flash`
   - `sonnet` → `openai/gpt-4.1` or `gemini/gemini-2.5-pro`
   - Direct model names get appropriate prefixes (`openai/`, `gemini/`, `anthropic/`)
4. `convert_anthropic_to_litellm()` transforms request to OpenAI format
5. LiteLLM forwards to appropriate backend based on model prefix
6. `convert_litellm_to_anthropic()` converts response back to Anthropic format
7. Streaming responses use Server-Sent Events (SSE) format with proper Anthropic event types

### Environment Configuration

Key environment variables (see .env.example):
- `ANTHROPIC_API_KEY` - For direct Anthropic proxy mode
- `OPENAI_API_KEY` - Required for OpenAI backend
- `GEMINI_API_KEY` - Required for Gemini AI Studio backend
- `VERTEX_PROJECT`/`VERTEX_LOCATION` - For Vertex AI with ADC
- `USE_VERTEX_AUTH=true` - Toggle between API key and ADC
- `PREFERRED_PROVIDER` - `openai` (default), `google`, or `anthropic`
- `BIG_MODEL`/`SMALL_MODEL` - Override default model mappings

### Important Implementation Details

**Model Mapping Logic** (server.py:200-265):
- Validates and transforms model names using Pydantic field_validator
- Handles provider prefixes and model name remapping
- Stores original model name for logging

**Content Block Conversion** (server.py:417-636):
- Converts Anthropic content blocks (text, image, tool_use, tool_result) to OpenAI format
- Special handling for tool_result blocks in user messages (converts to plain text)
- For OpenAI models, converts lists to simple strings
- Cap max_tokens at 16384 for non-Anthropic models

**Streaming Response Handling** (server.py:833-1094):
- Converts OpenAI streaming format to Anthropic SSE format
- Handles text deltas and tool call streaming
- Emits required event types: message_start, content_block_start, content_block_delta, content_block_stop, message_delta, message_stop

**Error Handling**: Extensive try/except blocks with detailed logging and graceful fallbacks for all conversion operations.

## Testing Approach

The test suite compares proxy responses against direct Anthropic API calls. Tests cover:
- Simple text responses
- Tool use (calculator, weather, search)
- Multi-turn conversations
- Content blocks
- Streaming (text and tool use)

When modifying request/response conversion logic, always run tests to ensure compatibility.

## Common Development Tasks

### Adding New Model Support

1. Add model name to OPENAI_MODELS or GEMINI_MODELS list (server.py:104-123)
2. Update model mapping logic if needed (field_validator in MessagesRequest)
3. Update .env.example with new default options
4. Update README.md model lists

### Modifying Request/Response Format

1. Update Pydantic models (MessagesRequest, MessagesResponse, etc.)
2. Modify `convert_anthropic_to_litellm()` for request transformation
3. Modify `convert_litellm_to_anthropic()` for response transformation
4. Update streaming handler if needed
5. Run full test suite to verify compatibility

### Debugging Provider-Specific Issues

- Set logging level to DEBUG in server.py:23
- Check model mapping logs (look for "MODEL MAPPING" messages)
- Verify API keys and environment configuration
- Use `--simple` test flag to isolate tool-related issues

## Dependencies

Managed via pyproject.toml using uv:
- fastapi[standard] - Web framework
- uvicorn - ASGI server
- litellm - API translation layer
- pydantic - Data validation
- python-dotenv - Environment management
- google-auth, google-cloud-aiplatform - Vertex AI support

Python 3.10+ required.
- 当前项目是我从 git@github.com:1rgs/claude-code-proxy.git fork 的项目, fork 后我的 public 项目地址为 git@github.com:jeejeeguan/claude-code-proxy.git , 本地工作目录同时设置了我的 github 远端和原始项目为 upstream.