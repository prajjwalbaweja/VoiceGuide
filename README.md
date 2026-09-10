# VoiceGuide — Voice Assistant for Visually Impaired Users

Real-time voice AI that reads phone screens and guides navigation using natural conversation.

## Architecture

```
Repository Root
├── .env                                # API keys (never committed)
├── voice-agent/                        # Core voice pipeline (Prajjwal)
│   ├── agent.py                        # LiveKit Agent entrypoint (run this)
│   ├── requirements.txt                # Python dependencies
│   ├── stt/
│   │   ├── __init__.py
│   │   └── assemblyai_stt.py           # STT factory (AssemblyAI real-time streaming)
│   ├── llm/
│   │   ├── __init__.py
│   │   └── prompt_builder.py           # screen_content + unheard → system prompt
│   ├── interruption/
│   │   ├── __init__.py
│   │   └── vad_handler.py              # interrupt() + cancel_pending_llm_call()
│   └── state/
│       ├── __init__.py
│       └── session_state.py            # "kaha tak bola / kya unheard hai" tracking
├── rime-integration/                   # Rime TTS module (Akshita)
│   └── README.md
├── frontend/                           # Web UI + screen extraction (Abhayraj)
│   └── README.md
├── testing-evidence/                   # Automated tests + latency dashboard (Nihal)
│   └── README.md
├── .gitignore
├── AGENTS.md
├── README.md
└── pyrightconfig.json
```

## Tech Stack

| Component | Provider |
|-----------|----------|
| Voice Activity Detection (VAD) | Silero (via `livekit-plugins-silero`) |
| Speech-to-Text (STT) | AssemblyAI (real-time streaming) |
| LLM | Groq — Qwen 3.8 27B |
| Text-to-Speech (TTS) | Rime (`mist` model, via `livekit-plugins-rime`) |
| Real-time transport | LiveKit (Agents SDK v1.8.0, Python) |
| Frontend | HTML / CSS / JS web app |

## Setup

### 1. Install dependencies

```bash
cd voice-agent
pip install -r requirements.txt
```

### 2. Configure environment

Create `.env` in the repo root (`vg/`) with:

```env
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
LIVEKIT_URL=wss://your-project.livekit.cloud
GROQ_API_KEY=gsk_...
ASSEMBLYAI_API_KEY=...
RIME_API_KEY=...
```

| Key | Where to get it |
|-----|-----------------|
| `LIVEKIT_*` | [livekit.io/cloud](https://livekit.io/cloud) |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) |
| `ASSEMBLYAI_API_KEY` | [assemblyai.com](https://www.assemblyai.com/) |
| `RIME_API_KEY` | [rime.ai](https://rime.ai) |

### 3. Run the agent

```bash
cd voice-agent

# Development mode (auto-reload, connects to LiveKit Cloud)
python agent.py dev

# Production mode
python agent.py start
```

## How It Works

### LiveKit Agents v1.8.0 API

The agent uses the latest LiveKit Agents SDK concepts:

| Concept | Purpose |
|---------|---------|
| `AgentServer` | Worker process — registers `prewarm` for one-time model loading |
| `Agent` | Personality / config — holds instructions, STT, LLM, TTS, VAD |
| `AgentSession` | Runtime session — manages the live voice pipeline per room |

### Data Flow

```
Phone App  ──data-channel──►  LiveKit Room  ──►  VoiceGuide Agent
  (screen_content JSON)                            │
                                                   ├─ STT: AssemblyAI transcribes user speech
                                                   ├─ LLM: Groq (Qwen 3.8 27B) reasons about screen
                                                   ├─ TTS: Rime (mist) speaks the response
                                                   └─ VAD: Silero detects interrupts
```

### Screen Content Protocol

The companion app sends screen accessibility data via LiveKit data-channel as JSON:

```json
{
  "screen_content": {
    "app_name": "WhatsApp",
    "activity": "ChatActivity",
    "elements": [
      {
        "type": "TextView",
        "text": "Hey, are you free today?",
        "content_description": "",
        "clickable": false
      },
      {
        "type": "Button",
        "text": "Send",
        "content_description": "Send message",
        "clickable": true
      }
    ],
    "notifications": ["2 new messages from Mom"]
  }
}
```

### Interrupt Logic ("kaha tak bola / kya unheard hai")

1. Agent starts speaking a response via TTS
2. User interrupts mid-speech (detected by Silero VAD)
3. `VADHandler.interrupt()` fires:
   - Saves the **unheard portion** of the response in `SessionState`
   - Cancels any pending LLM generation
4. On the next LLM call, `PromptBuilder` injects the unheard text into the system prompt
5. The LLM naturally weaves any still-relevant unheard info into its next response

### Key Modules

| Module | File | Responsibility |
|--------|------|---------------|
| **STT** | `stt/assemblyai_stt.py` | Factory that returns a configured `assemblyai.STT` instance |
| **Prompt Builder** | `llm/prompt_builder.py` | Builds system prompt with screen context + unheard recovery |
| **VAD Handler** | `interruption/vad_handler.py` | Application-level interrupt logic on top of Silero |
| **Session State** | `state/session_state.py` | Tracks screen content, conversation history, unheard segments |

## Team

| Member | Role |
|--------|------|
| **Prajjwal** | Voice Pipeline / Backend Lead — LiveKit Agent, STT, VAD + interrupt logic, state tracking |
| **Akshita** | Rime TTS Integration — streaming TTS, voice selection, prompt tuning |
| **Abhayraj** | Frontend — Web UI, DOM/accessibility extraction, LLM prompt design |
| **Nihal** | Testing & Evidence — automated interrupt tests, latency dashboard |
