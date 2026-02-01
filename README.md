# How-to-download-protected-view-only-files-from-google-drive??

1. Open or Preview Any view-only or protected files from google drive.

2. Open Developer Console.
    If you are previewing in Google Chrome or Firefox
    Press Shift + Ctrl + J ( on Windows / Linux) or Option + ⌘  + J (on Mac)
    If you are previewing in Microsoft Edge 
    Press Shift + Ctrl + I 
    If you are previewing in Apple Safari
    Press Option + ⌘ + C
    Then you will find yourself inside the developer tools.
    
3.  Navigate to the "Console" tab.

4.  Paste below code and press Enter:

        let jspdf = document.createElement( "script" );
        jspdf.onload = function () {
        let pdf = new jsPDF();
        let elements = document.getElementsByTagName( "img" );
        for ( let i in elements) {
        let img = elements[i];
        if (!/^blob:/.test(img.src)) {
        continue ;
        }
        let canvasElement = document.createElement( 'canvas' );
        let con = canvasElement.getContext( "2d" );
        canvasElement.width = img.width;
        canvasElement.height = img.height;
        con.drawImage(img, 0, 0,img.width, img.height);
        let imgData = canvasElement.toDataURL( "image/jpeg" , 1.0);
        pdf.addImage(imgData, 'JPEG' , 0, 0);
        pdf.addPage();
        }
        pdf.save( "download.pdf" );
        };
        jspdf.src = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf/1.3.2/jspdf.min.js' ;
        document.body.appendChild(jspdf);

5. Now, the pdf file start to download. This might take a few minutes depending on the file size.

6. Alternative one:
```
(function () {
  console.log("Loading script ...");

  let script = document.createElement("script");
  script.onload = function () {
    const { jsPDF } = window.jspdf;

    // Generate a PDF from images with "blob:" sources.
    let pdf = null;
    let imgElements = document.getElementsByTagName("img");
    let validImgs = [];

    console.log("Scanning content ...");
    for (let i = 0; i < imgElements.length; i++) {
      let img = imgElements[i];

      // specific check for Google Drive blob images
      let checkURLString = "blob:https://drive.google.com/";
      if (img.src.substring(0, checkURLString.length) !== checkURLString) {
        continue;
      }

      validImgs.push(img);
    }

    console.log(`${validImgs.length} content found!`);
    console.log("Generating PDF file ...");

    for (let i = 0; i < validImgs.length; i++) {
      let img = validImgs[i];
      
      // Convert image to DataURL via Canvas
      let canvasElement = document.createElement("canvas");
      let con = canvasElement.getContext("2d");
      canvasElement.width = img.naturalWidth;
      canvasElement.height = img.naturalHeight;
      con.drawImage(img, 0, 0, img.naturalWidth, img.naturalHeight);
      let imgData = canvasElement.toDataURL();

      // Determine orientation and dimensions for THIS specific image
      let orientation = img.naturalWidth > img.naturalHeight ? "l" : "p";
      let pageWidth = img.naturalWidth;
      let pageHeight = img.naturalHeight;

      if (i === 0) {
        // Initialize PDF with the dimensions of the FIRST image
        pdf = new jsPDF({
          orientation: orientation,
          unit: "px",
          format: [pageWidth, pageHeight],
        });
      } else {
        // For subsequent images, add a new page with THAT image's specific dimensions
        pdf.addPage([pageWidth, pageHeight], orientation);
      }

      // Add the image to the current page
      pdf.addImage(imgData, "PNG", 0, 0, pageWidth, pageHeight, "", "SLOW");

      const percentages = Math.floor(((i + 1) / validImgs.length) * 100);
      console.log(`Processing content ${percentages}%`);
    }

    // Check if title contains .pdf in end of the title
    // Use optional chaining to avoid errors if the meta tag isn't present.
    // Fall back to document.title when necessary. Note: if the PDF is inside a cross-origin iframe,
    // parent scripts cannot access the iframe document due to same-origin policy.
    let title = document.querySelector('meta[itemprop="name"]')?.content || document.title || 'download.pdf';
    if ((title.split(".").pop() || "").toLowerCase() !== "pdf") {
      title = title + ".pdf";
    }

    // Download the generated PDF.
    console.log("Downloading PDF file ...");
    pdf.save(title, { returnPromise: true }).then(() => {
      document.body.removeChild(script);
      console.log("PDF downloaded!");
    });
  };

  // Load the jsPDF library using the trusted URL.
  let scriptURL = "https://unpkg.com/jspdf@latest/dist/jspdf.umd.min.js";
  let trustedURL;
  if (window.trustedTypes && trustedTypes.createPolicy) {
    const policy = trustedTypes.createPolicy("myPolicy", {
      createScriptURL: (input) => {
        return input;
      },
    });
    trustedURL = policy.createScriptURL(scriptURL);
  } else {
    trustedURL = scriptURL;
  }

  script.src = trustedURL;
  document.body.appendChild(script);
})();
```
