## IXL AI Auto-Solver

is a Chrome Manifest V3 browser extension that automatically solves IXL homework and practice questions using AI vision models. When you click the floating button on any IXL page, the extension captures a screenshot of the visible tab, precisely crops the region containing the question using the browser's `OffscreenCanvas` API with full device-pixel-ratio awareness, and sends the cropped image to your chosen AI provider. The AI is asked to solve the problem and return an exact answer — the value you would type or click in IXL — along with a one-to-two sentence explanation of the reasoning. Before returning that answer to you, the extension makes a second, fully independent API call to the same model, showing it the same image and the proposed answer and asking it to solve the problem from scratch and assess whether the first answer is correct. If the second call disagrees, the extension automatically adopts the corrected answer, flags it with a yellow warning banner showing the original wrong answer for transparency, and surfaces the correction reasoning. Only after both calls complete does the final answer appear in the floating answer card and get automatically copied to your clipboard.

---

## Floating Button

The UI entry point is a pill-shaped floating button injected into every IXL page. It sits at the bottom-right corner by default and can be dragged anywhere on screen using pointer capture events with a six-pixel drag threshold to distinguish intentional drags from taps. Its position is saved to `localStorage` as `{ right, bottom }` pixel offsets and restored across page loads. The button has a layered box-shadow with a green border, a white-to-green-tinted gradient background, and a spring hover animation using `cubic-bezier(0.34, 1.56, 0.64, 1)`. When idle, it pulses with a subtle breathing ring animation. When a solve is running it shows a 🧠 brain icon; when the verification step is active it switches to a 🔍 magnifying glass. A green badge counter in the top-right corner increments with every successfully solved question and persists for the duration of the page session.

---

## Answer Card

The answer card appears anchored above the button position. It shows the final answer in large text, the explanation from the first AI call, and the verification verdict from the second call. If the answer was auto-corrected, a yellow banner shows the original wrong answer alongside the correction. The card has Copy and Dismiss buttons and auto-dismisses on any outside click.

---

## Two-Call Verification Architecture

The solve flow is:

1. Content script captures the tab screenshot via `chrome.runtime.sendMessage` → background caches it by integer ID.
2. Content opens a `chrome.runtime.connect` port (keeps the Service Worker alive across both long API calls).
3. Background crops the image, sends `{ status: "solving" }` back, makes the first API call with the solve prompt.
4. Parses `ANSWER:` and `EXPLAIN:` fields from the response.
5. Sends `{ status: "verifying" }`, makes the second API call with the verify prompt (includes the proposed answer).
6. Parses `CORRECT:`, `VERDICT:`, and `CORRECTION:` fields.
7. Reconciles: if `CORRECT: no` and `CORRECTION` is non-empty, the correction becomes the final answer.
8. Saves to history, posts the final message back through the port.

If the verification call fails for any reason (timeout, API error), the extension logs the failure and returns the initial answer without blocking the result.

---

## AI Providers

Every provider's API key is stored independently in `chrome.storage.local`. All keys persist simultaneously so you can switch providers freely without re-entering credentials. API calls are made from the background Service Worker to satisfy IXL's strict Content Security Policy, which blocks fetches from content scripts.

### NVIDIA NIM
OpenAI-compatible endpoint at `https://integrate.api.nvidia.com/v1/chat/completions`. Images passed as `image_url` content type with a data URI. Free API keys at build.nvidia.com.
- `nvidia/llama-3.1-nemotron-nano-vl-8b-v1` — Nano 8B (fastest)
- `meta/llama-3.2-11b-vision-instruct` — Llama 3.2 11B Vision
- `meta/llama-3.2-90b-vision-instruct` — Llama 3.2 90B Vision
- `minimaxai/minimax-m3` — MiniMax M3 427B (highest accuracy, default)

### OpenAI
`https://api.openai.com/v1/chat/completions`. Standard OpenAI vision API with `image_url` content blocks. Keys at platform.openai.com.
- `gpt-4o-mini` — GPT-4o Mini (fast, cheap)
- `gpt-4o` — GPT-4o (balanced, default)
- `o1` — o1 (best reasoning, slowest)

### Anthropic Claude
`https://api.anthropic.com/v1/messages`. Uses Anthropic's native messages format: image is passed as a `source` block with `type: "base64"` and `media_type: "image/jpeg"`. Requires `x-api-key` and `anthropic-version: 2023-06-01` headers. Keys at console.anthropic.com.
- `claude-haiku-4-5-20251001` — Haiku (fastest)
- `claude-sonnet-5` — Sonnet (balanced, default)
- `claude-opus-5` — Opus (smartest)

