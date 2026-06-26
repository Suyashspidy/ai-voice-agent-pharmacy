# AI Voice Agent — Pharmacy

A real-time voice assistant that answers phone calls for a pharmacy. It bridges
[Twilio Media Streams](https://www.twilio.com/docs/voice/media-streams) to the
[Deepgram Voice Agent API](https://developers.deepgram.com/docs/voice-agent),
letting callers talk to an AI that can look up drug information, place orders, and
check order status — all over a regular phone call.

## How it works

```
 Caller ──phone──> Twilio ──WebSocket (mulaw 8kHz)──> main.py ──WebSocket──> Deepgram Voice Agent
                                                          │                        │
                                                          │                  STT → LLM → TTS
                                                          └── function calls ──────┘
                                                              (pharmacy_functions.py)
```

`main.py` runs a WebSocket server that Twilio connects to when a call comes in. It
relays caller audio to Deepgram's Voice Agent and streams the agent's spoken
responses back to the caller. When the agent decides to call a tool (e.g.
"place an order"), the request is dispatched to the matching Python function and the
result is returned to the agent so it can speak the answer.

The agent pipeline is configured in `config.json`:

- **Speech-to-text:** Deepgram `nova-3`
- **Reasoning:** OpenAI `gpt-4o-mini` (via Deepgram's managed LLM)
- **Text-to-speech:** Deepgram `aura-2-thalia-en`
- **Audio format:** mulaw / 8 kHz (telephony standard)

It also handles **barge-in** — if the caller starts talking while the agent is
speaking, the agent's audio is cleared so the caller isn't talked over.

## Capabilities

The assistant exposes three functions (see `pharmacy_functions.py`):

| Function | What it does |
| --- | --- |
| `get_drug_info` | Returns name, description, price, and stock quantity for a drug |
| `place_order` | Creates an order for a customer with a predefined quantity |
| `lookup_order` | Retrieves an existing order by its ID |

Drug and order data are stored **in memory** (`DRUG_DB` / `ORDERS_DB`) for
demonstration purposes — orders are lost when the server restarts. The catalog
ships with 10 common medications (aspirin, ibuprofen, metformin, lisinopril, etc.).

## Requirements

- Python **3.13+**
- A [Deepgram API key](https://console.deepgram.com/) with Voice Agent access
- A [Twilio](https://www.twilio.com/) phone number (for live phone calls)
- A tunnel such as [ngrok](https://ngrok.com/) to expose the local server to Twilio

## Setup

This project uses [uv](https://github.com/astral-sh/uv) for dependency management.

```bash
# Install dependencies
uv sync

# Create your .env file with your Deepgram API key
echo 'DEEPGRAM_API_KEY="your_deepgram_api_key_here"' > .env
```

> **Note:** `.env` is git-ignored — never commit your API key.

## Running

1. **Start the server** (listens on `localhost:5000`):

   ```bash
   uv run main.py
   ```

2. **Expose it** with a tunnel so Twilio can reach it:

   ```bash
   ngrok http 5000
   ```

3. **Point Twilio at the WebSocket.** Configure your Twilio number's voice webhook
   to return TwiML that streams the call to your tunnel, for example:

   ```xml
   <Response>
     <Connect>
       <Stream url="wss://YOUR-NGROK-SUBDOMAIN.ngrok.io" />
     </Connect>
   </Response>
   ```

4. **Call your Twilio number** and talk to the pharmacy assistant.

## Project structure

```
.
├── main.py                 # WebSocket server bridging Twilio <-> Deepgram
├── pharmacy_functions.py   # Drug catalog, order store, and tool functions
├── config.json             # Deepgram Voice Agent configuration & prompt
├── pyproject.toml          # Project metadata and dependencies
└── .env                    # DEEPGRAM_API_KEY (not committed)
```

## Notes & limitations

- Data is **in-memory only** — not a real database; restarting clears all orders.
- Intended as a **demo / proof of concept**, not production-ready (no auth,
  persistence, error recovery, or PII handling for a real pharmacy).
