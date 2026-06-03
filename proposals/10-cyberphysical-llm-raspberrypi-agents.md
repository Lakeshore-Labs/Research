# Cyberphysical Agents: LLM-Driven Raspberry Pi Devices

**One-liner:** Design a reusable, safe pattern for letting an LLM drive physical hardware (a validated action space the model cannot escape) and use it to answer an honest question: which Pi device experiences genuinely need an LLM rather than rule-based firmware? Anchor build: a smart mirror you can actually talk to.
**Area:** ML / SW / Research   •   **Difficulty:** Intermediate (Advanced if pushing on-device VLM inference or autonomous multi-step actuation)

## Motivation
Wiring an LLM into a Raspberry Pi is largely a documented engineering exercise. The research is not the wiring; it's two things the wiring doesn't settle.

First, a safety problem worth solving once and reusing: a probabilistic model that can emit any token must not be able to do anything to physical hardware. The contribution is a **bounded, validated action space**: a small typed tool layer where every actuator effect is checked before it runs and there is no path from model output to GPIO that bypasses validation. That pattern is transferable to any LLM-driven device, not specific to this build.

Second, an honest empirical question. Classic Pi devices (MagicMirror², a thermostat, a doorbell) run fixed firmware: pre-configured widgets and hard-coded triggers. Putting an LLM in the loop *can* change the interaction model. A mirror you ask "what's on my calendar and will it rain?" answers differently from one with a fixed weather widget. But it is genuinely unclear which experiences need the LLM versus where rule-based firmware already does the job as well or better. The deliverable is a clear-eyed finding on that boundary, backed by demo evidence, not a claim that the LLM helps everywhere.

## Research Question(s)
1. **(Primary, safe-control pattern)** What does a bounded, validated action space for LLM-driven hardware look like, such that the model can express intent freely but cannot produce an unsafe or out-of-range physical effect? What are its failure modes, and how cleanly does the pattern generalize beyond one device?
2. **(Primary, the honest question)** Which device experiences actually need the LLM rather than rule-based firmware? Where does the LLM earn its place (open-ended language, fused context, vision grounding), and where is it strictly worse than firmware (latency, reliability, cost) for the same job? State the boundary as a finding.
3. (Optional) How far behind is on-device small-model inference? Where it doesn't compromise the build, characterize the latency/privacy/offline tradeoff against the cloud driver.

## Scope
- In scope:
  - One real, usable physical device: a Pi 5 with camera, mic, speaker, and a plain HDMI display, that someone can walk up to and talk to.
  - The full loop, wake word → STT → optional camera frame → LLM with function-calling → TTS + display update + optional GPIO action.
  - 1–2 LLM-enabled experiences end to end (e.g. multi-turn voice Q&A grounded in fetched calendar/weather; a vision-grounded turn like "what am I holding").
  - A bounded tool API mapping LLM function calls to `gpiozero` / `picamera2` / display / STT / TTS.
- Out of scope:
  - Multi-device fleets, custom PCB design, training or fine-tuning a model.
  - Production privacy hardening. Document the concerns; this is a prototype, not a shipped product.
  - Full home-automation integration (Home Assistant bridge is a stretch goal).
  - Reliable autonomous multi-step actuation (stretch).
  - On-device/local inference as a required deliverable (optional stretch only).

## Suggested Approach
A starting path, not prescriptive. Get a minimal end-to-end loop working first (steps 1–4 with one tool and a cloud model), then build out the mirror.
1. Stand up the Pi and I/O. Pi 5 (8GB), active cooler, Camera Module 3, USB or I2S mic, speaker, HDMI display. Verify `picamera2` capture and `gpiozero` (lgpio backend on Pi 5) blinking an LED and driving a relay.
2. Build the voice front-end. Wake word (openWakeWord or Picovoice Porcupine) → STT (whisper.cpp / faster-whisper, or cloud STT) → TTS (Piper, on-device). STT is often the slowest stage on-device; if it drags, lean on cloud STT.
3. Define the bounded tool layer. This is the core research artifact, so treat it as such. Expose a small set of typed tools via function-calling, e.g. `show_panel(name)`, `set_led(zone, state)`, `capture_and_describe()`, `get_calendar()`, `get_weather()`. The LLM never touches raw GPIO. Every tool validates its arguments (enum/range checks) before any hardware effect, and unknown or out-of-range calls are refused. Design it so the validation boundary, not the model, is the single point that authorizes any physical effect, and so the same boundary would hold for a different set of tools or a different device.
4. Minimal loop with the cloud model. The cloud frontier model (function-calling + vision) is the primary driver; the Pi is the sensing/acting edge. Get one spoken request → one tool call → spoken+display response working before adding more. For vision turns, send a single frame to the multimodal API rather than streaming.
5. Anchor build, the mirror. Either extend MagicMirror² with an LLM module that summarizes the other modules and answers questions, or build a minimal custom display on the HDMI monitor. The LLM decides what to surface and narrates it. Aim for something a person would actually use, not just a demo that fires once.
6. (Optional stretch) Measure local. If time allows, swap the driver for a small local model via Ollama / llama.cpp (e.g. Llama-3.2-3B, Qwen2.5), or an accelerated small model on a Hailo HAT, and compare against the cloud baseline. This is a side investigation, not a gate on the build.

