# EBM
Take off manager to upload into EBM/ personal estimator through a drawing 
 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/README.md b/README.md
index da68454743c39e15e2a773ecf6d296121877bb1c..6bebf1a51840c9b67d8e385990464e8dd1554e90 100644
--- a/README.md
+++ b/README.md
@@ -1,2 +1,21 @@
-# EBM
-Take off manager to upload into EBM/ personal estimator through a drawing 
+# EBM PDF Reader
+
+A lightweight browser-based PDF reader built with vanilla JavaScript and PDF.js.
+
+## Features
+
+- Open local PDF files
+- Previous/next page navigation
+- Jump directly to a page number
+- Zoom in/out controls
+- Loading and render status messages
+
+## Run locally
+
+Because modules are loaded in the browser, use a local server:
+
+```bash
+python3 -m http.server 8000
+```
+
+Then open `http://localhost:8000` in your browser.
 
EOF
)
 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/index.html b/index.html
new file mode 100644
index 0000000000000000000000000000000000000000..3075f30d0394abe74dc309d65245aee76433b392
--- /dev/null
+++ b/index.html
@@ -0,0 +1,43 @@
+<!DOCTYPE html>
+<html lang="en">
+  <head>
+    <meta charset="UTF-8" />
+    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
+    <title>PDF Reader</title>
+    <link rel="stylesheet" href="styles.css" />
+    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.min.mjs" type="module"></script>
+  </head>
+  <body>
+    <main class="app">
+      <h1>PDF Reader</h1>
+
+      <section class="toolbar">
+        <label class="upload">
+          <span>Open PDF</span>
+          <input id="file-input" type="file" accept="application/pdf" />
+        </label>
+
+        <button id="prev-page" disabled>Previous</button>
+        <button id="next-page" disabled>Next</button>
+
+        <label>
+          Page
+          <input id="page-number" type="number" min="1" value="1" disabled />
+          <span id="page-count">/ 0</span>
+        </label>
+
+        <button id="zoom-out" disabled>-</button>
+        <span id="zoom-level">100%</span>
+        <button id="zoom-in" disabled>+</button>
+      </section>
+
+      <p id="status" role="status">Choose a PDF file to begin.</p>
+
+      <section class="viewer-wrap">
+        <canvas id="pdf-canvas"></canvas>
+      </section>
+    </main>
+
+    <script type="module" src="reader.js"></script>
+  </body>
+</html>
 
EOF
)
 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/reader.js b/reader.js
new file mode 100644
index 0000000000000000000000000000000000000000..419d989c0536f7c391709d7cf2e1872b7b4f18d6
--- /dev/null
+++ b/reader.js
@@ -0,0 +1,135 @@
+import * as pdfjsLib from 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.min.mjs';
+
+pdfjsLib.GlobalWorkerOptions.workerSrc =
+  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.4.168/pdf.worker.min.mjs';
+
+const state = {
+  pdf: null,
+  pageNumber: 1,
+  scale: 1,
+  rendering: false,
+  pendingPage: null,
+};
+
+const fileInput = document.getElementById('file-input');
+const prevBtn = document.getElementById('prev-page');
+const nextBtn = document.getElementById('next-page');
+const pageNumberInput = document.getElementById('page-number');
+const pageCount = document.getElementById('page-count');
+const zoomOutBtn = document.getElementById('zoom-out');
+const zoomInBtn = document.getElementById('zoom-in');
+const zoomLevel = document.getElementById('zoom-level');
+const status = document.getElementById('status');
+const canvas = document.getElementById('pdf-canvas');
+const ctx = canvas.getContext('2d');
+
+function setControlsEnabled(enabled) {
+  prevBtn.disabled = !enabled;
+  nextBtn.disabled = !enabled;
+  pageNumberInput.disabled = !enabled;
+  zoomOutBtn.disabled = !enabled;
+  zoomInBtn.disabled = !enabled;
+}
+
+function setStatus(message) {
+  status.textContent = message;
+}
+
+async function renderPage(pageNumber) {
+  if (!state.pdf) {
+    return;
+  }
+
+  state.rendering = true;
+  setStatus(`Rendering page ${pageNumber}...`);
+
+  const page = await state.pdf.getPage(pageNumber);
+  const viewport = page.getViewport({ scale: state.scale });
+
+  canvas.width = viewport.width;
+  canvas.height = viewport.height;
+
+  await page.render({ canvasContext: ctx, viewport }).promise;
+
+  state.rendering = false;
+  pageNumberInput.value = state.pageNumber;
+  pageCount.textContent = `/ ${state.pdf.numPages}`;
+  zoomLevel.textContent = `${Math.round(state.scale * 100)}%`;
+  setStatus(`Showing page ${state.pageNumber} of ${state.pdf.numPages}.`);
+
+  if (state.pendingPage !== null) {
+    const pending = state.pendingPage;
+    state.pendingPage = null;
+    renderPage(pending);
+  }
+}
+
+function queueRenderPage(pageNumber) {
+  if (state.rendering) {
+    state.pendingPage = pageNumber;
+  } else {
+    renderPage(pageNumber);
+  }
+}
+
+function goToPage(pageNumber) {
+  if (!state.pdf) {
+    return;
+  }
+
+  if (pageNumber < 1 || pageNumber > state.pdf.numPages) {
+    return;
+  }
+
+  state.pageNumber = pageNumber;
+  queueRenderPage(pageNumber);
+}
+
+function changeZoom(step) {
+  if (!state.pdf) {
+    return;
+  }
+
+  const nextScale = Math.min(3, Math.max(0.4, state.scale + step));
+  if (nextScale === state.scale) {
+    return;
+  }
+
+  state.scale = Number(nextScale.toFixed(2));
+  queueRenderPage(state.pageNumber);
+}
+
+fileInput.addEventListener('change', async (event) => {
+  const file = event.target.files?.[0];
+  if (!file) {
+    return;
+  }
+
+  try {
+    setStatus('Loading PDF...');
+    const bytes = await file.arrayBuffer();
+    const loadingTask = pdfjsLib.getDocument({ data: bytes });
+    state.pdf = await loadingTask.promise;
+    state.pageNumber = 1;
+    state.scale = 1;
+    setControlsEnabled(true);
+    pageNumberInput.max = state.pdf.numPages;
+    queueRenderPage(1);
+  } catch (error) {
+    console.error(error);
+    setStatus('Failed to open PDF. Please choose a valid file.');
+    setControlsEnabled(false);
+  }
+});
+
+prevBtn.addEventListener('click', () => goToPage(state.pageNumber - 1));
+nextBtn.addEventListener('click', () => goToPage(state.pageNumber + 1));
+
+pageNumberInput.addEventListener('change', () => {
+  goToPage(Number(pageNumberInput.value));
+});
+
+zoomOutBtn.addEventListener('click', () => changeZoom(-0.1));
+zoomInBtn.addEventListener('click', () => changeZoom(0.1));
+
+setControlsEnabled(false);
 
