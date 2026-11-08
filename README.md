# How-to-download-protected-view-only-files-from-google-drive??

New Code:
```
(async function driveToPdf() {
  const log  = (...a) => console.log("[pdf]", ...a);
  const warn = (...a) => console.warn("[pdf]", ...a);

  /* ---------- deep DOM walk (shadow roots + same-origin iframes) ---------- */
  function allDocuments(root = document) {
    const out = [], seen = new Set();
    (function walk(node) {
      if (!node || seen.has(node)) return;
      seen.add(node);
      out.push(node);
      for (const el of node.querySelectorAll("*")) {
        if (el.shadowRoot) walk(el.shadowRoot);
        if (el.tagName === "IFRAME" || el.tagName === "FRAME") {
          let inner = null;
          try { inner = el.contentDocument; } catch {}
          if (inner) walk(inner);
        }
      }
    })(root);
    return out;
  }

  /* ---------- scroll so lazy pages render ---------- */
  const sc = document.scrollingElement;
  for (let y = 0; y < sc.scrollHeight; y += Math.round(innerHeight * 0.8)) {
    sc.scrollTop = y;
    await new Promise(r => setTimeout(r, 250));
  }
  sc.scrollTop = 0;
  await new Promise(r => setTimeout(r, 800));

  /* ---------- collect page bitmaps ---------- */
  const canvases = [], imgs = [], seenSrc = new Set();
  for (const doc of allDocuments()) {
    for (const c of doc.querySelectorAll("canvas")) {
      if (c.width < 300 || c.height < 300) continue;
      canvases.push({ node: c, w: c.width, h: c.height });
    }
    for (const img of doc.images) {
      const src = img.currentSrc || img.src || "";
      if (!/^(blob:|data:image\/)/i.test(src)) continue;
      if (img.naturalWidth < 300 || img.naturalHeight < 300) continue;
      if (seenSrc.has(src)) continue;
      seenSrc.add(src);
      imgs.push({ node: img, w: img.naturalWidth, h: img.naturalHeight });
    }
  }
  const sources = canvases.length ? canvases : imgs;
  log("pages found:", sources.length, canvases.length ? "(canvases)" : "(images)");
  if (!sources.length) { warn("Nothing to convert — is the preview open?"); return; }

  /* ---------- rasterise each page to JPEG bytes ---------- */
  const toJpeg = canvas => new Promise((resolve, reject) => {
    canvas.toBlob(b => b ? b.arrayBuffer().then(resolve, reject)
                        : reject(new Error("toBlob returned null")),
                  "image/jpeg", 0.92);
  });

  const pages = [];
  for (let i = 0; i < sources.length; i++) {
    const s = sources[i];
    let canvas;
    if (s.node.tagName === "CANVAS") {
      canvas = s.node;
    } else {
      canvas = document.createElement("canvas");
      canvas.width = s.w; canvas.height = s.h;
      canvas.getContext("2d").drawImage(s.node, 0, 0, s.w, s.h);
    }
    try {
      pages.push({ w: s.w, h: s.h, jpeg: await toJpeg(canvas) });
    } catch (e) { warn(`page ${i + 1}: unreadable`, e); }
    log(`rasterised ${i + 1}/${sources.length}`);
  }
  if (!pages.length) { warn("No page could be rasterised"); return; }

  /* ---------- build the PDF by hand ---------- */
  const enc = new TextEncoder();
  const parts = [];
  let length = 0;
  const offsets = [];
  const put    = u8 => { parts.push(u8); length += u8.length; };
  const putStr = s  => put(enc.encode(s));

  putStr("%PDF-1.4\n");

  const n = pages.length;
  const kids = pages.map((_, i) => `${3 + i * 3} 0 R`).join(" ");

  offsets[1] = length; putStr("1 0 obj\n<< /Type /Catalog /Pages 2 0 R >>\nendobj\n");
  offsets[2] = length; putStr(`2 0 obj\n<< /Type /Pages /Kids [${kids}] /Count ${n} >>\nendobj\n`);

  pages.forEach((p, i) => {
    const po = 3 + i * 3, co = po + 1, io = po + 2;

    offsets[po] = length;
    putStr(`${po} 0 obj\n<< /Type /Page /Parent 2 0 R /MediaBox [0 0 ${p.w} ${p.h}] ` +
           `/Resources << /XObject << /Im0 ${io} 0 R >> >> /Contents ${co} 0 R >>\nendobj\n`);

    const content = `q ${p.w} 0 0 ${p.h} 0 0 cm /Im0 Do Q`;
    offsets[co] = length;
    putStr(`${co} 0 obj\n<< /Length ${content.length} >>\nstream\n${content}\nendstream\nendobj\n`);

    offsets[io] = length;
    putStr(`${io} 0 obj\n<< /Type /XObject /Subtype /Image /Width ${p.w} /Height ${p.h} ` +
           `/ColorSpace /DeviceRGB /BitsPerComponent 8 /Filter /DCTDecode ` +
           `/Length ${p.jpeg.byteLength} >>\nstream\n`);
    put(new Uint8Array(p.jpeg));
    putStr("\nendstream\nendobj\n");
  });

  const last = 2 + n * 3;
  const xrefAt = length;
  putStr(`xref\n0 ${last + 1}\n`);
  putStr("0000000000 65535 f \n");
  for (let i = 1; i <= last; i++) {
    putStr(String(offsets[i]).padStart(10, "0") + " 00000 n \n");
  }
  putStr(`trailer\n<< /Size ${last + 1} /Root 1 0 R >>\nstartxref\n${xrefAt}\n%%EOF\n`);

  const blob = new Blob(parts, { type: "application/pdf" });

  /* ---------- filename: the preview overlay heading ---------- */
  const docs = allDocuments();
  const tidy = s => (s || "")
    .replace(/[\u00a0\u2007\u202f]/g, " ")
    .replace(/\.pdf\s*$/i, "")
    .replace(/\s*[-–—|]\s*Google\s*Drive\s*$/i, "")
    .replace(/[\\/:*?"<>|]/g, "-")
    .replace(/\s+/g, " ")
    .trim();

  let raw = "";
  outer:
  for (const doc of docs) {
    for (const sel of ['[jsname="oYxtQd"]', ".a6Vetd", '[role="heading"][aria-level="2"]']) {
      for (const el of doc.querySelectorAll(sel)) {
        if (el.closest("tr[data-id]")) continue;      // skip folder listing rows
        const t = el.textContent.trim();
        if (/\.pdf$/i.test(t) && t.length < 200) { raw = t; break outer; }
      }
    }
  }
  if (!raw) {
    outer2:
    for (const doc of docs) {
      for (const el of doc.querySelectorAll("span,div,strong")) {
        if (el.children.length || el.closest("tr[data-id]")) continue;
        const t = el.textContent.trim();
        if (/\.pdf$/i.test(t) && t.length < 200) { raw = t; break outer2; }
      }
    }
  }
  if (!raw) raw = document.title || "";

  const filename = tidy(raw) || "download";
  log("saving as:", JSON.stringify(filename),
      `(${n} pages, ${Math.round(blob.size / 1024)} KB)`);

  /* ---------- download ---------- */
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `${filename}.pdf`;
  a.style.display = "none";
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(url), 60000);
})();


```