## Deliverables
- **The safe-control pattern (headline contribution):** the bounded tool API as reusable code plus a design note that states the pattern generally: how the action space is constructed, why no model output can reach hardware without validation, which failure modes are handled, and how to apply it to a different device. This is the part meant to outlive the build.
- **The honest finding (headline contribution):** a written verdict on which experiences genuinely need the LLM versus where rule-based firmware does the job as well or better, backed by side-by-side demo evidence. Including negative results (cases where the LLM added nothing or made things worse) is a success, not a failure.
- A working, usable physical Pi device running the full perceive→reason→act loop: something you can walk up to, talk to, and that does something a rule-based build can't.
- 1–2 LLM-enabled experiences demonstrated with before/after framing against a rule-based equivalent.
- A short evaluation write-up: how reliably the device handles real requests, where the time goes (latency breakdown by stage), and how it behaves on requests it should refuse. If the local stretch was attempted, a cloud-vs-local comparison.
- README, wiring diagram, and bill of materials so the build is reproducible.

## Demo
Live on the device. A person says the wake word and has a multi-turn conversation with the mirror. First a context turn: "What's on my calendar today and is it going to rain?" The mirror fetches calendar and weather, answers in speech, and updates the display. Then a vision turn: "What am I holding up?" The mirror captures a single frame and the model describes it. Then a bounded actuation: "turn on the hallway light," which the LLM routes to the `set_led`/relay tool, and the hardware responds. Each turn is conversational and context-dependent rather than a fixed trigger, which is the gap from stock MagicMirror².

These are goals, not pass/fail gates. The aim is a device that genuinely works, not a number to hit.
- Works physically and holds up: the device runs the full loop live, and keeps working across a normal session of use rather than only on a lucky take.
- Handles real requests reliably: across a spread of spoken requests covering the supported tools, the device picks the right tool, passes valid args, and gives a sensible response most of the time. Note where it falls down and why.
- Feels responsive: wake-to-response is quick enough to be pleasant to use. Capture a stage-by-stage latency breakdown (wake, STT, network/LLM, TTS, actuation) so it's clear where the time goes.
- Refuses unsafe actions: every actuator call is validated before execution; unknown or out-of-range requests are refused, not run. There is no path from the LLM to GPIO that bypasses validation. Show this with a few adversarial requests.
- The honest finding lands: for each shown experience, a written verdict with demo evidence on whether the LLM genuinely earns its place or whether rule-based firmware does it as well or better. A credible "firmware wins here" conclusion counts as success.

## Tech & Prerequisites
Hardware (bill of materials, rough USD; prices vary by region/retailer, treat as estimates):
- Raspberry Pi 5, 8GB (~$80) + official active cooler (~$5) + 27W PSU (~$12) + microSD or NVMe (~$15–40)
- Raspberry Pi Camera Module 3 (~$25)
- USB or I2S microphone (~$10–30); small speaker or amp (~$10–20)
- Display: a plain HDMI monitor (reuse what you have). No mirror enclosure needed; the "smart mirror" is the software concept, on an ordinary display.
- Optional, only for the local-inference stretch: Raspberry Pi AI HAT+ (Hailo-8L, 13 TOPS, ~$70) or AI HAT+ 2 (Hailo-10H, 40 TOPS INT4, ~$130). Vendor TOPS figures; useful VLM inference quality on these is unproven (see Open Questions).
- Rough total: ~$180–250 baseline; add ~$70–130 only if attempting the local stretch with an AI HAT.

