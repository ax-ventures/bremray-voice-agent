# Gemini Multimodal Live API — Node.js Evaluation

**Date:** 2026-03-13
**Investigator:** Claude Code
**Question:** Does the Gemini Multimodal Live API support bidirectional audio streaming via WebSockets in Node.js? Should we proceed in Node.js or switch to Python?

---

## Summary / Bottom Line

**Proceed in Node.js. The `@google/genai` JavaScript SDK fully supports bidirectional audio streaming via WebSockets for the Gemini Live API.** It is actively maintained by Google, supports Node.js 20+, and includes official code examples for real-time audio. The Python SDK has a head start in maturity but the Node.js SDK is not a blocker.

---

## Findings

### 1. SDK Status

| Item | Status |
|---|---|
| Package | `@google/genai` (npm) |
| Current version | 1.44.0 (published ~March 2026) |
| Maintained by | Google (googleapis org) |
| Node.js minimum | 20+ |
| TypeScript support | Yes (first-class) |
| Live API support | ✅ Yes |

Install: `npm i @google/genai`

### 2. Bidirectional Audio Streaming — Fully Supported ✅

The SDK exposes `ai.live.connect()` which establishes a persistent WebSocket session to the Gemini Live API. This is fully bidirectional:

- **Client → Gemini:** Send raw PCM audio chunks in real-time via `session.sendRealtimeInput()`
- **Gemini → Client:** Receive audio response chunks via the `onmessage` callback

Official Node.js example (from Google's documentation):

```js
import { GoogleGenAI, Modality } from '@google/genai';

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const model = 'gemini-2.5-flash-native-audio-preview';

const session = await ai.live.connect({
  model,
  config: { responseModalities: [Modality.AUDIO] },
  callbacks: {
    onopen: () => console.log('Session open'),
    onmessage: (message) => { /* handle audio response chunks */ },
    onerror: (e) => console.error('Error:', e.message),
    onclose: (e) => console.log('Closed:', e.reason),
  },
});

// Send audio
session.sendRealtimeInput({ audio: { data: base64PcmChunk, mimeType: 'audio/pcm;rate=16000' } });
```

### 3. Audio Format Requirements

| Direction | Format | Sample Rate | Bit Depth | Encoding |
|---|---|---|---|---|
| Input (to Gemini) | Raw PCM | 16,000 Hz | 16-bit little-endian | — |
| Output (from Gemini) | Raw PCM | 24,000 Hz | 16-bit little-endian | — |

**Twilio Media Streams uses G.711 mu-law at 8,000 Hz (base64-encoded).** This requires transcoding:

```
Twilio → decode base64 → G.711 mu-law @ 8kHz → upsample to PCM @ 16kHz → Gemini
Gemini → PCM @ 24kHz → downsample to G.711 mu-law @ 8kHz → base64 → Twilio
```

This transcoding is achievable in Node.js using Buffer manipulation or npm DSP libraries (e.g., `alawmulaw`, `@flo-bit/audio-utils`). It is more verbose than Python's `audioop` stdlib but not a blocker.

### 4. Recommended Model

Use `gemini-2.5-flash-native-audio-preview` (or `gemini-live-2.5-flash-native-audio` for GA).

> ⚠️ **Deprecation note:** `gemini-live-2.5-flash-preview-native-audio-09-2025` is deprecated and removed March 19, 2026. Do not use it.

Native audio models output audio directly (not TTS on top of a text response), which means lower latency and more natural-sounding speech — critical for a phone agent.

### 5. Session Constraints

| Constraint | Value |
|---|---|
| Max session duration (audio only) | 15 minutes |
| Max context window (native audio model) | 128k tokens |
| Typical first-token latency | ~600ms |

15 minutes is more than sufficient for a service intake call. Longer calls should be rare for Bremray's use case.

### 6. Function Calling / Tool Use ✅

The Live API supports function calling in Node.js. This is important for our architecture: the agent will call tools to look up customers in HCP, create jobs/leads, and trigger SMS/email — all mid-conversation. This works in the Node.js SDK.

### 7. Known Issues

**Transcription in AUDIO-only mode:** A known issue exists in `@google/genai@^1.34.0+` where `inputAudioTranscription` and `outputAudioTranscription` objects may not be received in `onmessage` when `Modality.TEXT` is excluded from `responseModalities`. This does not affect audio streaming itself — only the optional text transcription side-channel. For our use case (audio agent + function calls for structured data), this is not a blocker.

### 8. Python vs. Node.js Trade-off

| Dimension | Node.js (`@google/genai`) | Python (`google-genai`) |
|---|---|---|
| SDK maturity | Good — v1.44, actively updated | Slightly more mature |
| Live API support | ✅ Full | ✅ Full |
| Audio streaming | ✅ Supported | ✅ Supported |
| Audio transcoding libs | Fewer npm options vs Python stdlib | `audioop` (stdlib) simplifies this |
| WebSocket server (Fastify/ws) | Excellent | Good (FastAPI/websockets) |
| Emille's experience | ✅ Primary language | ❌ Not preferred |
| Official code examples | ✅ Yes | ✅ Yes (more) |

**Verdict:** The Python advantage is primarily `audioop` for audio transcoding, which is a small inconvenience in Node.js, not a fundamental blocker. Emille's Node.js proficiency is a larger factor in the other direction. **Stay in Node.js.**

---

## Recommendation

✅ **Proceed in Node.js with `@google/genai` v1.44+.**

Action items:
1. `npm i @google/genai` — use `gemini-2.5-flash-native-audio-preview` model
2. Use `ai.live.connect()` for the WebSocket session
3. Implement G.711 ↔ PCM transcoding using the `alawmulaw` npm package (lightweight, no native bindings)
4. Use function calling for mid-call HCP lookups and lead creation

---

## Sources

- [Gemini Live API Capabilities Guide | Google AI for Developers](https://ai.google.dev/gemini-api/docs/live-guide)
- [Get Started with Live API | Google AI for Developers](https://ai.google.dev/gemini-api/docs/multimodal-live)
- [Tool Use with Live API | Google AI for Developers](https://ai.google.dev/gemini-api/docs/live-api/tools)
- [@google/genai on npm](https://www.npmjs.com/package/@google/genai)
- [googleapis/js-genai on GitHub](https://github.com/googleapis/js-genai)
- [Known issue #1212 — transcription in Node.js SDK](https://github.com/googleapis/js-genai/issues/1212)
- [Gemini Live API overview | Vertex AI](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/live-api)