EOF
) (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/sample.pdf b/sample.pdf
new file mode 100644
index 0000000000000000000000000000000000000000..853897e7355f3c6611153cc59974b02f4e566312
--- /dev/null
+++ b/sample.pdf
@@ -0,0 +1,36 @@
+%PDF-1.1
+1 0 obj
+<< /Type /Catalog /Pages 2 0 R >>
+endobj
+2 0 obj
+<< /Type /Pages /Kids [3 0 R] /Count 1 >>
+endobj
+3 0 obj
+<< /Type /Page /Parent 2 0 R /MediaBox [0 0 300 144] /Contents 4 0 R /Resources << /Font << /F1 5 0 R >> >> >>
+endobj
+4 0 obj
+<< /Length 55 >>
+stream
+BT
+/F1 24 Tf
+72 72 Td
+(Hello PDF Reader) Tj
+ET
+endstream
+endobj
+5 0 obj
+<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica >>
+endobj
+xref
+0 6
+0000000000 65535 f 
+0000000010 00000 n 
+0000000062 00000 n 
+0000000117 00000 n 
+0000000243 00000 n 
+0000000348 00000 n 
+trailer
+<< /Size 6 /Root 1 0 R >>
+startxref
+418
+%%EOF
 
EOF
)
 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/styles.css b/styles.css
new file mode 100644
index 0000000000000000000000000000000000000000..1e3c72d7d79ccc1d294e86504368baa918a27ba3
--- /dev/null
+++ b/styles.css
@@ -0,0 +1,89 @@
+:root {
+  color-scheme: light dark;
+  font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
+}
+
+* {
+  box-sizing: border-box;
+}
+
+body {
+  margin: 0;
+  min-height: 100vh;
+  background: #0f172a;
+  color: #e2e8f0;
+}
+
+.app {
+  display: grid;
+  gap: 1rem;
+  padding: 1.5rem;
+  max-width: 1200px;
+  margin: 0 auto;
+}
+
+h1 {
+  margin: 0;
+  font-size: 1.8rem;
+}
+
+.toolbar {
+  display: flex;
+  flex-wrap: wrap;
+  gap: 0.75rem;
+  align-items: center;
+  background: #1e293b;
+  padding: 0.75rem;
+  border-radius: 0.5rem;
+}
+
+.upload {
+  display: inline-flex;
+  align-items: center;
+  gap: 0.5rem;
+  background: #334155;
+  padding: 0.5rem 0.75rem;
+  border-radius: 0.5rem;
+  cursor: pointer;
+}
+
+.upload input {
+  max-width: 13rem;
+}
+
+button,
+input {
+  padding: 0.45rem 0.65rem;
+  border-radius: 0.4rem;
+  border: 1px solid #475569;
+  background: #0f172a;
+  color: inherit;
+}
+
+button:disabled,
+input:disabled {
+  opacity: 0.45;
+  cursor: not-allowed;
+}
+
+#status {
+  margin: 0;
+  color: #93c5fd;
+}
+
+.viewer-wrap {
+  background: #1e293b;
+  border-radius: 0.75rem;
+  min-height: 65vh;
+  display: grid;
+  place-items: center;
+  padding: 1rem;
+  overflow: auto;
+}
+
+#pdf-canvas {
+  max-width: 100%;
+  background: white;
+  border-radius: 0.4rem;
+  box-shadow: 0 8px 25px rgb(0 0 0 / 35%);
+}
 
EOF
)