Software / libraries: Python; `picamera2`, `gpiozero` (+ lgpio on Pi 5); openWakeWord or Porcupine; whisper.cpp / faster-whisper; Piper TTS; Ollama or llama.cpp for local models; an LLM provider SDK with function-calling and vision for the cloud driver; optionally MagicMirror² (Node.js) and the Hailo runtime / `hailo-apps` (the former `hailo-rpi5-examples` is now deprecated in favor of `hailo-apps`).

Background assumed: comfortable with Python and the Linux command line; basic electronics safety (don't drive mains from GPIO directly, use a relay or HAT); willing to learn function-calling and basic prompt design. No ML training experience needed.

## Phases
- Explore: stand up Pi I/O; get camera, mic, GPIO, and a hello-world cloud LLM tool call each working in isolation. Survey MagicMirror² and one existing LLM-mirror project.
- Build: assemble the voice loop, the bounded tool layer, and the display; integrate the cloud LLM driver; get the mirror experience working end to end and usable.
- Evaluate: exercise the device on a spread of real requests; capture the latency breakdown; check refusal behavior. Cloud-vs-local comparison only if the local stretch was attempted.
- Polish: failure handling (network drop, STT miss, tool error), README and wiring diagram, demo script.

## Stretch Goals
- Fully local/offline operation (on-device VLM via Hailo) and measure the quality gap against cloud.
- Presence-aware adaptation: detect that someone is present (not identity) and adapt content or persona.
- Multi-step autonomy: chain tool calls for a compound request, with a confirmation step before any actuation.
- Home Assistant bridge for real smart-home control.
- Latency work: streaming STT, response streamed to TTS, frame-on-demand vs. periodic vision.

## Starter References
- MagicMirror², open-source modular smart mirror platform: https://github.com/MagicMirrorOrg/MagicMirror
- `picamera2`, official libcamera-based Python camera library: https://github.com/raspberrypi/picamera2  •  `gpiozero` docs: https://gpiozero.readthedocs.io/
- Arm Learning Path: "Build a Privacy-First LLM Smart Home on Raspberry Pi 5" (LLM → JSON → gpiozero, local Ollama): https://learn.arm.com/learning-paths/embedded-and-microcontrollers/raspberry-pi-smart-home/
- Raspberry Pi AI HAT+ 2 / Hailo (on-device generative AI accelerator) and Hailo Pi 5 examples: official AI HATs docs https://www.raspberrypi.com/documentation/accessories/ai-hat-plus.html  •  AI HAT+ 2 announcement https://www.raspberrypi.com/news/introducing-the-raspberry-pi-ai-hat-plus-2-generative-ai-on-raspberry-pi-5/  •  https://github.com/hailo-ai/hailo-apps
- Ollama on Raspberry Pi 5 (local small-model inference, perf data): https://raspberry.tips/en/raspberrypi-tutorials/ollama-raspberry-pi-5  •  benchmark context: https://www.stratosphereips.org/blog/2025/6/5/how-well-do-llms-perform-on-a-raspberry-pi-5
- Offline voice stack (Porcupine/openWakeWord + Whisper + Piper) reference: https://www.promptquorum.com/power-local-llm/build-local-voice-assistant-2026

## Open Questions / Assumptions
- Decided, cloud is the driver: cost is not a concern, so a cloud frontier model drives the agent (best quality and latency). Local/on-device inference is an optional stretch, not a required arm.
- Decided, plain HDMI display: no two-way-mirror enclosure. The "smart mirror" is the software concept, running on an ordinary monitor.
- Privacy/safety posture: always-on camera/mic is a real concern. The posture is on-demand capture (frames and audio only after the wake word, not continuous streaming) with the data path documented (what leaves the device, when, to where). Confirm this fits the deployment setting.
- On-device VLM maturity (uncertain): a useful vision LLM running locally on Pi 5 + Hailo is plausible but there's no confirmed turnkey, well-supported path at the needed quality. The local stretch is exploratory; cloud vision (single frame to a multimodal API) is the reliable baseline either way.
- Latency figures in public reports (Whisper small ~2–4 s; 1–3B models ~4–8 tok/s on Pi 5 CPU) are rough context, not targets. Measure on the actual hardware.
