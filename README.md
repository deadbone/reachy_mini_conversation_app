---
title: Reachy Mini Conversation App
emoji: 🎤
colorFrom: red
colorTo: blue
sdk: static
pinned: false
short_description: Talk with Reachy Mini!
suggested_storage: large
tags:
 - reachy_mini
 - reachy_mini_python_app
---

# Reachy Mini conversation app

Conversational app for the Reachy Mini robot combining realtime voice, vision, personality-aware tools, and choreographed motion.

![Reachy Mini Dance](docs/assets/reachy_mini_dance.gif)

## Table of contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the app](#running-the-app)
- [LLM tools](#llm-tools-exposed-to-the-assistant)
- [Creating and adding tools](#creating-and-adding-tools)
- [Advanced features](#advanced-features)
- [Contributing](#contributing)
- [License](#license)

## Overview

- Low-latency audio conversation through the Hugging Face realtime backend, using the built-in server or a local endpoint.
- Vision is handled by the realtime backend when the `camera` tool is used.
- Layered motion system queues primary moves (dances, emotions, goto poses, breathing) while blending speech-reactive wobble.
- Async tools integrate motion, camera capture, and MCP Tool Spaces. The optional web UI (`--ui`) manages conversations, personalities, tools, and settings, and its Talk view can show the live transcript. The backend's per-turn latencies appear there when **Settings → Developer options → Show response latencies** is on; the choice is saved across launches.

## Architecture

The app connects the user, AI services, and robot hardware:

<p align="center">
  <img src="docs/assets/conversation_app_arch.svg" alt="Architecture Diagram" width="600"/>
</p>

## Installation

> [!IMPORTANT]
> Install [Reachy Mini's SDK](https://github.com/pollen-robotics/reachy_mini/) before using this app.<br>
> Windows support is currently experimental and has not been extensively tested. Use with caution.

<details open>
<summary>Using uv (recommended)</summary>

Set up with [uv](https://docs.astral.sh/uv/):

```bash
# macOS (Homebrew)
uv venv --python /opt/homebrew/bin/python3.12 .venv

# Linux / Windows (Python in PATH)
uv venv --python python3.12 .venv

source .venv/bin/activate
uv sync
```

Include dev dependencies:
```bash
uv sync --group dev
```

</details>

> [!NOTE]
> Run `uv sync --frozen` to install the exact dependency set from `uv.lock` without re-resolving versions.

<details>
<summary>Using pip</summary>

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

Install dev dependencies:
```bash
pip install -e .[dev]                   # Development tools
```

</details>

## Configuration

The default setup uses the Hugging Face backend and does not require an API key.

Copy `.env.example` to `.env` when you want to point Hugging Face at your own local endpoint.

| Variable | Description |
|----------|-------------|
| `REALTIME_TRANSCRIPTION_LANGUAGE` | Optional input transcription language for the realtime backend. Defaults to `auto` for automatic language detection; set to a backend-supported code such as `en` for English or `zh` for Chinese to fix the language. |
| `LOCAL_BARGE_IN_ENABLED` | Local audio interruption. Defaults to `true`; requires `LOCAL_VAD_ENABLED=true`. |
| `LOCAL_VAD_ENABLED` | Local Silero speech detection for faster physical listening reactions. Defaults to `true`; set to `false` to use server VAD alone for movement. |
| `HF_REALTIME_CONNECTION_MODE` | Hugging Face connection selector: `deployed` uses the built-in Hugging Face server; `local` uses `HF_REALTIME_WS_URL`. Defaults to `deployed`. |
| `HF_REALTIME_WS_URL` | Direct websocket endpoint for your own Hugging Face backend. Accepts either a base URL like `ws://127.0.0.1:8765/v1` or the full websocket URL `ws://127.0.0.1:8765/v1/realtime`. Used when `HF_REALTIME_CONNECTION_MODE=local`. |
| `HF_TOKEN` | Optional token for Hugging Face access. Local endpoints receive only this explicitly configured token. |
| `REACHY_MINI_APP_TIMEOUT_MINUTES` | Minutes of inactivity before Reachy goes to sleep and the app stops. Defaults to `1440` (one day); set to `0` to disable. |
| `REACHY_MINI_INSTANCE_PATH` | Directory for this instance's data: long-term memory, user personalities and settings. Defaults to `conversation_app` beside the daemon's own config (`~/.config/reachy_mini` on Linux). Data written by an older version inside the installed package is copied over once. |
| `REACHY_MINI_MEMORY_ENABLED` | Long-term memory. When `false`, remembered facts stay on disk but are not injected into the prompt, and the `remember` / `forget` tools decline. Defaults to `true`. |
| `REACHY_MINI_CAMERA_ENABLED` | Camera tool. `--no-camera` forces it off regardless. Defaults to `true`. |

These variables are the starting value. What you change through the settings UI
(or from the mobile app) is written to `settings.json` in the instance directory
and wins over them, so a toggle always does what it says.

`LOCAL_VAD_ENABLED` defaults to `true`. Change **Local speech detection** in the gear-wheel **Settings → Speech behavior** view. Saved choices take precedence over environment defaults and persist across launches; saving reconnects the conversation to apply the change. Local Silero VAD and server VAD both control physical listening: either detector freezes the antennas and suppresses idle movement. Movement resumes smoothly once both report silence. Local detection holds through 400 ms of silence and clears after 500 ms without microphone frames. Model history clears after each completed speech segment and after 500 ms of consecutive low-confidence idle audio (512 ms at the 32 ms inference cadence). These resets preserve pending microphone samples and context during speech and within-segment pauses. It uses the SDK's 16 kHz microphone audio; conversation turns continue to use server VAD. If local detection fails, the app logs a warning and uses server VAD alone.

Local barge-in defaults to enabled. Change **Local barge-in** in **Settings → Speech behavior**, or set `LOCAL_BARGE_IN_ENABLED=false` to disable it when no saved preference exists. It requires local speech detection; disabling that switch also disables local interruption while preserving the saved barge-in preference. While audio is buffered or being submitted for playback (plus a 100 ms trailing margin), a Silero speech candidate lowers upcoming playback to 25% amplitude. Playback only stops once server VAD confirms the user turn, so loud background speech ducks the robot without dropping its answer; late audio from the interrupted response is discarded. A rejected candidate restores normal volume after 64 ms of silence. Server interruptions remain enabled, and microphone audio continues streaming to the backend. Playback is fed to the SDK in paced 20 ms chunks; already-submitted audio and SDK/transport latency delay audible volume changes. This is experimental: speaker echo, motor noise, audible latency and CPU overhead still need robot testing. With `--debug`, open `/static/vad-monitor.html` on the app settings server for a read-only live monitor with completed user and robot transcripts. The transcript view keeps the last 100 turns received while the page is open and clears on refresh. The `/rpc` WebSocket emits `conversation.vad` diagnostics with a Unix timestamp, the latest Silero speech probability, local/server/combined listening states, barge-in candidate and playback gain. Probability updates are limited to 10 Hz; state changes are reported immediately. Diagnostics reuse existing inference and do not change the inactivity timer.

On the first startup with local VAD enabled, the app downloads the [Silero VAD v6.2.1 model and MIT license](https://github.com/snakers4/silero-vad/tree/v6.2.1) directly from the pinned upstream commit and verifies their SHA-256 checksums. They are stored in the Hugging Face assets cache (`HF_HOME/assets`, or `HF_ASSETS_CACHE` when set), like the cached emotion recordings. Later starts reuse the files without a network request. The model runs on CPU with ONNX Runtime. If the download or model load fails, the app logs a warning and continues with server VAD alone.

### Hugging Face Connection Modes

Use the built-in Hugging Face server through the app-managed Space proxy. This is the default for a new install; set it explicitly only when you want to switch back from a saved local endpoint:

```env
HF_REALTIME_CONNECTION_MODE=deployed
```

Deployed session allocation falls back to cached `hf auth login` credentials and reports the daemon-provided hardware ID when available. Cached credentials and the hardware ID are not sent to local endpoints.

Run your own realtime voice backend using [speech-to-speech](https://github.com/huggingface/speech-to-speech) on the same machine as the conversation app:

```env
HF_REALTIME_CONNECTION_MODE=local
HF_REALTIME_WS_URL=ws://127.0.0.1:8765/v1/realtime
```

Run your own Hugging Face backend on your laptop and connect to it from Reachy Mini Wireless over the same Wi-Fi network:

```env
HF_REALTIME_CONNECTION_MODE=local
HF_REALTIME_WS_URL=ws://<your-laptop-lan-ip>:8765/v1/realtime
```

For that LAN setup, make sure the backend listens on an address reachable from the robot, not only on `127.0.0.1`.

If the backend stays bound to loopback on your laptop, you can forward it into the robot over SSH instead:

```bash
ssh -N -R 8765:127.0.0.1:8765 <robot-user>@<robot-host>
```

Then set this on the robot:

```env
HF_REALTIME_CONNECTION_MODE=local
HF_REALTIME_WS_URL=ws://127.0.0.1:8765/v1/realtime
```

In the web UI's Settings view, the Connection section lets you choose either the built-in server or a local `host:port` target. The UI writes `HF_REALTIME_CONNECTION_MODE` for you, and the local path writes `HF_REALTIME_WS_URL` with a default of `localhost:8765`.

## Running the app

Activate your virtual environment, then launch:

```bash
reachy-mini-conversation-app
```

> [!TIP]
> Make sure the Reachy Mini daemon is running before launching the app. If you see a `TimeoutError`, it means the daemon isn't started. See [Reachy Mini's SDK](https://github.com/pollen-robotics/reachy_mini/) for setup instructions.

The app runs in console mode. Add `--ui` to serve the web interface at http://127.0.0.1:7860/.

### CLI options

| Option | Default | Description |
|--------|---------|-------------|
| `--no-camera` | `False` | Run without camera capture. |
| `--ui` | `False` | Serve the web UI at http://127.0.0.1:7860/, in addition to console mode. |
| `--robot-name` | `None` | Optional. Connect to a specific robot by name when running multiple daemons on the same subnet. See [Multiple robots on the same subnet](#advanced-features). |
| `--debug` | `False` | Enable verbose logging for troubleshooting. |

### Examples

```bash
# Audio-only conversation (no camera)
reachy-mini-conversation-app --no-camera

# Launch with the minimal web UI for personality/mic/settings control
reachy-mini-conversation-app --ui
```

## LLM tools exposed to the assistant

The default profile exposes these tools. Use Tools → Tool access to customize any profile.
Every bundled profile enables `head_tracking` by default; users can still disable it per personality.

| Tool | Action | Dependencies |
|------|--------|--------------|
| `dance` | Queue a dance from `reachy_mini_dances_library`. | Core install only. |
| `stop_dance` | Clear queued dances. | Core install only. |
| `play_emotion` | Play a recorded emotion clip via Hugging Face datasets. | Core install only. Uses the default open emotions dataset: [`pollen-robotics/reachy-mini-emotions-library`](https://huggingface.co/datasets/pollen-robotics/reachy-mini-emotions-library). |
| `stop_emotion` | Clear queued emotions. | Core install only. |
| `camera` | Capture the latest camera frame and analyze it with the selected realtime backend. | Core install only. Requires the camera (disable with `--no-camera`). |
| `idle_do_nothing` | Explicitly remain idle during an idle turn. Not intended for normal conversation turns. | Core install only. |
| `move_head` | Queue a head pose change (left/right/up/down/front). | Core install only. |
| `head_tracking` | Follow the user's face with the head, or stop following. | Core install only. Requires a daemon with the `vision` extra and a camera. |
| `go_to_sleep` | Run Reachy's sleep movement and stop the current app after an explicit user request. | Core install only. |
| `sweep_look` | Sweep Reachy's head left, right, and back to center. | Shared tool, enabled by default in the default profile. |
| `remember` | Save one short, stable fact about the user for future sessions. | Core install only. Stored in the app instance data directory. |
| `forget` | Remove a saved memory fact by matching a short query. | Core install only. |
| `volume_control` | Read or change Reachy's speaker or microphone volume. | Core install only. Uses the daemon REST API; setting the speaker volume plays a short confirmation sound. |
| `robot_status` | Read one status topic: `name`, `software` (version, update available), `wifi` (IP address, network), `account` (Hugging Face sign-in), `imu` (which way the head is tilted, motion, temperature), `apps` (installed apps). | Core install only. Uses the daemon REST API. The update check and the Wi-Fi network details are wireless-version only; the IP address is reported on any robot. |
| `pollen_robotics_reachy_mini_search_tool__search_web` | Search the web and return a short list of results. | Preinstalled MCP Space: `pollen-robotics/reachy-mini-search-tool`. |
| `pollen_robotics_reachy_mini_weather_tool__get_weather` | Report today's weather for a place: current conditions, high and low temperature, and rain chance. | Preinstalled MCP Space: `pollen-robotics/reachy-mini-weather-tool`. |
| `pollen_robotics_reachy_mini_time_tool__get_time` | Report the current time for a timezone or the user's local time, or the difference between two timezones. | Preinstalled MCP Space: `pollen-robotics/reachy-mini-time-tool`. |
| `pollen_robotics_reachy_mini_knowledge_tool__get_info` | Read the official overview, capabilities, versions, and prices for a Pollen Robotics product or the company. | Preinstalled MCP Space: `pollen-robotics/reachy-mini-knowledge-tool`. |
| `pollen_robotics_reachy_mini_knowledge_tool__search_docs` | Search the official Reachy Mini FAQ and troubleshooting documentation. | Preinstalled MCP Space: `pollen-robotics/reachy-mini-knowledge-tool`. |

> [!NOTE]
> `remember`/`forget` facts are stored in `memory.v1.json` inside the app's instance data directory (`~/.local/share/reachy_mini_conversation_app/` by default, or the instance path used by the desktop launcher). `forget` only removes facts matched by query. To reset all remembered facts, delete this file.

## Creating and adding tools

Tools can run locally as Python code or remotely in an MCP-compatible Hugging Face Space. Keep robot, camera, and local-data operations in local tools. A Space is a better fit for shareable, stateless services such as search and external API lookups.

### Local tools

Create one Python module per tool, with the file name matching the tool's unique `name`. See [`idle_do_nothing.py`](src/reachy_mini_conversation_app/tools/idle_do_nothing.py) for a minimal implementation.

Each tool subclasses `Tool` and defines `name`, a model-facing `description`, an object-shaped JSON Schema in `parameters_schema`, and an async `__call__` method. Use `ToolDependencies` for runtime services, and set `needs_response = False` for actions that should not trigger a spoken follow-up. Catch expected operational failures, log them with the module logger, and return `{"error": "..."}` so the conversation can continue.

Restart the app after adding the module. Use Tools → Tool access to enable it for a personality, or add its name to that profile's `default_tools` in `profile.md`. See [External profiles and tools](#external-profiles-and-tools) for external directories and autoload behavior.

### Hugging Face Space tools

To publish a remote tool, create a Gradio Space, expose its API as MCP with `mcp_server=True`, and give each function clear type hints and docstrings. Verify that `https://<space-subdomain>.hf.space/gradio_api/mcp/schema` lists the expected tools before installing the Space.

Use the maintained [weather](https://huggingface.co/spaces/pollen-robotics/reachy-mini-weather-tool), [time](https://huggingface.co/spaces/pollen-robotics/reachy-mini-time-tool), and [search](https://huggingface.co/spaces/pollen-robotics/reachy-mini-search-tool) Spaces as examples. See Gradio's [MCP server guide](https://www.gradio.app/guides/building-mcp-server-with-gradio) for additional publishing guidance and [Installing Hugging Face Space tools](#installing-hugging-face-space-tools) for this app's installation steps.

## Advanced features

Built-in motion content is published as open Hugging Face datasets:

- Emotions: [`pollen-robotics/reachy-mini-emotions-library`](https://huggingface.co/datasets/pollen-robotics/reachy-mini-emotions-library)
- Dances: [`pollen-robotics/reachy-mini-dances-library`](https://huggingface.co/datasets/pollen-robotics/reachy-mini-dances-library)

<details>
<summary>Custom profiles</summary>

Create custom profiles with dedicated instructions and per-profile tool access.

Select and save a startup profile in the UI. The choice is stored in `startup_settings.json`. Before one is saved, `REACHY_MINI_CUSTOM_PROFILE=<name>` can select `profiles/<name>/`; otherwise the app uses `default`.

Every profile directory contains one strict schema-version-1 `profile.md`. TOML metadata is enclosed by `+++`; the remaining Markdown body is the realtime assistant prompt:

```markdown
+++
schema_version = 1
voice = "Aiden"
greeting = "Greet me warmly in one sentence, in character, and vary the wording each time."
hidden = false
default_tools = [
  "dance",
  "camera",
  "sweep_look",
]
+++

## Identity

You are a concise, friendly robot guide.
```

`schema_version`, `default_tools`, and a non-empty Markdown body are required. `voice`, `greeting`, and `hidden` are optional. Set `hidden = true` to omit a profile from the UI. An empty `default_tools` list is valid and inherits nothing.

`default_tools` is the authored baseline. Tools → Tool access stores overrides in instance-local `profile_toolsets.json` without changing bundled profiles. Restoring defaults removes the override. Active-profile changes reconnect the conversation; other changes apply when selected.

Profile directories are data-only. Python tool implementations belong in `src/reachy_mini_conversation_app/tools/`, or in `REACHY_MINI_EXTERNAL_TOOLS_DIRECTORY` for external tools. Each enabled tool ID must resolve to a shared tool, an external tool, or a tool from an installed Hugging Face Space.

See [Creating and adding tools](#creating-and-adding-tools) for the local tool interface and a maintained example.

To manage personalities in the UI:

With `--ui`, Home lists the available profiles and the built-in default:

- Tap a card to apply that personality and start talking.
- Tap "Manage tools" on a saved personality to open its tool access directly.
- Tap "Custom" to create a personality with a name, instructions, and optional greeting. It inherits the default tools, which can be changed under "Manage tools". Managed instances store it at `user_personalities/<name>/profile.md`; standalone runs use `external_content/user_personalities/<name>/profile.md`.

Switching a personality reloads its prompt and effective tools through a quick backend reconnect. Editing `profile.md` directly requires re-selecting the profile or restarting the app.

</details>

<details>
<summary>Locked profile mode</summary>

To create a locked variant of the app that cannot switch profiles, edit `src/reachy_mini_conversation_app/config.py` and set the `LOCKED_PROFILE` constant to the desired profile name:
```python
LOCKED_PROFILE: str | None = "mars_rover"  # Lock to this profile
```
When set, the app ignores saved startup settings, `REACHY_MINI_CUSTOM_PROFILE`, and UI selection. The UI marks the profile as locked and disables editing.

</details>

<a id="external-profiles-and-tools"></a>

<details>
<summary>External profiles and tools</summary>

You can extend the app with profiles/tools stored outside the repository defaults.

- Core profiles are under `profiles/`.
- Core tools are under `src/reachy_mini_conversation_app/tools/`.

Recommended layout:

```text
external_content/
├── external_profiles/
│   └── my_profile/
│       └── profile.md
├── external_tools/
│   └── my_custom_tool.py
├── user_personalities/
│   └── my_custom_profile/
│       └── profile.md
├── installed_tool_spaces.json
└── profile_toolsets.json
```

Environment variables:

Set these values in your `.env` when you want env-driven external profile/tool selection:

```env
# Optional fallback/manual profile selector:
REACHY_MINI_CUSTOM_PROFILE=my_profile
REACHY_MINI_EXTERNAL_PROFILES_DIRECTORY=./external_content/external_profiles
REACHY_MINI_EXTERNAL_TOOLS_DIRECTORY=./external_content/external_tools
# Optional convenience mode:
# AUTOLOAD_EXTERNAL_TOOLS=1
```

Loading rules:

- Profiles: each directory requires a schema-version-1 `profile.md` with explicit `default_tools`; there is no cross-profile fallback.
- Default mode: enabled IDs must resolve to a shared, external, or installed Tool Space tool.
- Autoload: `AUTOLOAD_EXTERNAL_TOOLS=1` adds every valid `*.py` module from `REACHY_MINI_EXTERNAL_TOOLS_DIRECTORY`.
- Web UI: Tools → Tool access enables external modules per profile; it does not upload or edit Python.
- Separation: profile directories contain data only; external Python belongs in `REACHY_MINI_EXTERNAL_TOOLS_DIRECTORY`.
- Tool names: every loaded class needs a unique `Tool.name`; duplicates fail fast.

</details>

<a id="installing-hugging-face-space-tools"></a>

<details>
<summary>Installing Hugging Face Space tools</summary>

You can install MCP-compatible Hugging Face Spaces as remote tool sources for this app. Private Spaces work too, as long as `HF_TOKEN` is set (or you have run `hf auth login`) for an account that can access them. To publish a new Space, follow [Creating and adding tools](#hugging-face-space-tools).

Tools → Tool Spaces installs or refreshes a global source. Its tools then appear under Tools → Tool access for per-profile selection. Removing a Space removes its tools from every profile. Active-profile changes reconnect the conversation; other changes apply when selected.

The app accepts Hugging Face Spaces exposing the standard `/gradio_api/mcp/` endpoint, not arbitrary MCP URLs. Installation discovers the Space's tools and assigns namespaced local IDs, so do not guess or hard-code those IDs beforehand.

```bash
# install + enable in active profile
reachy-mini-conversation-app tool-spaces add <owner/space-name>

# enable in a specific profile
reachy-mini-conversation-app tool-spaces add <owner/space-name> --profile NAME

# install without enabling
reachy-mini-conversation-app tool-spaces add <owner/space-name> --install-only

# list installed spaces
reachy-mini-conversation-app tool-spaces list

# remove an installed space
reachy-mini-conversation-app tool-spaces remove owner/space-name
```

Bundled Pollen Spaces use static specs and are enabled by the default profile. Newly bundled Spaces are added to existing installations without restoring older tools the user removed. Custom Spaces are validated through the Hugging Face Hub; HF tokens are sent only to private Spaces. Tool metadata is cached in:

- `installed_tool_spaces.json` in the managed app instance directory
- `external_content/installed_tool_spaces.json` in terminal mode

Startup and profile switching read this cache without discovery or MCP probing. Network access occurs only during install, refresh, or remote tool calls. Per-profile access is stored in `profile_toolsets.json` beside the manifest, or under `external_content/` in terminal mode.

Recommended tags for discoverability on Hugging Face:

- `reachy-mini-tool`
- `mcp`

Tags are advisory; installation still requires successful MCP validation.

> [!NOTE]
> Preinstalled Pollen Spaces can be removed like any other (`tool-spaces remove pollen-robotics/reachy-mini-weather-tool`). To restore access, reinstall the Space and restore or update the relevant profile under "Tool access".

</details>

<details>
<summary>Multiple robots on the same subnet</summary>

If you run multiple Reachy Mini daemons on the same network, use:

```bash
reachy-mini-conversation-app --robot-name <name>
```

`<name>` must match the daemon's `--robot-name` value so the app connects to the correct robot.

</details>

## Fork personnel deadbone : audit et maintenance

Ajout propre au fork, le 16 octobre 2026. L'application, ses identifiants, ses
licences et ses attributions Pollen Robotics sont conservés. Cette section est
un audit et un guide de préparation ; les fonctions des phases 2 à 7 ne sont pas
encore implémentées. Aucune issue ni PR upstream n'est créée.

### Périmètre et architecture vérifiés

Base auditée : application `0a3ef5c07c4c9ba58ceaf788244393033acd88c7` (1.1.1), SDK
source `4c2b51a53f7a57c0e1eb1cc41c0cfda0e5c601b0`. Le SDK installé par `uv.lock`
est **1.12.0**, tandis que le checkout SDK consulté affiche 1.12.0.dev0 : les
signatures utiles ont aussi été vérifiées dans le SDK installé.

Lecture intégrale des fichiers racine `AGENTS.md`, `CONTRIBUTING.md`, `README.md`,
`pyproject.toml`, `CLAUDE.md`, de `.env.example` et des workflows de validation ;
aucune instruction d'agent imbriquée trouvée dans l'application. Les règles de
création d'une nouvelle application du SDK ne s'appliquent pas à ce fork d'une
application existante. Le SDK reste une dépendance, sans modification ni copie
de ses composants internes.

| Composant existant | Rôle et réutilisation |
|---|---|
| `main.py`, `app_lifecycle.py` | CLI, classe `ReachyMiniApp`, démarrage, arrêt, sommeil et serveur UI. |
| `console.py`, `streaming.py` | Flux microphone/haut-parleur, lecture audio cadencée, interruptions, reconnexion, JSON-RPC `/rpc`, historique temporaire et diagnostic. |
| `huggingface_realtime.py` | Session WebSocket, transcription, réponses audio, outils, images, envoi sérialisé des demandes de réponse. Un seul backend logique Hugging Face, avec serveur déployé ou direct local. |
| `audio/local_vad.py` | Silero ONNX local, barge-in avec confirmation du VAD serveur. Dégradation vers le VAD serveur en cas d'échec. |
| `moves.py`, `dance_emotion_moves.py` | File de mouvements, respiration, danse, émotions et mouvement pendant la parole. |
| `tools/core_tools.py`, `background_tool_manager.py` | Découverte et exécution des outils internes/externes, tâches asynchrones, annulation et état. |
| `personality.py`, `profile_store.py`, `profile_toolsets.py` | Profils déclaratifs, personnalités utilisateur, sélection des outils par profil. |
| `memory.py`, `memory_routes.py` | Mémoire JSON commune, 60 faits de 280 caractères maximum ; consultation, suppression, activation et actualisation du prompt. |
| `idle_policy.py`, `ConversationHandler.emit()` | Initiatives locales après 180 secondes d'inactivité, si les mouvements et la réponse en cours le permettent. |
| `settings_store.py`, `startup_settings.py`, `static/` | Paramètres persistants et interface existante à étendre ; aucun second panneau d'administration nécessaire. |

La conversation, le suivi de tête, les outils caméra, les profils, la mémoire
commune, les diagnostics VAD/latence et le choix local/déployé sont réutilisables.
Le suivi de visage public du SDK suit un visage proche, sans identité ni
identification du locuteur. Le daemon utilise déjà YuNet avec ONNX Runtime ;
`get_tracked_face()` ne fournit pas une galerie d'identités ni les cinq points
nécessaires à SFace. Ne pas importer son détecteur interne dans le fork.

### Arbitrages et modifications nécessaires

**Visages (phase 2).** Prévoir un module optionnel, désactivé par défaut, avec
une tâche à faible fréquence lisant `robot.media.get_frame()` ; ne jamais ouvrir
une deuxième caméra OpenCV. Faire l'inférence hors de la boucle asyncio audio,
avec une seule analyse en cours, abandon des images périmées et arrêt borné.
Commencer avec 1 analyse/seconde et un nombre de threads limité, à ajuster par
mesure. Plusieurs visages détectés ne signifient pas plusieurs inférences à
chaque image : réutiliser les observations stables et borner le travail.

Enrôlement : consentement explicite, nom et identifiant stable, 3 à 5 photos,
un seul visage par photo, contrôles de taille, flou et pose, empreintes et
modèles versionnés. Prévoir liste, mise à jour, suppression complète, seuil de
similarité, marge entre les deux meilleurs candidats, état inconnu et plusieurs
observations concordantes. Aucun seuil de publication ne vaut calibration pour
ce foyer. Une photo peut tromper ce dispositif : ce n'est pas une authentification.
Photos et empreintes resteront hors Git et ne seront jamais ajoutées au prompt,
aux outils distants ou aux journaux. Répertoire privé (0700) et fichiers privés
(0600) sur les systèmes POSIX, permissions équivalentes à documenter ailleurs.

**Mémoire (phase 3).** Conserver le JSON commun et ses commandes. Ajouter un
stockage séparé par identifiant, probablement SQLite de la bibliothèque standard
pour les transactions, la rétention et la suppression. Ne pas convertir tous les
anciens faits en souvenirs attribués à une personne. L'identité visuelle est un
contexte de présence ; l'identité du locuteur exige une confirmation explicite.
En présence ambiguë, rester sur la mémoire commune. Tester accès croisés,
suppression, expiration, changements d'interlocuteur et absence d'identité.

**Confidentialité.** Aujourd'hui la mémoire activée est injectée dans les
instructions du backend sélectionné ; le stockage local ne garantit donc pas
l'absence d'envoi au cloud. L'outil caméra transmet aussi des images au backend.
Pour les premiers essais privés : désactiver mémoire et caméra, ou sélectionner
un endpoint local vérifié et désactiver les outils distants du profil. Les futurs
souvenirs personnels nécessitent une politique explicite de destination avant
injection. L'enrôlement local ne devra jamais appeler l'outil caméra conversationnel.

**Initiatives (phase 4).** Réutiliser `ConversationHandler.say()` et son envoi
sérialisé existant : cette API injecte une instruction que le modèle peut
reformuler, ce n'est pas une synthèse littérale. Ajouter une file d'événements
bornée distincte de l'audio : priorité, expiration, déduplication, délai minimal,
heures de silence avec fuseau, activation par comportement, attente de fin de
parole humaine et de lecture du robot. `say()` seul ne protège pas contre une
injection au milieu d'une parole humaine : prévoir un garde explicite dans le
handler partagé plutôt qu'une seconde session audio. Les salutations doivent
suivre les transitions de présence, avec temporisation, jamais chaque image.

**IA (phase 5).** Le choix officiel/local existe déjà ; ne pas ressusciter les
anciens sélecteurs `BACKEND_PROVIDER` et `MODEL_NAME`. La piste prioritaire est
[speech-to-speech](https://github.com/huggingface/speech-to-speech), serveur vocal
compatible avec le protocole Realtime attendu. Sa documentation prévoit Apple
Silicon/MLX et 16 Go avec un petit modèle quantifié. llama.cpp peut alimenter son
étage LLM ; Ollama demande une vérification équivalente de l'adaptateur texte.
Ces serveurs texte seuls ne remplacent pas le serveur vocal Realtime. Aucun
serveur vocal ni modèle LLM n'a été installé ou mesuré pendant la phase 1.
Commencer par la sélection entre sessions. Un basculement automatique devra
préserver explicitement l'historique et les outils, gérer les délais et ne jamais
exporter silencieusement des souvenirs. Les reconnexions actuelles ne prouvent
pas cette continuité. Aucun fallback cloud automatique en phase 1.

Le mode déployé officiel ne requiert pas de clé API OpenAI dans l'application,
mais sa disponibilité et ses conditions futures ne sont pas garanties. Le paquet
`openai` sert de client de protocole ; il n'implique pas de facturation OpenAI.
Un fonctionnement vocal entièrement autonome, gratuit et sans Mac sur CM4 n'est
pas démontré. Mac éteint : la vision locale et les outils locaux pourront rester
actifs, mais la conversation dépendra d'un serveur vocal disponible. Le NAS peut
héberger de petits services et sauvegardes ; ne pas présumer qu'il peut remplacer
le Mac pour l'inférence vocale.

**Intégrations (phase 6).** Synology : outils Python externes locaux, client
`httpx` déjà présent, découverte via `SYNO.API.Info`, authentification officielle,
TLS, délais bornés, compte minimal et lecture seule initialement. Vérifier sur
la version DSM réellement installée les API disponibles pour sauvegardes et
conteneurs ; ne pas inventer de routes ni supposer des permissions Docker.
Une action sensible exige une confirmation portant sur ses paramètres et sa
validité, contrôlée côté outil. Ne pas confier cette sécurité au seul prompt.
Référence : [documentation développeur Synology](https://www.synology.com/en-global/support/developer).

Telegram : API Bot officielle avec `httpx`, liste d'identifiants numériques
autorisés avant tout traitement, envoi texte puis `sendVoice`, réception par
long polling avec timeout, téléchargement et taille bornés, validation des types
et suppression des fichiers temporaires. Secrets hors Git. Telegram transmet
les messages à son infrastructure : destination à autoriser explicitement pour
les contenus privés. Les voix entrantes exigent un chemin STT à étudier ; ne pas
ouvrir une deuxième session vocale du robot. Les appels ne font pas partie de
la [Bot API standard](https://core.telegram.org/bots/api) ; MTProto est hors périmètre.
L'échec de ces intégrations ne devra pas empêcher le démarrage de l'application.

### Dépendances et licences envisagées

Aucune dépendance applicative ajoutée en phase 1 ; `uv.lock` est inchangé.

| Élément | Justification, licence et réserve |
|---|---|
| Application et SDK | Apache-2.0 ; conserver les notices. Le SDK reste installé depuis le lock. |
| OpenCV CPU | Candidat pour `FaceDetectorYN` / `FaceRecognizerSF`, sans GPU ni CUDA. OpenCV récent : Apache-2.0 ; packaging opencv-python : MIT et notices tierces. Choisir un seul paquet `cv2`, headless si compatible avec l'installation SDK ; ne pas installer les variantes en parallèle. |
| YuNet OpenCV Zoo | [MIT](https://github.com/opencv/opencv_zoo/blob/main/models/face_detection_yunet/LICENSE), détection et cinq points ; version et hash des poids à figer avant intégration. |
| SFace OpenCV Zoo | [Apache-2.0](https://github.com/opencv/opencv_zoo/blob/main/models/face_recognition_sface/LICENSE), empreintes locales. Vérifier ensemble conversion ONNX, alignement et API OpenCV. |
| ONNX Runtime / NumPy | Déjà transitifs via le SDK, MIT / BSD-3-Clause ; éviter un deuxième runtime inutile. |
| SQLite | Module Python standard, moteur dans le domaine public ; pas de serveur additionnel. |
| speech-to-speech, MLX | Serveur séparé sur Mac : Apache-2.0 / MIT. Ne pas alourdir les dépendances embarquées du fork. |
| Modèles vocaux/LLM | Qwen3.5-4B et Qwen3-TTS 0.6B CustomVoice : fiches officielles Apache-2.0. Chaque checkpoint STT et conversion MLX devra avoir sa propre provenance/licence vérifiée avant choix final ; la fiche de la conversion MLX consultée n'affiche pas de model card. |
| Synology / Telegram | Réutiliser `httpx` (BSD-3-Clause) ; aucun SDK tiers nécessaire pour le premier incrément. |

Sources modèles : [Qwen3.5](https://huggingface.co/Qwen/Qwen3.5-4B),
[Qwen3-TTS](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice),
[conversion MLX envisagée](https://huggingface.co/mlx-community/Qwen3.5-4B-4bit).
Les roues GStreamer installées par le SDK incluent des composants GPL et des
plugins restreints : conserver leurs notices et examiner leurs conditions avant
toute redistribution d'une image ou d'un bundle binaire. Une licence de code
ne remplace pas la vérification de celle des poids.

### Matériel, mesures et limites

Le [matériel officiel](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/platforms/reachy_mini/hardware.md)
indique CM4 ARM64, 4 Go de RAM, 16 Go de flash, caméra IMX708 grand angle et audio
XMOS. OpenCV Zoo publie des essais CPU sur Raspberry Pi 4B : environ 6,23 ms pour
YuNet à 160×120 et 68,82 ms pour SFace à 150×150 dans son
[benchmark](https://github.com/opencv/opencv_zoo/tree/main/benchmark).
Ce sont des références sur Pi 4B, pas des mesures sur Reachy CM4 chargé par
l'audio, les mouvements et le daemon. Le multi-visage augmente le coût SFace.
PyPI publie des roues `opencv-python-headless` 5.0.0.93 pour macOS ARM64 et
Linux aarch64 (manylinux 2.17 et 2.28), vérifiées sans installation. Cela ne
valide pas cette version pour le SDK : son extra OpenCV impose actuellement
`opencv-python<=5.0`. Choisir et verrouiller une version compatible avec le
SDK, Python, NumPy et l’OS livré, puis tester ses API et les modèles.

Mesurer : durée d'inférence p50/p95, CPU du processus et du système, RSS,
température, images périmées, cadence réelle, délai vocal, interruptions et
arrêt des tâches. Les futurs objectifs doivent être confrontés à une mesure de
référence de l'application officielle, avec et sans reconnaissance activée.
Tester inconnus, lumière faible, lunettes, angles, mouvement, deux personnes
et progression vers dix identités consentantes. L'exactitude LFW publiée par
SFace ne garantit pas les résultats domestiques.

Le simulateur reproduit la mécanique, pas l'optique réelle, l'IMU Wireless,
l'écho XMOS, le Wi-Fi, la batterie ou l'échauffement du CM4. L'audio de simulation
utilise les périphériques du Mac et un traitement logiciel d'écho. Aucune
validation sur robot physique n'est revendiquée avant livraison.

### Phases et critères de sortie

| Phase | Incrément indépendant | Validation requise |
|---|---|---|
| 0 | Audit, points d'extension, risques et licences | Sources et versions identifiées, distinction présence/locuteur, aucune dépendance payante imposée. |
| 1 | Vrai fork, remotes, branche de travail, environnement verrouillé | Ruff, format, mypy, pytest ; connexion SDK MuJoCo ; distinguer démarrage UI et échange vocal. |
| 2 | Enrôlement, galerie locale, reconnaissance optionnelle, UI minimale | Tests images synthétiques/fixtures non biométriques, multi-visage/inconnu/stabilisation/suppression ; modèles publics pour essais consentis hors Git ; mesures CM4 en attente. |
| 3 | Mémoire personnelle séparée et contexte explicitement confirmé | Tests de non-mélange, export interdit vers cloud par défaut, rétention et suppression ; mémoire commune inchangée. |
| 4 | File d'événements, salutations et comportements optionnels | Horloge simulée, priorités, délais, silence, parole active, absence de répétition, arrêt propre ; une seule session. |
| 5 | Serveur local éprouvé, sélection entre sessions, étude auto | Français, STT/TTS, outils, caméra, interruption, panne/récupération, mémoire et latence sur M4 ; auto seulement après preuve de continuité et consentement cloud. |
| 6 | Synology lecture seule, puis Telegram texte/voix par incréments | Réponses simulées, authentification/pannes, refus des utilisateurs interdits et des écritures non confirmées ; essai réel séparé. |
| 7 | Installation Wireless, sauvegarde, démarrage et retour officiel | Validation physique, une app à la fois, données restaurables, mise à jour et retour arrière éprouvés. |

Outils Synology et profils personnels peuvent se développer sans modifier le
moteur grâce aux répertoires externes. Le serveur vocal local peut aussi être
configuré sans fork. La reconnaissance continue, sa gestion UI, la mémoire par
personne et l'ordonnancement d'initiatives demandent des extensions ciblées du
cycle de vie et du contexte. Aucun de ces modules n'est introduit dans cette base.

### Installation Mac et simulation

Installer `uv`, Python 3.12 et Git LFS selon leurs distributions officielles.
Git LFS est indispensable : sans lui les avatars restent des pointeurs texte.
Dans ce workspace, des binaires locaux `uv` et `git-lfs` sont disponibles dans
`../.tools/bin` ; ajouter ce dossier au PATH avant les commandes ci-dessous.

```bash
# Depuis le répertoire du fork déjà cloné dans ce workspace :
export PATH="$(cd ../.tools/bin && pwd):$PATH"
git lfs pull upstream
uv sync --frozen --group dev --python 3.12

# Extra de simulation, sans changer les dépendances de l'application :
uv pip install --python .venv/bin/python 'reachy-mini[mujoco]==1.12.0'
source .venv/bin/activate
mjpython -m reachy_mini.daemon.app.main --sim
```

Dans un second terminal, activer le même venv, puis lancer
`reachy-mini-conversation-app --ui`. Le daemon doit rester actif.
Pour les essais privés, créer `.env` depuis `.env.example`, désactiver mémoire
et caméra et sélectionner un serveur local disponible. Les préférences déjà
sauvées dans l'UI prévalent sur les variables d'environnement.

Le serveur vocal local est une étape ultérieure : suivre son guide officiel,
puis configurer `HF_REALTIME_CONNECTION_MODE=local` et `HF_REALTIME_WS_URL` vers
son endpoint `/v1/realtime`. Aucun téléchargement de modèles LLM n'est nécessaire
aux tests unitaires du fork. Une nouvelle exécution de `uv sync` peut retirer
les paquets MuJoCo installés hors lock ; réinstaller l'extra pour la simulation.

Pour vérifier seulement MuJoCo et le SDK sans audio ni caméra :

```bash
mjpython -m reachy_mini.daemon.app.main --sim --headless --no-media \
  --no-wake-up-on-start --no-goto-sleep-on-stop \
  --dataset-update-interval 0 --fastapi-port 18000 --robot-name deadbone_phase1
```

Ce mode valide le daemon et la télémétrie SDK, pas la conversation. Ne pas le
laisser tourner avec le flux audio de l'app : l'essai de phase 1 a montré une
boucle de warnings très rapide lorsque l'audio est absent. C'est une limite de
la base officielle à traiter dans un incrément `fix/` séparé si ce mode devient
un scénario d'usage ; ne pas contourner le SDK ni désactiver les logs globalement.

### Installation Wireless et protection des données

Procédure à revérifier à réception avec la version réellement livrée, selon le
[guide officiel Wireless](https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/platforms/reachy_mini/development_workflow.md).
Ne pas exécuter ces opérations maintenant : aucun robot n'est disponible.

Conserver l'installation officielle du gestionnaire d'apps. Pour le fork,
préférer un checkout et un venv dédiés dans le compte utilisateur, avec un
répertoire de données distinct ; arrêter l'app officielle dans Reachy Mini
Control et désactiver son lancement automatique avant de démarrer le fork.
Le SDK officiel documente `/venvs/apps_venv` pour l'environnement partagé : ne
pas y installer ce fork sous le même nom de package, cela remplacerait l'app
originale. Un venv distinct conserve le même package et la même CLI sans collision.

Sur le robot uniquement, après confirmation du compte SSH et de son hostname :

```bash
# Dans un répertoire utilisateur choisi, hors installation officielle :
git clone https://github.com/deadbone/reachy_mini_conversation_app.git
cd reachy_mini_conversation_app
git checkout chore/personal-fork-baseline
python3 -m venv .venv
.venv/bin/python -m pip install -e .
.venv/bin/reachy-mini-conversation-app --ui
```

Cette variante pip n'utilise pas `uv.lock` : pour une reproduction exacte,
utiliser `uv sync --frozen` lorsque uv et les dépendances système sont disponibles.
Sur Linux, suivre les prérequis GStreamer officiels ; ne pas copier les roues Mac.
La compatibilité de cet environnement isolé avec l'IPC/média sera à vérifier sur
le robot. Aucun service systemd personnel ni enregistrement doublonné dans le
catalogue n'est créé en phase 1.

Les lancements CLI utilisent des emplacements différents des instances gérées :
`memory.v1.json` sous XDG data (`~/.local/share/reachy_mini_conversation_app` par
défaut) et contenu utilisateur sous `external_content/`. Les instances gérées
résolvent leur dossier avec `resolve_instance_path()` et
`REACHY_MINI_INSTANCE_PATH` défini **dans l'environnement du processus**. Cette
variable seule ne fait pas passer la CLI dans le mode instance gérée. Ne pas
promettre une isolation des données sur la seule base d'un `.env`.

Pour isoler les essais CLI, utiliser un répertoire de travail et un
`XDG_DATA_HOME` dédiés, puis lancer la CLI installée par son chemin absolu.
Sauvegarder, application arrêtée : `.env`, mémoire, `settings.json`,
`startup_settings.json`, personnalités, `profile_toolsets.json`, manifeste des
Tools Spaces et futurs stores biométriques/personnels. Copier aussi les outils
externes. Protéger et chiffrer les sauvegardes NAS, limiter leurs permissions ;
la suppression biométrique doit inclure leur rétention. Aucun de ces contenus
ne doit être ajouté à Git. `/personal_data/` et `/backups/` sont ignorés dans ce fork.

Retour officiel : arrêter le processus du fork, fermer ses médias, réactiver
l'app officielle depuis Reachy Mini Control avec ses propres données. Ne pas
lancer simultanément deux apps ni réinstaller le SDK système. Le démarrage
au boot du fork et sa validation physique restent en phase 7.

### Synchronisation et retour à une base connue

`origin` désigne `deadbone/reachy_mini_conversation_app`, `upstream` le dépôt
Pollen Robotics. `main` conserve l'historique officiel et suit `origin/main`.
Les changements de cette préparation restent sur `chore/personal-fork-baseline`.
Ne pas pousser sur upstream et ne pas forcer les branches.

Pour une base sans commits personnels sur main, avec arbre de travail propre :

```bash
git fetch origin
git fetch upstream
git switch main
git merge --ff-only origin/main
git merge --ff-only upstream/main
# Après les vérifications :
git push origin main
git switch -c feat/local-face-recognition
```

Si `main` contient plus tard des commits personnels, effectuer l'intégration sur
une branche `chore/sync-upstream-<date>` : partir de `main`, fusionner
`upstream/main`, résoudre les conflits puis exécuter la gate avant intégration
au fork. Ne pas utiliser `reset --hard` ou réécrire l'historique publié. Sauvegarder
les données avant une migration, conserver le commit précédemment validé et
utiliser un checkout/venv distinct pour revenir à ce commit.

```bash
uv run ruff check . --fix
uv run ruff format .
uv run mypy --pretty --show-error-codes
uv run pytest tests/ -v
```

Examiner le diff après les corrections automatiques. Ne lancer ni workflow de
release ni miroir HF sans configuration personnelle explicite. Les workflows,
CONTRIBUTING, CODE_OF_CONDUCT et modèles de PR restent inchangés.

### Résultat de la préparation du 16 octobre 2026

- Vrai fork GitHub créé et parent vérifié ; historique complet de l'application conservé.
- Python 3.12.14 ARM64, installation de 123 paquets depuis le lock, SDK 1.12.0.
- Ruff : réussi ; format : 104 fichiers inchangés ; mypy : 53 fichiers sans erreur.
- Pytest complet : **578 tests réussis** après récupération Git LFS et autorisation des sockets de test MCP locales. Premier passage : 572 réussis, 4 échecs et 2 erreurs dus aux avatars LFS, à uv absent du PATH des sous-processus et aux sockets bloquées par le sandbox.
- Extra MuJoCo 3.3.0 installé hors dépendances applicatives ; daemon réellement lancé avec `--sim --headless --no-media`, statut simulation confirmé par SDK et pose 4×4 finie lue via `get_current_head_pose()`.
- Application démarrée sur ce simulateur avec mémoire/caméra/VAD local désactivés et backend local volontairement absent ; page UI récupérée par HTTP. Le flux sans audio a produit des warnings en boucle : cet essai ne valide pas le fonctionnement vocal et n'est pas un mode d'exploitation recommandé.
- Échange vocal réel, serveur local MLX, caméra, CM4, NAS et Telegram : **non testés**. La phase 1 constitue une base logicielle testée, avec validation vocale restant à compléter ; aucune validation matérielle implicite.
- Seuls README racine, commentaires `.env.example` et `.gitignore` sont modifiés. Aucun code moteur, nouvelle option runtime, dépendance, workflow ou SVG généré modifié. Les fonctions personnelles ne sont pas implémentées à ce stade.

Prochain incrément : phase 2, d'abord galerie et enrôlement testables sans robot,
puis tâche de reconnaissance locale et UI. Compléter en parallèle la validation
vocale de la base avec un serveur Realtime choisi et un véritable média de test.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the development workflow and [`AGENTS.md`](AGENTS.md) for coding-agent standards.

## License

Apache 2.0
