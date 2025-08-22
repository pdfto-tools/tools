# From PDF to PNG to ZIP — a preview-free, streaming-first pipeline in the browser

## TL;DR
We built a browser-only pipeline that takes a local PDF, rasterizes each page to PNG, and streams those PNGs into a ZIP file while writing directly to disk (when available) or falling back to a streaming download. The UX avoids rendering pages in the DOM and focuses on fast, cancellable export with a persistent progress bar and a scrollable log.

- No in-page PDF preview — we go straight from input PDF to exported ZIP.
- Save & Start Export button ensures the save dialog is triggered by a user gesture.
- Progress, live logs, and a Cancel button are included.
- Two saving paths:
  - [File System Access (FSA) API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API) for native-like disk writing on supported Chromium browsers.
  - [StreamSaver.js](https://github.com/jimmywarting/StreamSaver.js) fallback that streams the ZIP to a downloadable file via a local `mitm.html`.

The implementation lives in `index.html`.

➡️ **Live demo**: [pdfto.tools/pdf-to-png](https://www.pdfto.tools/pdf-to-png)

---

## Goals

- Provide an efficient way to bulk-export PDF pages as images.
- Keep users in control: explicit save location, visible progress, and cancel at any time.
- Stream data as it is produced to avoid large memory spikes for big documents.
- Work across modern browsers by feature-detecting I/O capabilities and falling back when necessary.

---

## UX design highlights

- Select a PDF file.
- Click "Save & Start Export" — this click launches the system save dialog and begins the pipeline.
- Watch the progress bar and a continuously updating, scrollable activity log.
- Optional: Cancel at any time; the pipeline aborts ongoing work and closes resources.

We deliberately removed any in-page rendering of PDF pages. All work happens offscreen, dedicated to export speed and correctness.

---

## Architecture overview

- MuPDF WebAssembly is used to open and iterate the PDF.
- Each page is rendered to an OffscreenCanvas and encoded to PNG.
- A ZIP writer consumes each PNG as soon as it is ready — no need to buffer all pages.
- The ZIP stream is written to a sink selected at runtime:
  - FSA (showSaveFilePicker + createWritable) for direct-to-disk streaming.
  - StreamSaver.js for streaming downloads when FSA is not available.

This design eliminates waiting for the entire export to finish before writing to disk and keeps memory usage bounded.

---

## Flow in detail

1) User chooses a PDF.
2) User clicks "Save & Start Export" (user gesture requirement for native save dialogs).
3) The app picks the best available sink:
   - FSA write handle (native-like, continuous write).
   - Otherwise, StreamSaver with a same-origin `mitm.html` for download streaming.
4) The export loop:
   - Open the PDF with MuPDF WASM.
   - For each page: render → encode PNG → ZIP entry write.
   - Update progress and log; check for cancel requests.
5) Finalization:
   - Close ZIP stream, close writer, show success (or aborted) status.

---

## A unified writer interface

Internally we normalize both sinks to a minimal, common interface so the rest of the pipeline doesn’t care which sink is used:

```js
const writable = {
  write: (chunk) => /* write Uint8Array or ArrayBuffer */, 
  close: () => /* finalize file */, 
  abort: (reason) => /* abort writing */
}
```

- FSA path: `showSaveFilePicker()` → `createWritable()` gives us a native writer.
- StreamSaver path: `createWriteStream()` returns a writer-compatible stream that we adapt to the same interface.

The pipeline only calls `writable.write()`, `writable.close()`, and `writable.abort()`; switching sinks becomes a one-liner decision.

---

## Ensuring the save dialog is user-gesture driven

Browsers require that the native save dialog be triggered directly from a user action. We solve this by:

- Storing the selected PDF after the file input change.
- Deferring the save dialog until the user clicks a dedicated "Save & Start Export" button.
- Starting the export only inside that click handler.

This removes intermittent "must be called in a user activation" errors and makes the behavior predictable.

---

## Progress, Logs, and Cancellation

- The progress bar is based on page count: completed/total.
- The log shows milestones (file chosen, sink selected, page renders, zip entries, completion/abort, errors).
- Cancellation toggles a shared flag and calls `writable.abort()`. The render loop observes the flag, stops gracefully, and releases resources.

Together these make long exports transparent and controllable.

---

## StreamSaver fallback with a local MITM page

When FSA is unavailable, we fall back to StreamSaver. To keep everything self-contained and same-origin, we ship a local MITM page:

- `mitm.html`
- We set the MITM URL to `./mitm.html?version=2.0.0` at runtime.
- Streamed ZIPs download as they’re produced; users see a regular browser download without large memory spikes.

This integrates seamlessly with the same unified writer interface.

---

## Errors and resilience

- The export routine is wrapped with try/catch and always-finally semantics to close resources.
- We surface errors to the UI log and reset internal state, so the user can retry.
- On abort, we attempt to close or cancel active operations quickly to avoid partial files.

---

## Browser support strategy

- Prefer FSA when available for the best native-like experience.
- Automatically fall back to StreamSaver when FSA is not present.
- If neither sink is available, we show a helpful message instructing users to try a Chromium browser or enable the fallback-ready environment.

This layered approach brings good defaults without complex configuration.

---

## Testing checklist

- Small PDFs (1–5 pages): verify instant progress, correct ZIP content.
- Large PDFs (100+ pages): confirm steady progress, no memory spikes, responsive cancel.
- Cancel mid-export: ensure the file is not left in a corrupt partial state and the UI resets.
- Sink switching:
  - FSA-enabled browser → native save dialog, direct write.
  - FSA-disabled or non-Chromium → StreamSaver download.
- Pathological inputs: encrypted PDFs or unusual page sizes — confirm graceful error reporting.

---

## What to customize next

- Image quality: adjust scale/DPI per page depending on your requirements.
- Naming: use per-page filenames inside the ZIP (e.g., `page-001.png`, `page-002.png`).
- Concurrency: render multiple pages concurrently up to a safe limit to balance CPU vs. memory.
- Metadata: include a manifest (JSON) inside the ZIP that records page sizes or timestamps.
- UI polish: add estimated time remaining, counters, or a summary card on completion.

---

## Files of interest

- `index.html` — UI, feature detection, pipeline orchestration, progress/cancel logic.
- `mitm.html` — local MITM page for StreamSaver fallback.

If you want to plug this into your own app, keep the unified-writer abstraction and the user-gesture pattern — those two decisions make everything else straightforward.