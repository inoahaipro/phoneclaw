# PhoneClaw

An Android automation agent that turns a phone into something an AI agent can actually operate. Built as the execution layer behind a "Claude controls my phone" workflow: an AI sends a script, PhoneClaw runs it on-device and reports back.

**Live overview page:** https://phoneclawsite.vercel.app

## What it does

PhoneClaw isn't one app screen, it's four foreground services working together.

`LocalServerService` is the actual bridge -- a raw socket server on port `7779`. An AI agent, a script, Claude itself, whatever, opens a socket, sends a command, gets a result back. No cloud round-trip needed for the execution step.

`MyAccessibilityService` is the automation engine, built on Android's `AccessibilityService` API. Click by view ID, content-description, text, or raw coordinates. Swipe, type into fields, scroll to top/bottom, walk the accessibility node tree, dump the current UI state as JSON, find elements by class or bounding box. This is the piece that lets a script say "tap the button labeled X" and have it actually happen on any app, whether or not that app wants to cooperate.

`ScreenCaptureService` / `ScreenshotActivity` handle screen capture via `MediaProjection`, so the agent can see what it's driving while it works.

The Jarvis voice layer (`JarvisService`, `WakeWordService`, `VoiceCommandActivity`) is a wake-word voice assistant on top of all of it: double-click the side key, speak, it transcribes and hands the command to an LLM, the LLM's decision runs through the same automation engine above, TTS reads back the result.

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

## Getting started

PhoneClaw isn't on the Play Store. It's a sideloaded APK for a device you control.

1. **Install:** grab `phoneclaw-final2.apk` from this repo's [Releases](https://github.com/inoahaipro/phoneclaw/releases), enable "install from unknown sources" for your file manager/browser, install it.
2. **Grant permissions on first launch:**
   - Settings → Accessibility → PhoneClaw → turn the service **on** (this is what powers `MyAccessibilityService`'s tap/swipe/read automation; Android requires this to be enabled manually, it can't be granted programmatically).
   - Accept the screen-capture (MediaProjection) prompt when it appears, needed for `ScreenCaptureService`.
   - Grant any other runtime permission prompts (microphone for the Jarvis wake-word layer, etc.).
3. **Confirm the bridge is live:** open the app once. A persistent notification titled "PhoneClaw" with the text "Local server running on port 7779" means `LocalServerService` is up and listening.
4. **Talk to it:** open a raw TCP socket to the phone's IP on port `7779` and send a command. This is a script-facing integration point, not a polished public API. See *Known limitations* below before expecting a documented wire format.
5. **Voice mode:** double-click the side key to open the Jarvis mic dialog, speak a command, it executes through the same automation engine with a spoken response.

## Known limitations

Exact socket protocol isn't recovered. `LocalServerService.handleClient()`, the method that actually parses incoming commands, didn't decompile cleanly (JADX bailed with "Method not decompiled"), and the original source is gone -- see *Status* below. The port, service name, and overall bridge architecture are confirmed from the manifest and surrounding code, but the message format sent over that socket would need to be rebuilt or re-derived by hooking the running service.

No Play Store listing either. Sideload only, and Android will warn about installing from an unknown source -- normal for a personal tool, not something to worry about.

## Status

Built and verified working on a Samsung Galaxy S22+ (SM-S906U) in early 2026. The original Android/Kotlin source project didn't survive a later environment reset. This documentation was reconstructed by decompiling the last known-good build (`phoneclaw-final2.apk`). The compiled APK itself is attached to this repo's Releases.

**Stack:** Kotlin, Android `AccessibilityService`, `MediaProjection`, Kotlin Coroutines, raw sockets, Firebase (storage/db backend).

## Why it's different

Most "AI controls a phone" setups tether ADB to a computer. This one doesn't need that -- the whole execution loop runs on the device itself, as a background service, no tether and no ADB session at run time. That's the whole reason the wake-word Jarvis flow works at all: the phone runs itself, it isn't waiting on a computer.