### Google Gemini
`https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key={apiKey}`. Uses Google's native generateContent format: image is passed as `inline_data` with `mime_type: "image/jpeg"` and a base64 data string inside a `contents.parts` array. This is the only provider using a completely different API shape. Keys at aistudio.google.com.
- `gemini-2.0-flash` — Flash (fastest, default)
- `gemini-1.5-pro` — 1.5 Pro (balanced)
- `gemini-2.5-pro` — 2.5 Pro (smartest)

### Groq
OpenAI-compatible at `https://api.groq.com/openai/v1/chat/completions`. Ultra-fast inference. Images via `image_url`. Keys at console.groq.com.
- `llama-3.2-11b-vision-preview` — Llama 3.2 11B Vision (default)
- `llama-3.2-90b-vision-preview` — Llama 3.2 90B Vision

### Together AI
OpenAI-compatible at `https://api.together.ai/v1/chat/completions`. Images via `image_url`. Keys at api.together.ai.
- `meta-llama/Llama-3.2-11B-Vision-Instruct-Turbo` — Llama 11B Turbo (default)
- `meta-llama/Llama-3.2-90B-Vision-Instruct-Turbo` — Llama 90B Turbo

### xAI (Grok)
OpenAI-compatible at `https://api.x.ai/v1/chat/completions`. Images via `image_url`. Keys at console.x.ai.
- `grok-2-vision-1212` — Grok 2 Vision (default)
- `grok-vision-beta` — Grok Vision Beta

### Mistral
OpenAI-compatible at `https://api.mistral.ai/v1/chat/completions`. Pixtral models are native vision models trained by Mistral. Images via `image_url`. Keys at console.mistral.ai.
- `pixtral-12b-2409` — Pixtral 12B (default)
- `pixtral-large-2411` — Pixtral Large

### DeepSeek
OpenAI-compatible at `https://api.deepseek.com/v1/chat/completions`. Images via `image_url`. Keys at platform.deepseek.com.
- `deepseek-chat` — DeepSeek V3 (default)
- `deepseek-reasoner` — DeepSeek R1 (chain-of-thought reasoning)

---

## Popup — Settings Tab

Dropdown to select the active provider. Dynamic model picker (segmented button row, Fast → Smart left to right) that updates instantly when the provider changes. API key field for the selected provider, pre-filled if a key is already saved. A single Save button writes the active provider, model, and key to `chrome.storage.local` without touching keys stored for other providers.

## Popup — API Keys Tab

Lists all nine providers in individual cards, each showing the provider name, a green "✓ Saved" or grey "Not set" status badge, a password input pre-filled with any existing key, a per-provider Save button, and a link to that provider's key page. Each provider saves independently — the tab reloads from storage on every visit so it always reflects the true stored state. This is the recommended place to enter all your API keys at once so you can switch providers in Settings without re-entering credentials.

## Popup — History Tab

Scrollable list of the last 50 solved answers, stored in `chrome.storage.local` under `ixlHistory`. Each entry shows a color-coded provider badge (nine distinct colors), the timestamp formatted as abbreviated month + day + time, the model used, and the answer in large text. A Copy button on each entry writes the answer to the clipboard. A Clear History button wipes the list.

---

## Technical Implementation

- **Manifest V3** Service Worker with port-based keepalive (`chrome.runtime.connect`) to survive the 30-second idle shutdown across two sequential API calls (60-second port timeout).
- **Screenshot caching** — background caches the full tab screenshot by integer ID for 90 seconds. Content script receives only the ID, never re-transmits the image back to the background, eliminating the largest IPC bottleneck.
- **Crop pipeline** — `createImageBitmap` → `OffscreenCanvas` → `convertToBlob` → chunked `String.fromCharCode` + `btoa` for blob-to-base64 without `FileReader`. DPR-scaled crop coordinates for correct results on Retina/HiDPI displays.
- **Response parsing** — regex field extraction for structured `KEY: value` format. JSON extraction fallback. Common prefix stripping ("The answer is…", "FINAL ANSWER:…"). Trailing punctuation removal. Backtick/code-fence stripping.
- **All API calls from background** — IXL's CSP blocks external fetches from content scripts. Every API call goes through the background Service Worker via the persistent port.
- **Storage** — `chrome.storage.local` for all API keys, provider selection, model selections per provider, and answer history. `localStorage` for button position only (page-scoped, no cross-origin sharing needed).

<img width="1600" height="182" alt="image" src="https://github.com/user-attachments/assets/66cdb721-65e9-4c2a-9469-f4538a5da4cb" />


