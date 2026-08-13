# Babel Buddy

<p align="center">
  <img src="https://github.com/user-attachments/assets/018a847e-2ccc-4f78-9e92-29dcc05f799e" alt="babel fish">
</p>

Live speech-to-speech translation. Speak in one language, hear it back in another.

A full audio pipeline — **transcribe → translate → synthesise** — behind a Flask API, containerised
with nginx and gunicorn and deployed with TLS. Built in 2024 on Azure Cognitive Services Speech,
before conversational speech models made this a single API call.

## Demo

Initial proof of concept:

https://github.com/user-attachments/assets/a96efc79-0f02-4c79-bcf2-c1e3a2bfcffd

## How it works

```
audio in ──► SpeechTranscriber ──► translated text ──► SpeechSynthesizer ──► audio out
             (recognise + translate)                    (TTS in target voice)
```

| Component | Role |
|---|---|
| `library/speech/SpeechTranscriber` | Recognises source audio and returns text in the target language |
| `library/speech/SpeechSynthesizer` | Renders translated text to speech, with PCM/WAV normalisation and ffmpeg transcoding for inputs that don't conform |
| `backend/main.py` | Flask API and UI surface |
| `nginx/` | Reverse proxy, TLS termination via Let's Encrypt |
| `library/base/log_handler.py` | Structured logging |

Typed exception boundaries (`SpeechTranscriberException`, `SpeechSynthesizerException`,
`UnsupportedFileType`) so failures in the audio path surface as something the API can respond to
rather than a stack trace.

## Stack

Python · Flask · Azure Cognitive Services Speech · ffmpeg · gunicorn (4 workers) · nginx ·
Docker Compose · Let's Encrypt

## Quickstart

```bash
cp .env.example .env        # add SPEECH_API_KEY and the Azure region
docker compose up --build   # nginx on :80/:443, Flask on :5000
```

Running the backend alone:

```bash
pip install -r backend/requirements.txt
gunicorn -w 4 -b 0.0.0.0:5000 main:app
```

## Goals

- start with solving the problem in the simplest way ✔️
- adjust the solution to be cloud service agnostic
- adjust the solution to be scalable microservices

## Design

Two architectures considered for scaling past the synchronous baseline.

### Option 1: Worker Queue

- horizontal scalability at the API and worker level
- asynchronous processing of tasks
- webhook the completed workload back to the API, which returns it to the frontend

<p align="center">
  <img src="https://github.com/user-attachments/assets/07deffdb-5e8c-4699-a609-112cf122da3e">
</p>

### Option 2: Shared Responsibility

<p align="center">
  <img src="https://github.com/user-attachments/assets/63e7cf13-569c-400c-86a9-e1c18512e4d5">
</p>

## What I'd do differently now

Kept here deliberately as a snapshot of the pre-LLM approach. Rebuilding it today:

- **Stream instead of round-tripping files.** The pipeline writes and reads audio files at each
  stage. Continuous recognition, or a realtime speech-to-speech model, removes most of the latency
  and all of the format-conversion code.
- **Put an adapter interface in front of the provider.** "Cloud service agnostic" was a stated goal
  that the code never reached — the Azure SDK is called directly. One interface with a recorded
  fixture per provider would have delivered it.
- **Translate with an LLM, not literal MT.** Machine translation handles words; idiom, register and
  domain vocabulary need context. That's the single biggest quality gain available.
- **Measure it.** There are no evaluation datasets here — quality was judged by listening. A held-out
  set with scored outputs would have made "is this better?" answerable.
- **Actually build the worker queue.** It's designed above and never implemented; the deployed
  version is synchronous.

## Licence

MIT
