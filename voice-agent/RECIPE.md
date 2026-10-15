# Voice Agent Recipe: Twilio + Whisper + LLM + TTS

A pattern for giving an AI agent a telephone voice. Three variants:
turn-based with the agent as brain (slow, no public server needed),
Twilio ConversationRelay with a fast cloud brain (recommended for
real-time), and raw Media Streams (most control, most work).

## What it does

A caller dials a Twilio number and has a spoken conversation. Speech is
transcribed, a language model thinks of a reply, and the reply is spoken
back on the call.

## Variant 1: Turn-based (agent as brain, no public server)

The simplest thing that works. No websocket, no public endpoint — just
the Twilio REST API from any machine.

```
Call --> <Say>prompt</Say> --> <Record maxLength=10> --> hangup
                                                      |
Poll Recordings API --> download wav --> Whisper --> text --> agent
                                                                      |
                                              agent reply --> next call
                                              (or redirect live call)
```

Tested end-to-end 2026-10-15: outbound call placed, prompt spoken,
10s reply recorded, audio downloaded, transcribed with local Whisper
tiny, all from a sandboxed VM with no inbound networking.

**Latency:** 20-40s per turn (polling + transcription + agent thinking).
Voicemail-like, not conversational.

**To keep a dialog going** on one call instead of hanging up, redirect
the live call with new TwiML before `<Record>` ends:

```
POST /2010-04-01/Accounts/{SID}/Calls/{CallSid}.json
{"Twiml": "<Response><Say>...</Say><Record maxLength=\"10\"/></Response>"}
```

### Voice

Twilio's `<Say>` defaults to a female voice. For a male voice at zero
extra latency, use an Amazon Polly voice:

```xml
<Say voice="Polly.Matthew">Hello.</Say>
```

No audio hosting, no extra delay — it's still Twilio's infrastructure.

### Transcription accuracy

Whisper `tiny` on phone audio is fast but lossy. Real example:
said "why are you still speaking in a woman's voice", heard as
"how come you're still using one of the boys". For anything that
matters, use `base` or a cloud STT.

## Variant 2: ConversationRelay (recommended for real-time)

Twilio's managed facility for voice AI. Twilio handles STT, TTS, and
barge-in; your server just exchanges **text** over a websocket.

```xml
<Response>
  <Connect>
    <ConversationRelay url="wss://your-server/ws"/>
  </Connect>
</Response>
```

WebSocket protocol (JSON): Twilio sends `setup`, `prompt` (transcribed
speech), `interrupt`, `dtmf`. You reply with
`{"type": "text", "token": "...", "last": true}` and Twilio speaks it.
Stream tokens with `last: false` for lower latency.

Your server is trivial: receive text → call LLM → send text back. No
audio processing at all. Bring any LLM (OpenRouter works).

**Latency:** Twilio reports median under 0.5s for their side; total
~1-2s with a fast model.

**Cost:** $0.07/min on top of the call (vs $0.0044/min for raw Media
Streams). The premium buys you out of the entire audio pipeline.

## Variant 3: Raw Media Streams (most control)

Twilio streams raw mulaw 8kHz audio over `wss://`; you do STT, LLM, and
TTS yourself.

```
Caller --(audio)--> Twilio --(Media Streams wss)--> Voice Server
Voice Server --(mulaw 8kHz)--> Whisper (STT) --> text --> LLM --> TTS
Voice Server --(mulaw 8kHz)--> Twilio --(audio)--> Caller
```

Media Streams sends JSON frames: `connected`, `start`, `media` (base64
mulaw 8kHz), `stop`. Send `media` frames back to play audio.

**To reach ~1s** (human-like): stream everything — streaming STT
(Deepgram), streaming LLM tokens (Groq), streaming TTS — each stage
starting before the previous finishes. More engineering, more cost,
audio leaves your infrastructure.

## Latency and cost cheat sheet

| Setup | Per-turn latency | Cost per 5-min call |
|---|---|---|
| Turn-based, agent brain | 20-40s | ~$0.05 (Twilio only) |
| ConversationRelay + fast LLM | ~1-2s | ~$0.40 (Twilio $0.35 + LLM) |
| Raw streams, batch pipeline | 3-5s | ~$0.05-0.15 |
| Raw streams, streaming pipeline | ~1-1.5s | ~$0.50-1.00 |

Twilio voice ~$0.0085/min. A cloud LLM brain is fractions of a cent per
call at small-model prices.

## Infrastructure

Any variant needing a public `wss://` endpoint (2 and 3) needs a public
server. Options, cheapest first:

- **Oracle Cloud Always Free** — 2 ARM OCPUs + 12GB RAM, free forever.
  Best specs. Reclaims instances idle under 20% for 7 days, so keep a
  heartbeat. (Allowance halved June 2026; was 4/24GB.)
- **Google Cloud e2-micro** — 0.25 vCPU + 1GB, free forever in US
  regions. Thin, but a text-relay websocket fits.
- **Railway** — ~$5/mo Hobby. Easiest deploy (git push → public URL),
  zero ops. Free plan's $1 credit is borderline for always-on.

Not suitable: Cloudflare Workers free tier (10ms CPU/request; Durable
Objects, needed for websockets, aren't free). Sandbox VMs with no
inbound networking (Cloudflare Tunnel blocked on outbound 7844 too).

## SMS

Sending and receiving texts needs no public endpoint — the REST API
covers both directions. Poll `GET /Messages.json` for inbound.

## Reference implementation

`server.py` (companion workspace): asyncio websocket server, Twilio
Media Streams protocol, Whisper transcription, pluggable brain
(`stub` / cloud LLM), TTS to mulaw. Written against the sandbox;
needs a public host to run live.

`tw.py` turn-based commands: `ask` (call + speak + record),
`recordings`, `rec-audio`, `say` (redirect live call with reply).

## Lessons

- Twilio's `<Say>` and inline TwiML make one-way voice trivial. Two-way
  is where the work is.
- `<Say>` defaults to female; `voice="Polly.Matthew"` fixes it with no
  latency cost. (Don't wire a custom TTS + `<Play>` for turn-based —
  it adds seconds for no benefit.)
- Whisper tiny is fast but garbles phone audio. Size up the model or
  use cloud STT when accuracy matters.
- The sandbox firewall blocked Cloudflare Tunnel (QUIC/UDP and TCP
  7844 refused). Don't assume you can get a public endpoint from
  anywhere.
- Local LLM inference on 2 vCPUs is ~60s/reply for a 0.5B model. The
  brain must be a cloud API for anything real-time.
- faster-whisper's bundled HTTP client chokes on bracketed IPv6
  `no_proxy` entries; unset `no_proxy`/`NO_PROXY` before loading.
- TTS WAV output may carry an ID3 header; strip it before decoding.
