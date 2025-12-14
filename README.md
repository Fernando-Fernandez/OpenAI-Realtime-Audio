# Realtime Transcription Playground

This repo contains two self‑contained browser demos for OpenAI’s WebRTC/Realtime stack. Serve the folder locally (e.g., `npx serve .`) so `getUserMedia` works, open each page in Chrome/Edge/Safari, and paste an API key or ephemeral token as indicated.

## `transcriber.html` – Streaming Audio Transcription (Realtime API)

Implements the workflow from the “Streaming the transcription of an ongoing audio recording” guide.

- **Ephemeral handshake**: Uses your standard API key once (via `POST /v1/realtime/transcription_sessions`) to mint a short‑lived client secret + websocket URL. The browser never stores the real key.
- **Realtime websocket**: Connects to `wss://api.openai.com/v1/realtime?intent=transcription` using the recommended subprotocols (`realtime`, `openai-insecure-api-key.<secret>`, `openai-beta.realtime-v1`) and sends a `transcription_session.update` payload with server VAD, near-field noise reduction, PCM16 audio, and logprob metadata.
- **Audio pipeline**: Captures the microphone with an `AudioWorkletNode`, converts float samples to PCM16, base64-encodes each chunk, and pushes it through `input_audio_buffer.append`.
- **VAD + results**: Shows all realtime events in the right panel (speech start/stop, buffer commits, errors) and streams transcript deltas on the left. When a `conversation.item.input_audio_transcription.completed` event matches the previously streamed text, it only prints a separator (`---`) to avoid duplicates.

Use this page when you want to replicate the official Realtime transcription flow end-to-end or debug low-level events.

## `listener.html` – Legacy WebRTC Listener

Earlier prototype that talks to the Realtime API over plain WebRTC SDP instead of the dedicated transcription intent.

- Prompts for your API key, creates an `RTCPeerConnection`, and posts its SDP offer to `https://api.openai.com/v1/realtime?...&mode=webrtc`.
- Streams the microphone via RTP media tracks and receives text/audio responses over a data channel (`oai-events`).
- Includes optional turn detection, rate-limit handling, and a transcript log but still relies on the general-purpose Realtime model (e.g., `gpt-4o-realtime-preview`), not the transcription intent.

Use `listener.html` if you’re experimenting with two-way Realtime conversations or need the general-purpose model to synthesize output as well as transcribe input.

---

Both pages expect to be served over `https://` or `http://localhost` due to browser media security rules. Neither stores keys; refresh the page between tests if you need to reset state. Adjust the models, VAD settings, or UI as needed for your project.

### Glossary

- **VAD (Voice Activity Detection)** – Logic that detects when speech starts or stops. The server VAD settings (`threshold`, `prefix_padding_ms`, `silence_duration_ms`) tell OpenAI when to commit audio chunks so we only request transcripts while someone is speaking.
- **PCM16** – Pulse-code modulation with 16-bit signed samples at 16 kHz. This raw `int16` format is what the transcription intent accepts; the AudioWorklet converts browser floats to PCM16 before base64 encoding each chunk.
- **SDP (Session Description Protocol)** – Text blob describing media capabilities in WebRTC offers/answers. `listener.html` creates an SDP offer for its audio track + data channel, sends it to OpenAI, and receives an SDP answer so the peer connection can exchange RTP packets.***
