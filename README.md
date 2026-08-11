# PhoneClaw

An Android automation agent that turns a phone into something an AI agent can actually operate — not just query. Built as the execution layer behind a "Claude controls my phone" workflow: an AI sends a script, PhoneClaw runs it on-device and reports back.

**Live overview page:** https://phoneclawsite.vercel.app

## What it does

PhoneClaw is a foreground Android service stack, not a single app screen. Four pieces work together:

- **`LocalServerService`** — a raw socket server on port `7779` running as a foreground service. This is the actual bridge: an external process (an AI agent, a script, Claude itself) opens a socket, sends a command, and gets a result back. No cloud round-trip required for the execution step itself.
- **`MyAccessibilityService`** — the automation engine, built on Android's `AccessibilityService` API. Implements a real general-purpose UI toolkit: click by view ID / content-description / text / coordinates, swipe, type into fields, scroll to top/bottom, walk the accessibility node tree, dump the current UI state as JSON, find elements by class or bounding-box area. This is what lets a script say "tap the button labeled X" and have it actually happen, on any app, without that app cooperating.
- **`ScreenCaptureService` / `ScreenshotActivity`** — screen capture via `MediaProjection`, so an agent can *see* the screen it's driving, not just blindly send gestures.
- **Jarvis voice layer** (`JarvisService`, `WakeWordService`, `VoiceCommandActivity`) — a wake-word-triggered voice assistant: speak a command, it gets transcribed and handed to an LLM, the LLM's decision gets executed through the same automation engine above, with TTS feedback. Side-key double-click → mic → speak → act → respond.

## Architecture

```
   AI agent / script
          |
          v  (socket, port 7779)
  LocalServerService  <-------- command/response bridge
          |
          v
  MyAccessibilityService  ----> drives any app's UI (click/type/swipe/read)
          |
  ScreenCaptureService  ------> gives the agent eyes on the current screen
          |
  Jarvis voice layer  --------> wake word -> STT -> LLM decision -> same
                                automation engine -> TTS response
```

## Status

Built and verified working on a Samsung Galaxy S22+ (SM-S906U) in early 2026. The original Android/Kotlin source project didn't survive a later environment reset — this documentation was reconstructed by decompiling the last known-good build (`phoneclaw-final2.apk`) rather than from the lost source tree. The compiled APK itself is attached to this repo's Releases.

**Stack:** Kotlin, Android `AccessibilityService`, `MediaProjection`, Kotlin Coroutines, raw sockets, Firebase (storage/db backend).

## Why it's interesting

Most "AI controls a phone" demos rely on ADB tethered to a computer. PhoneClaw runs the whole execution loop *on the device itself* as a background service — no tether, no ADB session required at run time — which is what makes the wake-word Jarvis flow possible: the phone is the agent, not just a screen someone else is remoting into.
