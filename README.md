# How-to-download-protected-view-only-files-from-google-drive??

New Code:
```
(async function () {
  console.log("Loading jsPDF...");

  const script = document.createElement("script");
  script.src = "https://unpkg.com/jspdf@latest/dist/jspdf.umd.min.js";
  document.body.appendChild(script);

  await new Promise(res => script.onload = res);
  const { jsPDF } = window.jspdf;

  const imgs = [...document.images].filter(img =>
    img.src.startsWith("blob:https://drive.google.com/")
  );

  if (!imgs.length) {
    console.warn("No valid images found");
    return;
  }

  let pdf;

  for (let i = 0; i < imgs.length; i++) {
    const img = imgs[i];

    const w = img.naturalWidth;
    const h = img.naturalHeight;
    const orientation = w > h ? "l" : "p";

    const canvas = document.createElement("canvas");
    canvas.width = w;
    canvas.height = h;

    canvas.getContext("2d").drawImage(img, 0, 0, w, h);

    // JPEG is much faster & smaller than PNG
    const imgData = canvas.toDataURL("image/jpeg", 0.92);

    if (i === 0) {
      pdf = new jsPDF({
        orientation,
        unit: "px",
        format: [w, h],
      });
    } else {
      pdf.addPage([w, h], orientation);
    }

    pdf.addImage(imgData, "JPEG", 0, 0, w, h);

    console.log(`Progress: ${Math.round(((i + 1) / imgs.length) * 100)}%`);
  }

  // --- UPDATED TITLE EXTRACTION LOGIC ---
  let title = null;
  
  // 1. Try to grab it from the hidden JSON info div
  const driveInfoNode = document.getElementById("drive-active-item-info");
  if (driveInfoNode) {
    try {
      const driveInfo = JSON.parse(driveInfoNode.textContent);
      title = driveInfo.title;
    } catch (e) {
      console.warn("Failed to parse drive-active-item-info JSON", e);
    }
  }

  // 2. Fallbacks if the JSON div wasn't found or was missing the title
  if (!title) {
    title =
      document.querySelector('meta[itemprop="name"]')?.content ||
      document.title ||
      "download";
  }

  // Ensure it has exactly one .pdf extension at the end
  pdf.save(title.endsWith(".pdf") ? title : title + ".pdf");
})();


```
