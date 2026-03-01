# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Workshop repository for building real-time voice/video AI chatbots using **Pipecat** (framework) + **Google Gemini Multimodal Live API**. Transport layer is **Daily** (video rooms) or **Twilio** (phone). All bots use **Silero VAD** (Voice Activity Detection) to detect when users start/stop speaking.

## Repository Structure

- **`quickstart/`** — Minimal Gemini bot with function calling. Entry point for learning. Runs standalone with `DailyTransport` and opens a browser at `http://localhost:7860`.
- **`starters/simple-chatbot/`** — Production-style client/server architecture:
  - `server/` — FastAPI server that spawns bot subprocesses per Daily room (max 1 bot per room). Includes animated robot avatar using sprite frames.
  - `client/` — Six client implementations: Daily Prebuilt, vanilla JS, React, React Native, iOS, Android. All connect via RTVI protocol to the server's `POST /connect` endpoint.
- **`starters/twilio-chatbot/`** — Phone integration via Twilio WebSocket. FastAPI server with TwiML template. Requires ngrok for local development.

## Running the Projects

### Quickstart
```bash
cd quickstart
python3 -m venv env && source env/bin/activate
pip install -r requirements.txt
cp env.example .env   # add GOOGLE_API_KEY
python gemini-bot.py  # opens http://localhost:7860
```

### Simple-Chatbot Server
```bash
cd starters/simple-chatbot/server
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp env.example .env   # add GOOGLE_API_KEY and DAILY_API_KEY
python server.py      # serves at http://localhost:7860
```

Then run any client from `starters/simple-chatbot/client/` (JS/React clients use `npm install && npm run dev` on port 5173).

### Twilio-Chatbot
```bash
cd starters/twilio-chatbot
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp env.example .env   # add GOOGLE_API_KEY
python server.py      # serves on port 8765, needs ngrok tunnel
```

## Architecture: Pipecat Pipeline

Every bot follows the same pipeline pattern — a linear chain of frame processors:

```
Transport Input → Context Aggregator (user) → LLM Service → Transport Output → Context Aggregator (assistant)
```

- **Transport** handles audio/video I/O (DailyTransport or FastAPIWebsocketTransport)
- **Context Aggregator** manages conversation history via `OpenAILLMContext` (name is historical; works with any LLM)
- **LLM Service** is `GeminiMultimodalLiveLLMService` configured with voice, system instructions, and tools
- Custom `FrameProcessor` subclasses can be inserted into the pipeline (e.g., `TalkingAnimation` for sprite-based avatar)

### Key Event Handlers
- `on_client_connected` — queues the initial context frame to start the conversation
- `on_client_disconnected` — cancels the pipeline task

### Function Calling
Tools are defined with `FunctionSchema` and grouped into `ToolsSchema`. Handlers are registered with `llm.register_function("name", handler)`. Gemini-specific tools (like Google Search) use `custom_tools={AdapterType.GEMINI: [...]}`.

## Key Configuration

- **Gemini voices**: Aoede, Charon, Fenrir, Kore, Puck
- **VAD**: `SileroVADAnalyzer(params=VADParams(stop_secs=0.5))` — 0.5s silence triggers end-of-speech
- **Audio sample rates**: 24kHz for Daily transport, 8kHz for Twilio (phone quality)
- **Environment variables**: `GOOGLE_API_KEY` (required everywhere), `DAILY_API_KEY` (quickstart/simple-chatbot), `TWILIO_ACCOUNT_SID`/`TWILIO_AUTH_TOKEN` (twilio-chatbot)

## Dependencies

Python extras on `pipecat-ai` control which integrations are installed:
- `pipecat-ai[daily,google,silero]` — Daily transport + Gemini + Silero VAD
- `pipecat-ai[google,silero]` — Gemini + Silero VAD only (Twilio chatbot uses FastAPI WebSocket instead of Daily)

Client SDKs: `@pipecat-ai/client-js`, `@pipecat-ai/client-react`, `@pipecat-ai/daily-transport`

## Prerequisites

- Python 3.10+
- Linux, macOS, or WSL
- Google Gemini API key from https://aistudio.google.com/app/apikey
