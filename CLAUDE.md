# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an API proxy server that translates between Anthropic's Claude API format and OpenAI/Google Gemini APIs via LiteLLM. It enables using Anthropic clients (like Claude Code CLI) with OpenAI or Gemini models while maintaining full compatibility.

**Tech Stack**: Python 3.10+, FastAPI, LiteLLM, httpx, Pydantic

## Essential Commands

### Development Setup
```bash
# Install uv package manager (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Configure environment
cp .env.example .env
# Edit .env with API keys and provider settings
```

### Running the Server
```bash
# Development mode with hot reload
uv run uvicorn server:app --host 0.0.0.0 --port 8082 --reload

# Production mode
uv run uvicorn server:app --host 0.0.0.0 --port 8082
```

### Testing
```bash
# Run all tests (proxy must be running on port 8082)
python tests.py

# Test variations
python tests.py --no-streaming    # Skip streaming tests
python tests.py --streaming-only   # Only streaming tests
python tests.py --simple          # Only simple tests
python tests.py --tools-only      # Only tool-use tests
```

## Architecture & Code Structure

### Core Components

**server.py** - Main proxy implementation with key functions:
- `convert_anthropic_to_litellm()` - Translates Anthropic format to LiteLLM/OpenAI format
- `convert_litellm_to_anthropic()` - Translates responses back to Anthropic format
- `handle_streaming()` - Manages SSE streaming responses
- `clean_gemini_schema()` - Removes unsupported fields for Gemini compatibility

**API Endpoints**:
- `/v1/messages` - Main message completion endpoint
- `/v1/messages/count_tokens` - Token counting endpoint
- `/` - Health check

### Model Mapping Logic

The proxy maps Claude model names to provider-specific models based on configuration:

1. **Environment Variables**:
   - `PREFERRED_PROVIDER` - "openai" (default) or "google"
   - `BIG_MODEL` - Maps claude-3-sonnet (defaults: gpt-4.1 or gemini-2.5-pro-preview-03-25)
   - `SMALL_MODEL` - Maps claude-3-haiku (defaults: gpt-4.1-mini or gemini-2.0-flash)

2. **Mapping Process**:
   - Checks if requested model is already a known provider model
   - Maps haiku/sonnet to configured models with appropriate prefix
   - Adds provider prefix (openai/ or gemini/) automatically
   - Falls back to OpenAI if Gemini model not recognized

### Request/Response Flow

1. Receives Anthropic API format request
2. Validates and maps model names
3. Converts message format (including tool calls)
4. Sends to LiteLLM with appropriate provider
5. Converts response back to Anthropic format
6. Handles both streaming (SSE) and non-streaming responses

### Error Handling

- Comprehensive error logging with full tracebacks
- Preserves LiteLLM error attributes in responses
- Graceful fallback for provider-specific issues
- Detailed request/response dumps on errors

## Development Tips

### Debugging
- Change logging level to `logging.DEBUG` in server.py:23 for verbose output
- Monitor colored console output for model mapping and request details
- Check logs for `📋 MODEL VALIDATION` and `📌 MODEL MAPPING` messages

### Testing Changes
- Always run full test suite after modifications
- Test both streaming and non-streaming modes
- Verify model mapping with different PREFERRED_PROVIDER settings
- Check tool/function calling translation

### Environment Configuration
Essential `.env` variables:
- `ANTHROPIC_API_KEY` - Only if proxying to Anthropic (optional)
- `OPENAI_API_KEY` - Required for OpenAI models
- `GEMINI_API_KEY` - Required for Google/Gemini models
- `PREFERRED_PROVIDER` - Controls default model mapping
- `BIG_MODEL` / `SMALL_MODEL` - Custom model mappings