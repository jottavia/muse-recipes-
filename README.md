# muse-recipes

Reusable patterns for building with the Muse agent harness. Each recipe is
a self-contained directory with a `RECIPE.md` explaining the architecture,
a reference implementation where one exists, and the lessons learned
building it.

## Recipes

- [Voice Agent](voice-agent/RECIPE.md) — give an AI assistant a telephone
  voice: Twilio + local Whisper STT + LLM brain + TTS, with latency and
  cost analysis for three setups (async agent brain, fast batch, streaming).

## Contributing

A recipe should be something actually built, not just designed. Include
what worked, what didn't, and the numbers (latency, cost) you measured.
