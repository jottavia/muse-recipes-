# Voice Agent Recipe: Twilio + Whisper + LLM + TTS

A pattern for giving an AI agent a telephone voice. Two variants: the
agent itself as the brain (slow, simple) and a fast cloud model as the
brain (real-time, more moving parts).

## What it does

A caller dials a Twilio number and has a spoken conversation. Speech is
transcribed locally, a language model thinks of a reply, and the reply is
spoken back on the call.

## Architecture

```
Caller --(audio)--> Twilio --(Media Streams wss)--> Voice Server
                                                          |
Voice Server --(mulaw 8kHz)--> Whisper (STT, local) --> text
                                                          |
                        text --> Brain (LLM) --> reply text
                                                          |
Voice Server <--(mulaw 8kHz)-- TTS <-- reply text
                                                          |
Caller <--(audio)-- Twilio <--(Media Streams wss)-- Voice Server
```

## Components

### Telephony: Twilio

- One Twilio number with Voice enabled. Outbound calls need no public
  infrastructure: the REST API accepts inline TwiML.
- Two-way audio needs **Twilio Media Streams**, which requires a publicly
  reachable `wss://` endpoint. A sandboxed VM without inbound networking
  cannot host this; use a small public server (e.g. Oracle Cloud free tier).

Outbound speaking (no public endpoint needed):

```python
# POST /2010-04-01/Accounts/{SID}/Calls.json
{"To": "+15551234567", "From": "+15559876543",
 "Twiml": "<Response><Say>Hello.</Say></Response>"}
```

Two-way audio (public `wss://` required):

```xml
<Response>
  <Stream url="wss://voice.example.com/stream"/>
</Response>
```

Media Streams sends JSON frames: `connected`, `start`, `media` (base64
mulaw 8kHz), `stop`. Send `media` frames back to play audio to the caller.

### Ears: Whisper (local)

`faster-whisper`, `tiny` model, int8 on CPU. Tested: 2.8s of audio
transcribed in 1.2s on 2 vCPUs. Free, private, no API calls.

```python
from faster_whisper import WhisperModel
m = WhisperModel("tiny", device="cpu", compute_type="int8")
segs, _ = m.transcribe(audio_float32_16khz, beam_size=1)
text = " ".join(s.text for s in segs).strip()
```

Feed it 16kHz mono float32. Twilio sends 8kHz mulaw; convert with
`audioop.ulaw2lin` then resample.

### Mouth: TTS

Any TTS that can output audio convertible to 8kHz mulaw. Tested path:
cloud TTS to WAV, strip ID3 header, `audioop.ratecv` to 8kHz,
`audioop.lin2ulaw`. Male voices available; voice is swappable per call.

### Brain, variant A: Agent (the assistant itself)

Pipe the transcript to the agent, take its reply, speak it.

**How:** the voice server writes each transcript to a queue. A scheduled
job (or the agent's own loop) picks it up, the assistant responds in chat or
writes the reply to a file, the server speaks it on the call.

**Latency:** 20-30 seconds per turn. The assistant's inference loop is built
for thoughtful tool-using work, not real-time chat. Each turn also needs
a wake-up (poll interval).

**When it fits:** voicemail-style exchanges, async voice memos, anything
where the caller tolerates a long pause. It does not fit a live
conversation.

**When it doesn't:** real-time back-and-forth. The caller will hang up.

### Brain, variant B: fast cloud model

A small model behind a fast API (e.g. via OpenRouter: GPT-4o-mini, Groq
Llama). The voice server calls it directly over HTTPS.

```python
# pseudo-code
reply = llm.chat(
    model="openai/gpt-4o-mini",
    messages=[{"role": "system", "content": SYSTEM_PROMPT},
              *history, {"role": "user", "content": transcript}],
    max_tokens=60,
)
```

**Latency:** 1-2s for the brain. Total per turn: 3-5s (STT ~1s, brain
~1-2s, TTS ~1-2s). Noticeable pause, workable.

**To reach ~1s** (human-like): stream everything. Streaming STT
(Deepgram), streaming LLM tokens (Groq), streaming TTS — each stage starts
before the previous finishes. More engineering, more cost, audio leaves
your infrastructure.

## Latency and cost cheat sheet

| Setup | Per-turn latency | Cost per 5-min call |
|---|---|---|
| Agent as brain (async) | 20-30s | ~$0.05 (Twilio only) |
| Fast brain, batch pipeline | 3-5s | ~$0.05-0.15 |
| Fast brain, streaming pipeline | ~1-1.5s | ~$0.50-1.00 |

Twilio voice is ~$0.0085/min; Media Streams ~$0.002/min. STT via local
Whisper is free. A cloud LLM brain is fractions of a cent per call at
small-model prices.

## Infrastructure

The voice server needs a public IP and modest CPU. Oracle Cloud's Always
Free tier fits: 2 Ampere ARM OCPUs + 12GB RAM (as of mid-2026; was 4/24GB
before Oracle halved it). Keep a heartbeat running — Oracle reclaims
instances idle under 20% for 7 days.

## Reference implementation

`server.py` (in this repo's companion workspace): asyncio websocket
server, Twilio Media Streams protocol, Whisper transcription, pluggable
brain (`stub` / cloud LLM), TTS to mulaw. Written against the sandbox;
needs a public host to run live.

## Lessons

- Twilio's `<Say>` and inline TwiML make one-way voice trivial. Two-way
  is where the work is.
- The sandbox firewall blocked Cloudflare Tunnel (QUIC/UDP and TCP 7844
  refused). Don't assume you can get a public endpoint from anywhere.
- Local LLM inference on 2 vCPUs is ~60s/reply for a 0.5B model. The
  brain must be a cloud API for anything real-time.
- faster-whisper's bundled HTTP client chokes on bracketed IPv6
  `no_proxy` entries; unset `no_proxy`/`NO_PROXY` or download model files
  with curl directly.
- TTS WAV output may carry an ID3 header; strip it before decoding.
