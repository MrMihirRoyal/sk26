# 26088

A multilingual voice assistant - a Flask web app plus an ESP32 hardware
device - that listens to a spoken question, answers it (via live web
search or a local knowledge document), and speaks the answer back in the
same language.

## Features

- **Voice in, voice out.** Hold the mic button (or the `S` key) to ask a
  question by speaking; the reply plays back automatically the moment it
  arrives, in the same language, as real server-generated speech (not the
  browser's own text-to-speech, which usually has no installed voice for
  most of the languages below and would silently substitute English) -
  no extra tap needed.
- **10 supported languages:** English, Hindi, Bengali, Marathi, Telugu,
  Tamil, Gujarati, Kannada, Malayalam, and Punjabi. Urdu is intentionally
  not included. The spoken language is auto-detected; you never have to
  pick it manually.
- **Answer mode toggle:** answers come from a live Groq web search
  ("Web search") or from a local reference document ("Knowledge doc") -
  switchable instantly from the chat header, no restart needed.
- **Dark mode**, toggleable, remembered per browser.
- **Clean, mobile-first UI** that also runs on a phone browser.
- **ESP32 hardware device**: a push-to-talk microphone/speaker unit with
  an OLED status display, talking to this same server - see
  `firmware/README.md`.

## Setup

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # then edit .env and add your GROQ_API_KEY
python app.py
```

Open **http://localhost:5000** - no login, it's a single shared chat.

## Speech recognition, translation, and text-to-speech

Every one of these three has a priority order, falling back to the next
tier automatically on any failure (missing config, network error,
timeout):

**Speech recognition (input):** Groq Whisper always runs once first,
purely to auto-detect which of the 10 supported languages was spoken
(Bhashini's ASR can't auto-detect the language - it needs to be told up
front). Once the language is known, **Bhashini has priority** for
producing the actual transcript and English translation, if configured;
Groq's own transcript/translation (already obtained during detection) is
the automatic fallback.

**Translation:** Bhashini (priority) → Groq chat model (fallback).

**Text-to-speech (output), in priority order:**
1. **Bhashini TTS** - highest priority. Needs `BHASHINI_USER_ID` /
   `BHASHINI_ULCA_API_KEY` (see `.env.example` for where to get these).
2. **espeak-ng** - fully offline, the fallback if Bhashini isn't
   configured or fails. Install with `sudo apt-get install espeak-ng`
   (Debian/Ubuntu) or `brew install espeak-ng` (macOS). Speaks the
   **entire** response - including mixed scripts and embedded numbers -
   in a single correct voice for the target language. (This used to
   sometimes read the whole response in English instead when the text
   had numbers or mixed content mixed in; the fix was forcing UTF-8 input
   decoding explicitly with espeak-ng's `-b 1` flag, since some
   platforms - Windows especially - otherwise guess the wrong input
   encoding for non-Latin scripts and garble every non-ASCII character.)
3. **Groq TTS** - last resort, **English only**. Groq doesn't currently
   offer voices for the other 9 supported languages, so this tier is
   skipped automatically for anything but English.

Check `GET /esp/health` to see which backends are currently usable.

### How the browser actually plays the reply

The browser chat does **not** rely on the browser/OS's own built-in
text-to-speech (the Web Speech API's `speechSynthesis`) as the primary
playback method, because most browsers and operating systems simply don't
have an installed voice for languages like Telugu, Tamil, Bengali, etc. -
`speechSynthesis` doesn't error out in that case, it silently substitutes
whatever default voice is installed (almost always English), so the
person hears the *right text* read in the *wrong language*, which looks
identical to "it just isn't working."

Instead, `/api/text_message` and `/api/voice_message` both call the same
`synthesize_speech()` priority chain described above (Bhashini → espeak-ng
→ Groq TTS) to generate a real WAV file in the actual input language,
serve it from `/audio/<file>.wav`, and the page plays that file
immediately via a plain `<audio>` element - the same guaranteed-correct
audio the ESP32 device gets, just delivered to a browser tab instead of a
speaker. The browser's own `speechSynthesis` is kept only as a last-resort
fallback, used solely if the server couldn't produce any audio at all
(e.g. no TTS backend configured or installed).
Check `GET /esp/health` to see which backends are currently usable.

## Project structure

```
app.py                Flask routes: chat API, ESP32 device endpoints
modl.py                Chat history DB, knowledge base, Groq + Bhashini calls
data/knowledge.txt     Reference text used in "document" answer mode
templates/
  chat.html             The chat UI (voice + text, dark mode, mobile-first)
static/logo.png
firmware/
  esp32_voice_assistant.ino   ESP32 Devkit V1 firmware
  README.md                    Wiring, libraries, flashing instructions
requirements.txt
.env.example
```

## ESP32 hardware device

See `firmware/README.md` for wiring, required Arduino libraries, and
flashing steps. The device's conversation is kept separate from the
browser's, under its own fixed entry (`esp32-device`) on the server, so
questions asked on the physical device don't clutter the browser chat and
vice versa.

## Notes

- Chat history is stored in a local SQLite file (`app.db`), created
  automatically on first run - there are no accounts, so it's one shared
  conversation for the browser plus the separate device conversation.
- Deploying to a serverless platform (e.g. Vercel)? Its filesystem is
  read-only outside `/tmp`, and `/tmp` is wiped on cold starts, so chat
  history won't be durable there without pointing `DB_PATH` at a real
  persistent database.
